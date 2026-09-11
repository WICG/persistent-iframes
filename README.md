# Persistent Widgets

## Problem & Motivation

Navigations of top-level documents in Multi-Page Applications (MPAs) result in the destruction of the top-level document and all child browsing contexts (such as `<iframe>` elements) within that document.

In cases of same-origin navigations, embedded components frequently need to be recreated on the destination document. Many modern web applications rely on embedded components maintaining complex and stateful client-side workflows:
- **In-Page AI Chat & Support Assistants**: Users navigate across the site (looking up orders, reviewing specifications, comparing plans) while interacting with the assistant.
- **Continuous Media Players & Live Streams**: Users browse articles, comments, or catalog listings while watching a video in a persistent mini-player.
- **Collaborative Editing & Persistent Connections**: Ongoing WebSockets, WebRTC streams, or long-running background tasks.

Recreating these embedded frames on every page navigation causes severe user experience degradation:
- **Loss of in-memory JavaScript & DOM state.**
- **Loss of expensive, non-serializable state** (e.g., GPU Key-Value caches for on-device WebNN / WebGPU AI inference).
- **Disrupted network connections** (WebSockets, WebRTC).
- **Audio and video playback interruptions and re-buffering delays.**
- **Substantial user-visible latency, blank states, and layout flicker.**

This significantly disadvantages MPA architectures (which rely on standard same-origin navigations) compared to Single-Page Application (SPA) architectures, which can soft-navigate parts of the document while leaving child frames intact. Web developers are often forced into costly SPA rewrites solely to preserve UI continuity for embedded components.

This proposal introduces **Persistent Widgets** (`<persistentwidget>`), a web platform mechanism enabling embedded browsing contexts to survive same-origin top-level document navigations without reloading or losing state.

---

## Limitations of Existing Workarounds

Current workarounds to achieve UI continuity across navigations fall short:

### 1. State Offloading and Rehydration
Developers attempt to serialize component state before navigation (e.g., in `sessionStorage` or `IndexedDB`) and rebuild the UI on the new page:
- **Media & Connection Interruptions**: Re-establishing WebSockets, WebRTC channels, or media buffers takes noticeable time, causing visual drops in playback and delayed message receipt.
- **Unserializable In-Memory State**: Large in-memory states—such as GPU K-V caches for on-device AI inference—cannot be serialized efficiently, forcing compute tasks to restart from scratch.
- **SharedWorker Limitations**: While a `SharedWorker` (with `extendedlifetime`) can persist JavaScript state across navigations, workers cannot currently access permission-gated APIs.

### 2. Persistent Auxiliary Windows (Popups / Document Picture-in-Picture)
Developers can open an auxiliary window (`window.open` or Document PiP) that stays open while the opener navigates:
- **Disconnected UX**: Popups live in separate floating OS windows (or separate browser tabs on mobile), occluding the main page content and feeling unintegrated with the site.
- **Browser Abuse Protections**: Popups require transient user activation and are frequently blocked by popup blockers.

---

## Proposal: The `<persistentwidget>` Element

We propose a dedicated HTML element, `<persistentwidget>`, which acts as a **browsing context container** hosting an **independent top-level browsing context** that persists across same-origin top-level document navigations when both the navigating document and the destination document declare matching widgets.

### Basic Example

```html
<!-- Page A: https://example.com/index.html (before navigation) -->
<persistentwidget id="assistant" src="/assistant.html"></persistentwidget>

<!-- Page B: https://example.com/dashboard.html (after same-origin navigation) -->
<head>
  <!-- Ensure rendering is blocked until the widget is parsed and adopted -->
  <link rel="expect" href="#assistant" blocking="render">
</head>
<body>
  <persistentwidget id="assistant" src="/assistant.html"></persistentwidget>
</body>
```

When the top-level document navigates from Page A to Page B:
1. The browser retains the underlying browsing context for `/assistant.html`.
2. Upon loading Page B, the browser matches the `<persistentwidget>` element with the same `id` and resolved `src` URL.
3. The existing browsing context is reattached to Page B's `<persistentwidget>` element without reloading or resetting JavaScript execution.
4. An `openerchange` event is dispatched inside the widget to notify it that its opener page has navigated.

---

## Navigation Lifecycle & Visual Continuity

### Cross-Document View Transitions Integration
Persistent widgets integrate cleanly with Cross-Document View Transitions:
- If `<persistentwidget>` is assigned a `view-transition-name` (e.g., `view-transition-name: chat-widget;`), the browser suppresses capturing static `::view-transition-old` and `::view-transition-new` snapshot pairs for the widget.
- Instead, the live, running widget is smoothly transformed and animated from its layout geometry on Page A to its layout geometry on Page B without cross-fading or visual hitches.

### Back-Forward Cache (BFCache) Compatibility
- When Page A navigates to Page B, Page A may be preserved in the Back-Forward Cache (BFCache).
- Before Page A enters BFCache, the persistent widget is **detached** from Page A's document.
- Because the widget runs in a **distinct browsing context group** without direct `WindowProxy` handles, Page A remains eligible for BFCache without keeping a live, cross-document window connection.

---

## Communication and Lifecycle

### 1. Embedder to Widget Communication

Because persistent widgets run in an independent browsing context in a distinct browsing context group (rather than a nested browsing context within the embedder's document tree) and survive outer document destruction, direct script access (`contentWindow`, `parent`, `top`) is disallowed. Communication between the embedder document and the persistent widget occurs asynchronously via `postMessage`:

```javascript
// Outer document (Embedder)
const widget = document.querySelector('persistentwidget#assistant');

// Send message to the widget
widget.postMessage({ type: 'GREET', text: 'Hello from page!' }, '*');

// Listen for messages from the widget
widget.addEventListener('message', (event) => {
  console.log('Received from widget:', event.data);
});
// Messages also bubble / dispatch to window
window.addEventListener('message', (event) => {
  if (event.source === widget) {
    console.log('Widget sent:', event.data);
  }
});
```

### 2. Widget to Embedder Communication (`window.persistentWidgetOpener`)

Inside the persistent widget document, the global `window.persistentWidgetOpener` object acts as the handle to the current embedder page:

```javascript
// Inside the widget document (/assistant.html)
window.addEventListener('message', (event) => {
  console.log('Received from opener page:', event.data);

  // Reply back to the opener page
  if (window.persistentWidgetOpener) {
    window.persistentWidgetOpener.postMessage({ type: 'REPLY', status: 'OK' }, '*');
  }
});
```

### 3. Opener Navigation Notifications (`openerchange` event)

When the top-level embedder document navigates to a new page, the persistent widget remains alive and its opener handle is updated. The browser fires an `openerchange` event on `window` inside the widget:

```javascript
// Inside the widget document (/assistant.html)
window.addEventListener('openerchange', () => {
  console.log('The embedder document has navigated!');

  // Re-synchronize with the new page
  if (window.persistentWidgetOpener) {
    window.persistentWidgetOpener.postMessage({
      type: 'SYNC_STATE',
      activeSessionId: currentSession.id
    }, '*');
  }
});
```

### 4. Message Ordering, Time-of-Use (TOU), and Transferables

Because the embedder page can navigate while messages are in flight across processes:
- **Ordering with `openerchange`**: Incoming `postMessage` messages and `openerchange` notifications inside the persistent widget must be queued on the same task queue so that all messages sent by an old opener before navigation are delivered (or dropped) prior to the `openerchange` event firing.
- **Time-of-Use (TOU) Protection**: Messages sent by the widget via `window.persistentWidgetOpener.postMessage()` are bound to the specific opener document active at the time of sending. If the opener document navigates before the message is delivered, the message is dropped rather than delivered to the new destination document.
- **Transferable Objects (`MessagePort`, `SharedArrayBuffer`)**: Passing transferable handles such as `MessagePort` or shared memory across `postMessage` could allow persistent channels to bridge across separate navigations, bypass policies (such as `BroadcastChannel` restrictions), or keep Back-Forward Cache (BFCache) documents entangled with active widgets. To avoid these hazards, transferring `MessagePort`s and other transferable objects is disallowed (or ports are automatically disentangled/closed upon opener change). This could be revisited if use cases for transferables come up if we can find a way around these issues.

---

## Adoption and Lifetime Rules

For an existing persistent widget to survive a top-level navigation, all of the following conditions must be met:

1. **Same-Origin Navigation**: The navigation must be to a same-origin document.
2. **Matching Key (`src` + `id`)**: The destination document must include a `<persistentwidget>` element whose resolved `src` URL and `id` attribute match the existing widget.
3. **Permissions Policy Compatibility**: The destination document must have a matching Permissions Policy with the document that originally created the widget. If the new document specifies a stricter or different policy (e.g. disabling geolocation), the widget is destroyed to prevent policy bypasses.
4. **Attachment Before First Render**: The matching `<persistentwidget>` must be attached to the destination DOM before the new document's first render, where the [pagereveal event is fired](https://html.spec.whatwg.org/multipage/browsing-the-web.html#reveal).
5. **Tab Scoping**: Persistent widgets are strictly scoped to the top-level browser tab in which they were created. A persistent widget cannot be shared with or adopted by a document in another tab or window.

### Delayed First Render

If a page dynamically creates or appends `<persistentwidget>` elements via JavaScript or places them late in the HTML parser stream, it can delay first render to ensure the widget is attached before the adoption deadline:
- Using `<link rel="expect" href="#widget-id" blocking="render">`
- Using standard parser-blocking scripts

#### Orphaned / Unadopted Widget Teardown & Error Handling

- **Unadopted Teardown**: If the destination document finishes its first render without adopting an existing persistent widget, the browser destroys the widget instance.
- **Late Attachment**: Any `<persistentwidget>` appended *after* the first render will initialize a fresh browsing context rather than adopting the previous instance.
- **Navigation Errors / Error Pages**: If the top-level navigation fails (e.g., network error resulting in an error page), the widget remains bound to the old document if the old document is not unloaded.

---

## Architecture & Design Principles

### Why `<persistentwidget>` Instead of `<iframe persist>`?

Rather than modifying `<iframe>`, Persistent Widgets define a distinct **browsing context container** hosting an **isolated top-level browsing context**:

- **Isolated Browsing Context Group**: A persistent widget runs in an independent browsing context in a separate browsing context group, decoupled from the embedder's browsing context tree.
- **No Dangling References**: Standard nested browsing contexts (`<iframe>`) expose synchronous DOM properties like `iframe.contentWindow`, `window.parent`, and `window.top`. If an iframe survived across navigations while its parent document was garbage collected, references across the boundary would create severe lifetime, memory, and security hazards.
- **Exclusion from `window.frames`**: Like `<fencedframe>`, `<persistentwidget>` elements are not listed in the embedder document's `window.frames` collection (`window.length` / indexed frame access), preventing synchronous cross-tree frame enumeration.
- **Clean BFCache Boundaries**: Isolating the widget into its own browsing context group allows the navigating page to enter BFCache cleanly without cross-page script references.

### Supporting Third-Party / Cross-Origin Services

While the persistent widget's top-level document must be same-origin with the embedder, **third-party cross-origin content can easily be embedded inside the widget via standard `<iframe>`s**:

```html
<!-- Inside /assistant.html (same-origin with embedder) -->
<iframe src="https://third-party-ai-service.example/chat" allow="microphone"></iframe>
```

This pattern ensures:
1. The widget's root frame structurally mirrors the top-level page's security headers and Permissions Policy.
2. The third-party content runs securely inside standard iframe sandboxing within the widget.

---

## Security & Privacy Considerations

Because `<persistentwidget>` renders embedded content inline within the document without separate browser window decorations (like an address bar), its threat model is similar to iframes. Here are some security and privacy semantics for persistent widgets which will be made to match iframes:

- **Framing Restrictions**: CSP `frame-ancestors` and `X-Frame-Options` headers dictate whether a target URL is allowed to be loaded within a widget.
- **Cookie Security**: The widget's document respects `SameSite` cookie semantics, evaluating the top-level embedder site as the site-for-cookies rather than treating the widget as an independent top-level document.
- **Cross-Origin Isolation**: The widget abides by the embedder's Cross-Origin Embedder Policy (COEP). If the embedder has COEP enabled, any cross-origin subresources must respond with the `Cross-Origin-Resource-Policy` header.
- **Fetch Metadata**: Resource requests for the widget include standard subframe Fetch Metadata headers (`Sec-Fetch-Dest: iframe`, `Sec-Fetch-Site: same-origin`, `Sec-Fetch-Mode: navigate`).
- **Sandboxing**: Sandbox flags are strictly inherited by any nested browsing contexts or popups created within the widget.
- **Permissions Policy Enforcement**: The widget's access to permission-gated APIs (e.g., camera, microphone, on-device AI) is strictly governed by the top-level document via the `Permissions-Policy` HTTP header and the `allow` attribute.
- **UX & Permission Attribution**: Any user-facing indicators for resource usage triggered by the widget (tab audio indicator, mute controls, mic/camera recording badges) are visually attributed to the top-level tab.
- **Storage Partitioning**: Storage APIs (`localStorage`, `indexedDB`, `caches`, cookies) and network state accessed by the widget are partitioned by the top-level site.

---

## Alternatives Considered

### 1. Companion Windows / Imperative Detached Window API

An alternative approach is explored in the [Companion Windows explainer](https://github.com/explainers-by-googlers/companion-windows/), which uses an imperative JavaScript API (`companionWindows.open(...)`) to manage auxiliary floating, docked, or anchored windows attached to a top-level tab across navigations:

```javascript
// Opening an auxiliary companion window
companionWindows.open('/player.html', { name: 'player' }).then(playerWindow => {
  // Communicate with the companion window
  playerWindow.contentWindow?.postMessage({ action: 'PLAY' }, '*');
});
```

**Why `<persistentwidget>` is preferred for embedded in-page widgets**:
- **Inline Layout & CSS Integration**: Floating or docked auxiliary windows render in separate UI overlays above the content area. In contrast, `<persistentwidget>` is an inline replaced element that participates directly in the document's normal layout flow, responsive CSS design, and flexbox/grid without occluding surrounding content.
- **Declarative HTML & Early Adoption**: Declaring `<persistentwidget>` directly in markup allows the browser to parse and adopt the widget during the initial render-blocking phase before the page paints.
- **View Transitions Integration**: As a first-class DOM element, `<persistentwidget>` integrates naturally with Cross-Document View Transitions, smoothly animating between different sizes and layout positions across page changes.

### 2. Adding a `persist` attribute to `<iframe>`

**Why this was not chosen**:
- Iframes expose synchronous `window.parent`, `window.top`, and `iframe.contentWindow` properties. Maintaining live pointers across top-level navigations while the old document is destroyed or placed in BFCache creates severe memory safety, lifetime, and security hazards. We could try making `<iframe persist>` be the same as a persistent widget under the hood by removing all of the things we don't want from iframes when this attribute is set, which would make progressive enhancement better but would also confuse people who think its more like an iframe than it actually is.

---

## Future Work & Open Questions

- **Transferable Objects (`MessagePort`, `SharedArrayBuffer`) & BFCache / BroadcastChannel Policies**: Should `postMessage` on `<persistentwidget>` and `window.persistentWidgetOpener` allow transferring `MessagePort`s and shared memory (`SharedArrayBuffer`)? Allowing open `MessagePort`s across navigation boundaries could circumvent `BroadcastChannel` policies or prevent previous embedder documents from cleanly entering BFCache.
- **Prerendering (Speculation Rules) Interactions**: How should `<persistentwidget>` behave when a destination page is prerendered in the background while the active page is still displaying the widget? Prerendered pages must not prematurely steal or detach the persistent widget from the currently active tab until prerender activation occurs.
- **Browser Extension Integration (`chrome.webNavigation`, `chrome.tabs`, Content Scripts)**: How should persistent widgets be exposed to browser extensions?
- **Same-Document DOM Reparenting ([WHATWG Issue #5484](https://github.com/whatwg/html/issues/5484))**: In standard HTML, moving an `<iframe>` to a different place in the DOM destroys and reloads the frame. Because `<persistentwidget>` contexts can exist temporarily detached from a DOM node, the same mechanism could be extended to allow seamless in-page reparenting without reloads.
- **Directly Embedding Cross-Origin Widget Documents**: Currently, `<persistentwidget src="...">` requires its root document to be same-origin with the embedder (with third-party content nested inside a child `<iframe>`). Could persistent widgets be extended in the future to allow directly loading a cross-origin root document (e.g., `<persistentwidget src="https://third-party-service.example/widget.html">`) that persists across same-origin top-level navigations? This would require explicit mutual opt-in headers from both the embedder and the embedded origin, along with deeper exploration into cross-origin security policies, storage partitioning, and capability delegation.
- **Media & Animation Continuity**: Could video playback continues uninterrupted? Could CSS/JS animations in the widget keep producing frames? The browser compositor would re-composit the live widget on top of the retained background textures during paint holding.

---

## Frequently Asked Questions (FAQ)

### What happens if the navigated-to page has no matching `<persistentwidget>`?
If the new document does not attach a matching `<persistentwidget>` element by the time its first frame is rendered, the browser terminates and cleans up the persistent widget.

### What happens when navigating cross-origin?
All persistent widgets associated with the previous origin are immediately destroyed upon cross-origin navigation.

### What happens if a `<persistentwidget>` is removed from the DOM?
Removing the element from the DOM destroys the widget's browsing context, discards its document, and invalidates any subsequent `postMessage` calls.
