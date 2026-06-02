# web-performance-analysis

A practical guide to measuring and analyzing web performance using the browser's Performance APIs, HTTP/2 optimizations, and Chrome DevTools.

- [web-performance-analysis](#web-performance-analysis)
  - [1. Introduction](#1-introduction)
    - [Initiator Types](#initiator-types)
    - [Cached Resources](#cached-resources)
    - [304 Not Modified](#304-not-modified)
    - [Blocking Time](#blocking-time)
  - [2. Performance API](#2-performance-api)
    - [2.1 Analysis](#21-analysis)
    - [2.2 Error Reporting](#22-error-reporting)
  - [3. HTTP/2 Performance Optimization](#3-http2-performance-optimization)
  - [4. Chrome Performance Page Analysis](#4-chrome-performance-page-analysis)
    - [4.1 Simulating a Mobile Device CPU](#41-simulating-a-mobile-device-cpu)
    - [4.2 Reading the Report](#42-reading-the-report)
    - [4.3 Interface Overview](#43-interface-overview)

## 1. Introduction

**Resource Timing** is a specification from the W3C Web Performance Working Group. Its goal is to provide accurate performance metrics for every resource downloaded during page load — images, CSS, JavaScript, and more.

Built on top of **Navigation Timing**, Resource Timing exposes additional measurement points covering the DNS, TCP, request, and response phases, plus the final `loaded` timestamp.

The API is inspired by the resource-loading waterfall: a timeline of every network fetch that makes problems easy to spot at a glance. Below is an example from Chrome DevTools:

![Waterfall](images/waterfall.jpg)

The timeline can be broken down into several phases:

![Timing](images/timing.jpg)

| Property | Description |
| --- | --- |
| `name` | Fully-resolved URL of the resource (relative URLs are expanded to the full protocol, domain, and path). |
| `entryType` | Always `"resource"` for Resource Timing entries. |
| `startTime` | When the resource started being fetched. |
| `duration` | Total time required to fetch the resource. |
| `initiatorType` | The `localName` of the element that initiated the fetch (see below). |
| `nextHopProtocol` | ALPN Protocol ID — e.g. `http/0.9`, `http/1.0`, `http/1.1`, `h2`, `hq`, `spdy/3` (Resource Timing Level 2). |
| `workerStart` | Time just before an active Service Worker received the fetch event, if one is installed. |
| `redirectStart` / `redirectEnd` | Time spent fetching any previous resources that redirected to this one. `0` means no redirects, or a redirect from a different origin. |
| `fetchStart` | When this specific resource started being fetched, excluding redirects. |
| `domainLookupStart` / `domainLookupEnd` | DNS lookup timestamps. |
| `connectStart` / `connectEnd` | TCP connection timestamps. |
| `secureConnectionStart` | Start of the SSL handshake, if any. `0` for plain HTTP or unsupported browsers. |
| `requestStart` | When the browser started requesting the resource from the remote server. |
| `responseStart` / `responseEnd` | Start of the response and when it finished downloading. |
| `transferSize` | Bytes transferred for the HTTP response headers and body (Level 2). |
| `decodedBodySize` | Body size after removing any applied content-codings (Level 2). |
| `encodedBodySize` | Body size before removing any applied content-codings (Level 2). |

### Initiator Types

The possible `initiatorType` values are:

`img`, `link`, `script`, `css` (`url()`, `@import`), `xmlhttprequest`, `iframe` (`subdocument` in some IE versions), `body`, `input`, `frame`, `object`, `image`, `beacon`, `fetch`, `video`, `audio`, `source`, `track`, `embed`, `eventsource`, `navigation`, `use`, and `other`.

The mapping between common HTML elements / JavaScript APIs and their `initiatorType`:

```js
<img src="...">: img
<img srcset="...">: img
<link rel="stylesheet" href="...">: link
<link rel="prefetch" href="...">: link
<link rel="preload" href="...">: link
<link rel="prerender" href="...">: link
<link rel="manifest" href="...">: link
<script src="...">: script
CSS @font-face { src: url(...) }: css
CSS background: url(...): css
CSS @import url(...): css
CSS cursor: url(...): css
CSS list-style-image: url(...): css
<body background=''>: body
<input src=''>: input
XMLHttpRequest.open(...): xmlhttprequest
<iframe src="...">: iframe
<frame src="...">: frame
<object>: object
<svg><image xlink:href="...">: image
<svg><use>: use
navigator.sendBeacon(...): beacon
fetch(...): fetch
<video src="...">: video
<video poster="...">: video
<video><source src="..."></video>: source
<audio src="...">: audio
<audio><source src="..."></audio>: source
<picture><source srcset="..."></picture>: source
<picture><img src="..."></picture>: img
favicon.ico: link
EventSource: eventsource
```

### Cached Resources

The following heuristic determines whether a request was served from cache. It isn't perfect, but it covers ~99% of cases:

```js
function isCacheHit() {
  // If we transferred bytes, it can't be a cache hit.
  // (Returns false for 304 Not Modified.)
  if (transferSize > 0) return false;

  // A non-zero body size means this is a Resource Timing 2 browser,
  // the request was same-origin or TAO-enabled, and transferSize was 0 —
  // so it came from the cache.
  if (decodedBodySize > 0) return true;

  // Fall back to duration checking (non-RT2 or cross-origin).
  return duration < 30;
}
```

### 304 Not Modified

```js
function is304() {
  if (encodedBodySize > 0 &&
      transferSize > 0 &&
      transferSize < encodedBodySize) {
    return true;
  }

  // Unknown
  return null;
}
```

### Blocking Time

```js
var blockingTime = 0;
if (res.connectEnd && res.connectEnd === res.fetchStart) {
  blockingTime = res.requestStart - res.connectEnd;
} else if (res.domainLookupStart) {
  blockingTime = res.domainLookupStart - res.fetchStart;
}
```

## 2. Performance API

**1. PerformanceObserver API**

Observes performance events using the observer pattern. Useful for collecting resource information, Time to Interactive (TTI), and long tasks.

Collecting resource information:

![per](images/per1.jpg)

Observing TTI:

![per](images/per2.jpg)

Observing long tasks:

![per](images/per3.jpg)

**2. Navigation Timing API**

Spec: https://www.w3.org/TR/navigation-timing-2/

```js
performance.getEntriesByType("navigation");
```

![per](images/per4.jpg)

![per](images/per5.jpg)

Two things to keep in mind:

- **Are the phases continuous?** No — there can be gaps between them.
- **Does every phase always occur?** No — some phases are conditional.

Common metrics derived from Navigation Timing:

| Metric | Formula |
| --- | --- |
| Redirect count | `performance.navigation.redirectCount` |
| Redirect time | `redirectEnd - redirectStart` |
| DNS lookup | `domainLookupEnd - domainLookupStart` |
| TCP connection | `connectEnd - connectStart` |
| SSL handshake | `connectEnd - secureConnectionStart` |
| Request time (TTFB) | `responseStart - requestStart` |
| Response / transfer | `responseEnd - responseStart` |
| DOM parsing | `domInteractive - responseEnd` |
| Resource loading | `loadEventStart - domContentLoadedEventEnd` |
| First byte | `responseStart - domainLookupStart` |
| Blank-screen time | `responseEnd - fetchStart` |
| Time to interactive | `domInteractive - fetchStart` |
| DOM Ready | `domContentLoadedEventEnd - fetchStart` |
| Full page load | `loadEventStart - fetchStart` |
| HTTP header size | `transferSize - encodedBodySize` |

**3. Resource Timing API**

Spec: https://w3c.github.io/resource-timing/

```js
performance.getEntriesByType("resource");
```

![per](images/per6.jpg)
![per](images/per7.jpg)

```js
// Load time for a class of resources — works for images, JS, CSS, XHR, etc.
resourceListEntries.forEach(resource => {
  if (resource.initiatorType == 'img') {
    console.info(`Time taken to load ${resource.name}: `, resource.responseEnd - resource.startTime);
  }
});
```

**4. Paint Timing API**

Spec: https://w3c.github.io/paint-timing/

Reports First Paint (FP) and First Contentful Paint (FCP).

```js
const paintEntries = performance.getEntriesByType("paint");

// Returns an array of two objects:
[
  {
    "name": "first-paint",
    "entryType": "paint",
    "startTime": 17718.514999956824,
    "duration": 0
  },
  {
    "name": "first-contentful-paint",
    "entryType": "paint",
    "startTime": 17718.519999994896,
    "duration": 0
  }
]

// Extract the metrics from the entries:
paintEntries.forEach((paintMetric) => {
  console.info(`${paintMetric.name}: ${paintMetric.startTime}`);
});
```

![per](images/per8.jpg)

**5. User Timing API**

Spec: https://www.w3.org/TR/user-timing-2/#introduction

Use `mark` and `measure` to instrument and time arbitrary segments of code, such as how long a specific function takes.

```js
performance.mark('starting_calculations');
const multiply = 82 * 21;
performance.mark('ending_calculations');
performance.measure("multiply_measure", "starting_calculations", "ending_calculations");

performance.mark('starting_awesome_script');
function awesomeScript() {
  console.log('doing awesome stuff');
}
performance.mark('ending_awesome_script');
performance.measure("awesome_script", "starting_awesome_script", "ending_awesome_script");

// Retrieve all measures with getEntriesByType:
const measures = performance.getEntriesByType('measure');
measures.forEach(measureItem => {
  console.log(`${measureItem.name}: ${measureItem.duration}`);
});
```

**6. High Resolution Time API**

Spec: https://w3c.github.io/hr-time/#dom-performance-timeorigin

Primarily provides the `now()` method and the `timeOrigin` property.

**7. Performance Timeline API**

Spec: https://www.w3.org/TR/performance-timeline-2/#introduction

### 2.1 Analysis

The Performance API lets us measure several categories: `mark`, `measure`, `navigation`, `resource`, `paint`, and `frame`.

```js
let p = window.performance.getEntries();
```

| Metric | Expression |
| --- | --- |
| Redirect count | `performance.navigation.redirectCount` |
| JS resource count | `p.filter(ele => ele.initiatorType === "script").length` |
| CSS resource count | `p.filter(ele => ele.initiatorType === "css").length` |
| AJAX request count | `p.filter(ele => ele.initiatorType === "xmlhttprequest").length` |
| Image resource count | `p.filter(ele => ele.initiatorType === "img").length` |
| Total resource count | `window.performance.getEntriesByType("resource").length` |

Non-overlapping phase durations:

| Phase | Formula |
| --- | --- |
| Redirect | `redirectEnd - redirectStart` |
| DNS lookup | `domainLookupEnd - domainLookupStart` |
| TCP connection | `connectEnd - connectStart` |
| SSL handshake | `connectEnd - secureConnectionStart` |
| Request (TTFB) | `responseStart - requestStart` |
| HTML download | `responseEnd - responseStart` |
| DOM parsing | `domInteractive - responseEnd` |
| Resource loading | `loadEventStart - domContentLoadedEventEnd` |

Composite metrics:

| Metric | Formula |
| --- | --- |
| Blank-screen time | `domLoading - fetchStart` |
| Approx. first-screen time | `loadEventEnd - fetchStart` or `domInteractive - fetchStart` |
| DOM Ready | `domContentLoadedEventEnd - fetchStart` |
| Full page load | `loadEventStart - fetchStart` |

Total JS load time:

```js
const p = window.performance.getEntries();
let jsR = p.filter(ele => ele.initiatorType === "script");
Math.max(...jsR.map((ele) => ele.responseEnd)) - Math.min(...jsR.map((ele) => ele.startTime));
```

Total CSS load time:

```js
const p = window.performance.getEntries();
let cssR = p.filter(ele => ele.initiatorType === "css");
Math.max(...cssR.map((ele) => ele.responseEnd)) - Math.min(...cssR.map((ele) => ele.startTime));
```

### 2.2 Error Reporting

**1) JavaScript errors** — listen for the `window.onerror` event.

**2) Unhandled promise rejections** — listen for the `unhandledrejection` event.

```js
window.addEventListener("unhandledrejection", function (event) {
  console.warn("WARNING: Unhandled promise rejection. Reason: " + event.reason);
});
```

**3) Resource load failures** — `window.addEventListener('error', ...)`.

**4) Network request failures** — wrap `window.XMLHttpRequest` and `window.fetch` to capture request errors.

**5) iframe errors** — `window.frames[0].onerror`.

**6) Console errors** — `window.console.error`.

## 3. HTTP/2 Performance Optimization

### 1. Binary Framing

HTTP/1.x is a text protocol made of three parts: a start line (request line or status line), headers, and a body. Parsing it requires text-based parsing, which is inherently fragile because text can be expressed in many forms — robust parsing has to handle a lot of edge cases. Binary is different: it only deals with combinations of `0` and `1`. For this reason, HTTP/2 adopted a binary format, which is both simpler and more robust to parse.

HTTP/2 defines individual **frames** in a binary format. Compared with HTTP/1.x:

![http2](images/http1.jpg)

The HTTP/2 frame format is closer to how the TCP layer works — highly efficient and compact:

- **Length** — defines the frame's size from start to end.
- **Type** — the frame type (10 types in total).
- **Flags** — bit flags for important parameters.
- **Stream ID** — used for flow control.
- **Payload** — the request body.

So how does HTTP/2 *"break through HTTP/1.1's performance limits, improve transfer performance, and achieve low latency with high throughput"* without changing HTTP's semantics, methods, status codes, URIs, or header fields?

One key is a new **binary framing layer** inserted between the application layer (HTTP/2) and the transport layer (TCP or UDP).

![http2](images/http2.jpg)

Everything is encoded in binary: HTTP/1.x header information is wrapped into a `HEADERS` frame, and the request body into a `DATA` frame. All HTTP/2 communication happens over a single connection that can carry any number of bidirectional streams.

Historically, the key to HTTP performance optimization was not high bandwidth but **low latency**. A TCP connection "tunes" itself over time, initially limiting its maximum speed and then increasing the transfer rate as data is successfully delivered — a process known as **TCP slow start**. Because of this, the bursty and short-lived nature of HTTP connections made them very inefficient.

By sharing a single connection across all data streams, HTTP/2 uses TCP connections far more effectively, letting high bandwidth genuinely improve HTTP performance.

**Summary:** Multiplexing many resources over one connection reduces server connection pressure, uses less memory, and increases throughput. Fewer TCP connections ease network congestion, while reduced slow-start time speeds up recovery from congestion and packet loss.

![http2](images/http3.jpg)

### 2. Multiplexing (Connection Sharing)

Multiplexing allows multiple request/response messages to be sent concurrently over a single HTTP/2 connection.

In HTTP/1.1, browsers limit the number of simultaneous requests to a single domain; requests beyond that limit are blocked:

> Clients that use persistent connections SHOULD limit the number of simultaneous connections that they maintain to a given server. A single-user client SHOULD NOT maintain more than 2 connections with any server or proxy. A proxy SHOULD use up to 2*N connections to another server or proxy, where N is the number of simultaneously active users. These guidelines are intended to improve HTTP response times and avoid congestion.
>
> — RFC 2616, §8.1.4 Practical Considerations

For example, the TCP three-way handshake adds about 1.5 RTTs (round-trip times) of latency. To avoid paying that on every request, the application layer uses various HTTP keep-alive strategies. And because of TCP slow start, reusing a connection always outperforms opening a new one.

Each request maps to a **stream** with its own ID, so a single connection can carry many streams. Frames from different streams can be interleaved arbitrarily, and the receiver reassembles them by stream ID. This is what enables HTTP/2 multiplexing.

![http2](images/http4.jpg)

HTTP/2 therefore achieves parallel streams without relying on multiple TCP connections. It shrinks the basic unit of HTTP communication down to frames, which correspond to messages in a logical stream, and exchanges those messages bidirectionally over the same TCP connection.

As mentioned, connection sharing also needs **priority** and **dependency** mechanisms to prevent critical requests from being blocked. Every HTTP/2 stream can be assigned a priority and a dependency, both adjustable dynamically. Higher-priority streams are processed and returned first; streams can also depend on sub-streams. Dynamic adjustment is useful in practice — imagine a user quickly scrolling to the bottom of a product list: if later requests aren't given higher priority, the images the user is currently viewing would download last, hurting the experience.

### 3. Header Compression

HTTP/1.x headers easily bloat due to cookies and user-agent strings, and they're re-sent with every request.

HTTP/1.1 does not support header compression, which is why SPDY and HTTP/2 emerged.

A quick note on TCP slow start: after the three-way handshake, the number of un-ACKed segments that can be sent first is determined by the **initial TCP window** size. This varies by platform but is typically 2 segments or about 4 KB (a segment is roughly 1500 bytes). If your payload exceeds this value, later packets must wait for earlier ones to be ACKed — increasing latency. A larger initial window isn't always better either: too large causes network congestion and higher packet loss. HTTP headers can now exceed the initial window size, which makes header compression all the more important.

**Choosing a compression algorithm:** SPDY/2 used gzip, but two attacks — BREACH and CRIME — meant even SPDY over SSL could have its content decrypted. After weighing the options, HTTP/2 adopted an algorithm called **HPACK**. (These vulnerabilities mainly affect the browser side, since they rely on injecting content via JavaScript and observing payload changes.)

So SPDY uses the general-purpose DEFLATE algorithm, while HTTP/2 uses HPACK, designed specifically for header compression. HTTP/2 uses an encoder to reduce header size, and both endpoints maintain a cached table of header fields — avoiding redundant transmission and shrinking what must be sent. Efficient compression greatly reduces header size and packet count, lowering latency.

![http2](images/http5.jpg)

### 4. Server Push

Server Push is a mechanism for sending data before the client requests it. In HTTP/2, the server can send multiple responses to a single client request. This makes the HTTP/1.x optimization of inlining resources unnecessary: if a request comes from your homepage, the server can proactively respond with the homepage content, logo, and stylesheet because it knows the client will need them. Unlike inlining everything into one HTML document, pushed resources can be **cached** — and, under the same-origin policy, shared across different pages.

![http2](images/http6.jpg)

### 5. Better Stream Reset

Many client apps support canceling an image download. In HTTP/1.x, this is done by setting the TCP segment's reset flag to close the connection — but that fully tears down the connection, so the next request must re-establish it. HTTP/2 introduces the `RST_STREAM` frame, which cancels a single request's stream **without** dropping the connection, giving better performance.

### 6. Flow Control

TCP performs flow control via a sliding-window algorithm: the sender has a sending window and the receiver a receive window. HTTP/2's flow control is similar to a receive window — the data receiver tells the sender its flow-window size to indicate how much more data it can accept. Only `DATA` frames are subject to flow control. If a sender keeps sending frames while the receiver's flow window is zero, the receiver returns a `BLOCKED` frame, which usually signals a misconfigured HTTP/2 deployment.

### 7. Stronger SSL

HTTP/2 uses the TLS ALPN extension for protocol upgrade. It also strengthens TLS security by blacklisting hundreds of insecure cipher suites. If, during the SSL negotiation, the client and server share no common cipher suite, negotiation fails and the request fails outright. Pay special attention to this when deploying HTTP/2 on the server.

## 4. Chrome Performance Page Analysis

### 4.1 Simulating a Mobile Device CPU

Mobile CPUs are usually much weaker than those in desktops and laptops. When profiling a page, use **CPU Throttling** to simulate a mobile device's CPU.

1. In DevTools, open the **Performance** tab.
2. Make sure the **Screenshots** checkbox is checked.
3. Click the **Capture Settings** (⚙️) button to reveal options for simulating various conditions.
4. For CPU, choose **2x slowdown**; DevTools will now simulate a CPU running at half speed.
5. Click **Record** to start capturing performance metrics.
6. Perform your interactions, click **Stop**, and DevTools will process the data and show the report.

### 4.2 Reading the Report

**FPS** (frames per second) is a primary metric for analyzing animation. Aim for a refresh rate of **≥ 60 fps** to avoid jank — sustaining 60 fps generally means a good user experience.

**Why 60 fps?** Our goal is a refresh rate above 60 fps because most displays today refresh at 60 Hz. If page animations hit 60 fps, they sync with the display for the best visual result. That means 60 re-renders per second, with each render taking no more than **16.66 ms**.

### 4.3 Interface Overview

![performance](images/performanc1.jpg)

From top to bottom there are four regions:

1. **Controls** — record, reload-and-profile, clear results, and other actions.
2. **Overview** — a high-level chart of changes over the timeline, including FPS, CPU, and NET.
3. **Flame chart** — analyze the selected region from different angles: Network, Frames, Interactions, Main, etc.
4. **Summary** — millisecond-precise analysis, organized by call hierarchy and event category.

![performance](images/performanc2.jpg)

**Overview**

The Overview pane contains three charts:

1. **FPS** — frames per second. Taller green bars mean higher FPS. A red block above the FPS chart marks a long frame that likely caused jank.
2. **CPU** — CPU usage. This area chart shows which event types are consuming CPU.
3. **NET** — each colored bar represents one resource. The longer the bar, the longer the resource took to retrieve. The lighter portion of each bar is wait time (from requesting the resource to the first byte being downloaded).

You can zoom into a portion of the recording to simplify analysis. After zooming, the flame chart automatically scales to match the same range.

Once a range is selected, use **W**, **A**, **S**, and **D** to adjust it: **W** and **S** zoom in and out; **A** and **D** move left and right.

**Flame Chart**

You'll see one to three vertical dashed lines on the flame chart. The blue line marks the `DOMContentLoaded` event, the green line marks First Paint, and the red line marks the `load` event.

Selecting an event in the flame chart shows additional details in the Details pane.

**Summary**

| Color | Category | Meaning |
| --- | --- | --- |
| Blue | Loading | Network communication and HTML parsing |
| Yellow | Scripting | JavaScript execution |
| Purple | Rendering | Style calculation and layout (reflow) |
| Green | Painting | Repaint |
| Gray | Other | Time spent on other events |
| White | Idle | Idle time |

**Loading events**

| Event | Description |
| --- | --- |
| Parse HTML | Triggered when the browser parses HTML. |
| Finish Loading | Triggered when a network request completes. |
| Receive Data | Triggered when response data arrives; may fire multiple times for large responses split across packets. |
| Receive Response | Triggered when the response headers arrive. |
| Send Request | Triggered when a network request is sent. |

**Scripting events**

| Event | Description |
| --- | --- |
| Animation Frame Fired | Triggered when a defined animation frame fires and its callback begins. |
| Cancel Animation Frame | Triggered when an animation frame is canceled. |
| GC Event | Triggered during garbage collection. |
| DOMContentLoaded | Triggered when the page's DOM content has finished loading and parsing. |
| Evaluate Script | Triggered when a script is evaluated. |
| Event | A JavaScript event. |
| Function Call | Triggered only when a top-level JavaScript function is called. |
| Install Timer | Triggered when a timer is created via `setTimeout()` or `setInterval()`. |
| Request Animation Frame | A `requestAnimationFrame()` call scheduled a new frame. |
| Remove Timer | Triggered when a timer is cleared. |
| Time | Triggered when `console.time()` is called. |
| Time End | Triggered when `console.timeEnd()` is called. |
| Timer Fired | Triggered when a timer's callback is activated. |
| XHR Ready State Change | Triggered when an async request's ready state changes. |
| XHR Load | Triggered when an async request finishes loading. |

**Rendering events**

| Event | Description |
| --- | --- |
| Invalidate layout | Triggered when a DOM change invalidates the page layout. |
| Layout | Triggered when the page layout is computed. |
| Recalculate style | Triggered when Chrome recalculates element styles. |
| Scroll | Triggered when a nested viewport scrolls. |

**Painting events**

| Event | Description |
| --- | --- |
| Composite Layers | Triggered when Chrome's rendering engine finishes compositing image layers. |
| Image Decode | Triggered when an image resource finishes decoding. |
| Image Resize | Triggered when an image is resized. |
| Paint | Triggered when composited layers are painted to their display region. |

## References

- https://nicj.net/navigationtiming-in-practice/
- https://nicj.net/resourcetiming-in-practice/
- http://www.alloyteam.com/2020/01/14184/#prettyPhoto
- https://www.w3.org/TR/resource-timing-1/
- https://www.w3.org/TR/resource-timing-2/
