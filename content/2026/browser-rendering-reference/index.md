---
layout: post
title: "Browser Rendering Reference"
description: "How Browser renders html, css and js content"
date: 2026-10-13
type: guide
tags: ["html", "web", "css", "js"]
featured: false
ai: generated
---

> **Comprehensive Engineering Guide for Modern Frontend Developers**  
> *Topics covered: Navigation, DOM, CSSOM, Render Tree, Layout (Reflow), Paint (Repaint), Compositing, GPU Acceleration, Input Handling, and Performance Diagnostics.*

---

## Table of Contents
1. [Architectural Overview & Multi-Threaded Engine Model](#1-architectural-overview--multi-threaded-engine-model)
2. [Phase 1: Navigation & Network Transmission](#2-phase-1-navigation--network-transmission)
3. [Phase 2: Parsing & Object Model Construction](#3-phase-2-parsing--object-model-construction)
   - [2.1 Document Object Model (DOM)](#21-document-object-model-dom)
   - [2.2 Preload Scanner & Speculative Parsing](#22-preload-scanner--speculative-parsing)
   - [2.3 CSS Object Model (CSSOM)](#23-css-object-model-cssom)
   - [2.4 Script Parsing & Parser-Blocking Mechanics](#24-script-parsing--parser-blocking-mechanics)
   - [2.5 Accessibility Object Model (AOM)](#25-accessibility-object-model-aom)
4. [Phase 3: Render Tree (Layout Tree) Construction](#4-phase-3-render-tree-layout-tree-construction)
5. [Phase 4: Layout (Reflow)](#5-phase-4-layout-reflow)
6. [Phase 5: Paint (Repaint) & Display Lists](#6-phase-5-paint-repaint--display-lists)
7. [Phase 6: Compositing & Off-Main-Thread Rasterization](#7-phase-6-compositing--off-main-thread-rasterization)
8. [Phase 7: User Input & Interaction Handling on the Compositor](#8-phase-7-user-input--interaction-handling-on-the-compositor)
9. [Performance Engineering: Reflow vs. Repaint vs. Composite](#9-performance-engineering-reflow-vs-repaint-vs-composite)
10. [Diagnostic Code Samples & Production Patterns](#10-diagnostic-code-samples--production-patterns)
11. [Annotated Bibliography: The 7 Core External References](#11-annotated-bibliography-the-7-core-external-references)

---

## 1. Architectural Overview & Multi-Threaded Engine Model

Modern web browsers (such as Chromium/Blink, WebKit, and Gecko) do not run page rendering as a monolithic single-threaded task. Instead, modern browsers employ a multi-process, multi-threaded architecture:

```
┌────────────────────────────────────────────────────────────────────────┐
│ Browser Process (Coordinates UI, Network, Storage, Device APIs)       │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ IPC
┌───────────────────────────────────▼────────────────────────────────────┐
│ Renderer Process (Sandboxed, per site-instance / tab)                  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Main Thread                                                      │  │
│  │ - HTML/CSS Parsing ──► DOM & CSSOM                               │  │
│  │ - JavaScript Execution (V8 / JavaScriptCore)                     │  │
│  │ - Style Recalculation & Render Tree Construction                 │  │
│  │ - Layout / Reflow (Geometry calculation)                         │  │
│  │ - Paint (Generates Paint Records / Display Lists)                │  │
│  └──────────────────────────────┬───────────────────────────────────┘  │
│                                 │ Commit Layer Tree & Paint Records    │
│  ┌──────────────────────────────▼───────────────────────────────────┐  │
│  │ Compositor Thread                                                │  │
│  │ - Divides Layers into Tiles                                      │  │
│  │ - Schedules Raster Tasks                                         │  │
│  │ - Generates Draw Quads & Assembles Compositor Frames             │  │
│  │ - Smooth Scrolling & Direct Composited Animations                │  │
│  └──────────────┬───────────────────────────────────▲───────────────┘  │
│                 │ Raster Tasks                      │ Completed Tiles  │
│  ┌──────────────▼───────────────────────────────────┴───────────────┐  │
│  │ Raster Worker Threads (Convert Draw Commands to Bitmaps / Skia)  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ IPC (Compositor Frames)
┌───────────────────────────────────▼────────────────────────────────────┐
│ GPU Process (Renders Draw Quads to Hardware Screen / Framebuffer)      │
└────────────────────────────────────────────────────────────────────────┘
```

### Key Threads in the Renderer Process
- **Main Thread:** Handles DOM parsing, CSS parsing, JavaScript execution, layout calculation, and paint record creation. Because the main thread handles all these tasks, long-running JavaScript halts layout and paint, resulting in dropped frames (jank).
- **Compositor Thread:** Runs independently of the main thread. Responsible for layer tiling, handling user scroll/pinch gestures, and submitting Compositor Frames directly to the GPU process.
- **Raster Threads:** Background worker threads spawned to turn vector display lists of individual layer tiles into raster bitmaps stored in GPU memory.

---

## 2. Phase 1: Navigation & Network Transmission

Before any byte can be converted into visual pixels, navigation establishes the network transport:
1. **DNS Lookup:** Resolves the hostname into an IP address. Lookups occur once per unique hostname referenced in the document (fonts, CDNs, scripts, analytics).
2. **TCP Three-Way Handshake:** Performs `SYN ➔ SYN-ACK ➔ ACK` negotiation to set up socket communication.
3. **TLS Negotiation (HTTPS):** Negotiates cipher suites, verifies certificates, and generates encryption keys (requiring up to five network round trips).
4. **HTTP Request & Time to First Byte (TTFB):** The browser dispatches an `HTTP GET` request for the HTML document.
5. **TCP Slow Start & Initial Congestion Window (CWND):** The server sends data in segment bursts. The initial CWND typically delivers up to 10 TCP segments (~14 KB). Optimizing critical inline CSS and skeleton HTML within the first 14 KB allows the browser to begin constructing the UI immediately on the first packet arrival.

---

## 3. Phase 2: Parsing & Object Model Construction

### 2.1 Document Object Model (DOM)
HTML parsing is **incremental and streaming**. The browser consumes raw byte chunks as they arrive over the wire without waiting for the complete file to finish downloading:
- **Bytes ➔ Characters:** Bytes are decoded into characters based on the specified character encoding (e.g., `UTF-8`).
- **Characters ➔ Tokens:** Lexical analysis splits the character stream into discrete tokens: `StartTag`, `EndTag`, `Attribute`, and `CharacterData`.
- **Tokens ➔ Nodes:** Tokens are converted into W3C DOM `Node` objects containing tag attributes and relationships.
- **Nodes ➔ DOM Tree:** The parser nests nodes hierarchically based on token order, establishing parent-child-sibling relationships rooted at `Document` and `<html>`.

### 2.2 Preload Scanner & Speculative Parsing
While the main thread parser is blocked by scripts or busy assembling nodes, a secondary lightweight thread—the **Preload Scanner**—scans ahead in the raw token stream. It detects linked subresources (`<link rel="stylesheet">`, `<script src="...">`, `<img>`, `@font-face`) and dispatches network fetch requests in parallel, preventing network idle waterfalls.

### 2.3 CSS Object Model (CSSOM)
The CSS Object Model represents all CSS styling rules attached to the document:
- **Non-Streaming / Non-Incremental:** Unlike HTML, **CSS is strictly render-blocking and cannot be parsed incrementally**. Because cascading rules, specificity hierarchies, and later declarations can override earlier rules, the browser cannot compute a node's styles until the stylesheet is parsed in its entirety.
- **Right-to-Left Selector Matching:** Browsers evaluate CSS selectors from right to left (starting with the *key selector*). For example, with `div.sidebar ul > li a`, the engine first locates all `<a>` tags, then checks their parents for `<li>`, `<ul>`, and `div.sidebar`.
- **Recalculate Style Cost:** CSS parser execution is generally fast, but complex selector trees with deep ancestor traversals increase the work required during style recalculation.

### 2.4 Script Parsing & Parser-Blocking Mechanics
- **Default `<script>`:** The HTML parser immediately pauses tokenization when encountering an inline or external synchronous `<script>`. It must fetch, compile, and execute the JavaScript before proceeding because JavaScript can modify the DOM via `document.write()`.
- **CSS Blocks Script Execution:** If an external stylesheet is downloading when a `<script>` tag is reached, script execution is delayed until the CSSOM finishes building, as JavaScript code may query computed layout styles via `window.getComputedStyle()`.
- **Asynchronous Execution Modes:**
  - `<script async>`: Downloads concurrently without blocking HTML parsing. Executes as soon as downloaded (execution order is non-deterministic; may interrupt HTML parsing).
  - `<script defer>`: Downloads concurrently. Defers execution until HTML parsing has completely finished, executing in strict document order prior to `DOMContentLoaded`.

### 2.5 Accessibility Object Model (AOM)
Simultaneously with DOM and CSSOM processing, the browser builds an **accessibility tree** (AOM). The AOM is a semantic representation used by assistive technologies (screen readers, braille displays). Dynamic style changes like `display: none` remove nodes from both the Render Tree and the Accessibility Tree, whereas `visibility: hidden` and `opacity: 0` retain them in the layout and accessibility tree.

---

## 4. Phase 3: Render Tree (Layout Tree) Construction

The **Render Tree** (also referred to as the **Layout Tree**) merges the content structure of the DOM with the computed rules of the CSSOM:

```
DOM Tree              CSSOM Tree
┌──────────────┐     ┌──────────────┐
│ <html>       │     │ html { ... } │
│  <body>      │  +  │ body { ... } │ ──► Render Tree (Layout Tree)
│   <h1>Title  │     │ h1 { color } │     [Visible elements with computed styles]
│   <p>Text    │     │ p { display }│
└──────────────┘     └──────────────┘
```

### Filtering and Tree Pruning Rules:
1. **Omission of Non-Visual Nodes:** Head metadata (`<head>`, `<meta>`, `<link>`, `<script>`, `<style>`) is omitted.
2. **`display: none` Elimination:** Elements with `display: none` (and all their descendants) are completely excluded from the Render Tree. They generate no layout boxes and consume no paint operations.
3. **`visibility: hidden` Preservation:** Elements with `visibility: hidden` **are included** in the Render Tree. They generate invisible layout boxes that occupy geometric space in the document flow.
4. **Generated Pseudo-Elements:** CSS generated content such as `::before` and `::after` exists only in the Render Tree; they do not appear in the DOM.

---

## 5. Phase 4: Layout (Reflow)

With the Render Tree constructed, the browser calculates the precise geometry of every element:
- **Geometry Resolution:** Layout calculates the exact hardware pixel coordinates $(x, y)$ and physical dimensions $(\text{width}, \text{height})$ of every box in the viewport.
- **Relative to Absolute Unit Conversion:** Translates relative CSS units (`em`, `rem`, `%`, `vw`, `vh`) into physical device pixels based on the layout viewport.
- **Viewport Constraints:** Influenced directly by the viewport meta tag (`<meta name="viewport" content="width=device-width, initial-scale=1">`). Without this tag, mobile browsers default to a desktop canvas width (typically 960px) and scale down.
- **Global vs. Incremental Layout:**
  - *Global Layout:* Affects the entire document tree; triggered by viewport resize, font changes, or root class alterations.
  - *Incremental Layout:* Modifies localized subtrees; triggered by bounded DOM mutations.

---

## 6. Phase 5: Paint (Repaint) & Display Lists

Having the dimensions and positions of elements is insufficient to render them to the screen; the browser must also determine the **drawing order**:
- **Paint Records (Display Lists):** The main thread walks the Render Tree to generate a stream of draw calls (e.g., "draw background rect at $(0, 0)$", "draw text glyphs at $(10, 20)$", "apply border").
- **Stacking Context & Painting Order:** Drawing adheres strictly to the stacking context:
  1. Background and borders of the root element
  2. Descendant blocks in normal document flow
  3. Non-positioned floats
  4. In-flow inline elements (text, inline images)
  5. Positioned descendants with `z-index: 0` or `auto`
  6. Positioned descendants with positive `z-index` in ascending order

---

## 7. Phase 6: Compositing & Off-Main-Thread Rasterization

To deliver smooth 60 FPS / 120 FPS frame rates, modern browser engines decouple drawing and display via **Compositing**:

1. **Update Layer Tree:** The main thread analyzes paint records and identifies elements that require isolated graphics layers.
   - *Promotion Criteria:* Hardware 3D transforms (`transform: translateZ(0)`), `<video>`, `<canvas>`, CSS animations on `transform` or `opacity`, or elements declared with `will-change: transform`.
2. **Commit to Compositor Thread:** The layer tree and paint records are committed across threads to the **Compositor Thread**.
3. **Tiling:** Large layers (which can span thousands of pixels vertically) are segmented into uniform rectangular **tiles** (typically $256 \times 256$ or $512 \times 512$ pixels).
4. **Rasterization via Worker Threads:** Raster worker threads convert the draw commands of each tile into bitmap textures stored in GPU VRAM (utilizing graphics libraries such as Skia). Tiles inside or adjacent to the viewport are rasterized first.
5. **Draw Quads & Compositor Frame:** The compositor compiles tile location metadata into **Draw Quads**. It aggregates these into a **Compositor Frame** representing the assembled screen.
6. **IPC to GPU Process:** The Compositor Frame is dispatched via Inter-Process Communication (IPC) to the browser's **GPU Process**, which sends the pixels to the physical display screen.

---

## 8. Phase 7: User Input & Interaction Handling on the Compositor

When users interact with a webpage via touch, trackpad, or mouse, inputs follow a specialized pipeline:
- **Input Routing:** The OS delivers input events to the Browser Process, which forwards coordinates to the Compositor Thread.
- **Compositor Hit-Testing:** The compositor inspects paint records and layer geometry to determine which element lies beneath the gesture coordinates.
- **Non-Fast Scrollable Regions:** If a scroll or touch gesture occurs on an element that has non-passive JavaScript event listeners attached (`touchstart`, `wheel`), the compositor cannot scroll the page immediately. It must pause and send an IPC message to the main thread to see if JavaScript calls `event.preventDefault()`. This causes scroll latency.
- **Passive Event Listeners:** By specifying `{ passive: true }`, developers inform the compositor that JavaScript will never call `event.preventDefault()`, allowing the compositor thread to scroll the page smoothly without waiting for the main thread.
- **Event Coalescing:** High-frequency input devices (e.g., 120Hz touchscreens or 1000Hz gaming mice) generate events faster than the display's 60Hz/120Hz refresh rate. Chrome coalesces continuous events (`pointermove`, `touchmove`, `wheel`) and dispatches them once per frame immediately before `requestAnimationFrame`. Developers can access intra-frame coordinates using `event.getCoalescedEvents()`.

---

## 9. Performance Engineering: Reflow vs. Repaint vs. Composite

Every visual change on a web page triggers a subset of the rendering pipeline. Understanding which CSS properties trigger which stages is essential for performance optimization:

| Pipeline Stage | Pipeline Operations Executed | Typical CSS Properties | Performance Impact |
| :--- | :--- | :--- | :--- |
| **Reflow (Layout)** | Recalculate Styles ➔ Layout ➔ Paint ➔ Composite | `width`, `height`, `margin`, `padding`, `top`, `left`, `fontSize`, `display`, `grid`, `flex` | **Highest Cost** (Forces full geometry calculation on Main Thread) |
| **Repaint** | Recalculate Styles ➔ Paint ➔ Composite (Skips Layout) | `color`, `background-color`, `visibility`, `box-shadow`, `border-color`, `outline` | **Medium Cost** (Re-rasterizes affected pixels; skips geometry) |
| **Composite Only** | Composite only (Skips Layout and Paint) | **`transform`**, **`opacity`**, `filter` (with GPU layer backing) | **Lowest Cost** (Bypasses Main Thread entirely; handled on GPU) |

---

## 10. Diagnostic Code Samples & Production Patterns

### 10.1 Avoiding Layout Thrashing (Forced Synchronous Layout)
Layout thrashing occurs when JavaScript interleaves DOM writes and DOM reads in a loop, repeatedly invalidating and recalculating geometry:

```javascript
// ❌ ANTI-PATTERN: Forces N synchronous layout calculations
function updateElementHeights(elements) {
  for (let i = 0; i < elements.length; i++) {
    // Reading offsetHeight forces immediate layout calculation
    const height = elements[i].offsetHeight;
    // Writing style invalidates layout for the next iteration
    elements[i].style.height = `${height + 20}px`;
  }
}

// ✅ OPTIMIZED PATTERN: Batched Reads followed by Batched Writes
function updateElementHeightsOptimized(elements) {
  // Phase 1: Pure Geometric Reads (Single layout pass)
  const newHeights = elements.map(el => el.offsetHeight + 20);

  // Phase 2: Pure Batched Writes (Scheduled before next frame paint)
  requestAnimationFrame(() => {
    elements.forEach((el, index) => {
      el.style.height = `${newHeights[index]}px`;
    });
  });
}
```

### 10.2 Non-Blocking Input Listeners & CSS Touch Action
```javascript
// ❌ ANTI-PATTERN: Marks the entire page as a Non-Fast Scrollable Region
document.addEventListener('touchstart', (e) => {
  trackUserTouch(e);
});

// ✅ OPTIMIZED: Enables smooth compositor scrolling via passive flag
document.addEventListener('touchstart', (e) => {
  trackUserTouch(e);
}, { passive: true });
```

```css
/* ✅ OPTIMIZED: Pure CSS touch handling without JavaScript intervention */
.horizontal-carousel {
  touch-action: pan-y; /* Lets compositor handle vertical page scrolling directly */
}
```

---

## 11. Annotated Bibliography: The 7 Core External References

Below are the 7 authoritative external documentation sources and engineering analyses that provide the foundation for browser rendering pipeline specifications:

### 1. MDN Web Docs: "Populating the page: how browsers work"
- **Publisher:** Mozilla Developer Network (MDN)
- **Official URL:** `https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/How_browsers_work`
- **Core Scope:** Comprehensive overview of the complete browser lifecycle from navigation, DNS lookups, TCP handshakes, and TLS negotiations through to HTML parsing, CSSOM generation, render tree creation, layout reflow, painting, and Time to Interactive (TTI).

### 2. MDN Web Docs: "Critical rendering path"
- **Publisher:** Mozilla Developer Network (MDN)
- **Official URL:** `https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Critical_rendering_path`
- **Core Scope:** Formal breakdown of the Critical Rendering Path (CRP). Explains the streaming nature of DOM construction, why CSS is render-blocking, the parser-blocking behavior of synchronous scripts, and optimization strategies like resource preloading and deferred execution.

### 3. web.dev: "Render-tree Construction, Layout, and Paint"
- **Publisher:** Google Chrome Developer Relations (web.dev)
- **Author:** Ilya Grigorik
- **Official URL:** `https://web.dev/articles/critical-rendering-path/render-tree-construction`
- **Core Scope:** Detailed examination of how DOM and CSSOM trees merge into the Render Tree. Clarifies the difference between `display: none` (omitted from render tree) and `visibility: hidden` (retained in layout), viewport geometry calculation, and display list painting.

### 4. Chrome for Developers: "Inside look at modern web browser (part 3): Renderer Process"
- **Publisher:** Google Chrome Developer Blog
- **Author:** Mariko Kosaka
- **Official URL:** `https://developer.chrome.com/blog/inside-browser-part3`
- **Core Scope:** In-depth architectural explanation of Chromium's renderer process. Covers the main thread, the preload scanner, layout trees, paint records, the compositor thread, layer splitting, tile rasterization via worker threads, draw quads, and IPC to the GPU process.

### 5. Chrome for Developers: "Inside look at modern web browser (part 4): Input & Compositor"
- **Publisher:** Google Chrome Developer Blog
- **Author:** Mariko Kosaka
- **Official URL:** `https://developer.chrome.com/blog/inside-browser-part4`
- **Core Scope:** Focuses on the compositor thread's role during user interaction. Details non-fast scrollable regions, event delegation bottlenecks, the performance mechanics of `{ passive: true }`, compositor-driven hit-testing, and event coalescing.

### 6. GreatFrontEnd: "50 Must-know Front End Interview Questions: Rendering & Layout Thrashing"
- **Publisher:** GreatFrontEnd
- **Official URL:** `https://www.greatfrontend.com/blog/50-must-know-html-css-and-javascript-interview-questions-by-ex-interviewers`
- **Core Scope:** Senior frontend engineering interview guide focusing on layout thrashing, forced synchronous layouts, reflow triggers, stacking contexts, CSS selector matching efficiency, and performance profiling.

### 7. Technical Synthesis: "Research report: Browser Rendering Pipeline"
- **Publisher:** Frontend Architecture Research Series
- **Document Identifier:** ContribService Stored Reference (`Research_report_Browser_Rendering_Pipeline.md`)
- **Core Scope:** Curated synthesis synthesizing the six web specifications into an integrated technical reference covering thread separation, render-blocking resources, hardware acceleration trade-offs, and production profiling strategies.
