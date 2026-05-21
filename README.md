# Persistent iframes

## Problem
Hard navigations of top-level documents result in the destruction of all the child iframes of those documents.
In cases of same-origin navigations, there's high likelihood that at least some of those iframes would need to be recreated on the new document.
For iframes maintaining complex client-side state (e.g. chat widgets, media players, collaborative editing sessions, payment flows, embedded third-party apps), that results in loss of state and user-visible performance implications.

This significantly disadvantages MPA architectures (that rely on hard same-origin navigations) compared to SPA architectures, that can soft-navigate parts of the document while leaving the iframes intact.
That is particularly problematic for the use-case of [in-page agents](https://github.com/webmachinelearning/webmcp/issues/51), and runs a risk of causing web developers to prefer SPAs due to that, despite potential performance and maintenance costs related to that preference.

This proposal aims to create a web-platform mechanism that would enable us to persist iframes across same-origin navigations of their parent.

## Proposal
Allow a cross-origin iframe to survive a same-origin top-level navigation when both the old and new documents opt in via a `persist` attribute and matching `id`s. 

``` html
<!-- Page A (before navigation) -->
<iframe src="https://chat.example.com" id="sidebar-chat" persist></iframe>

<!-- Page B (after navigation to same-origin URL) -->
<iframe src="https://chat.example.com" id="sidebar-chat" persist></iframe>
```

When the top-level frame navigates from page A to page B (same-origin), the browser will preserve the iframe, and reattach it an `HTMLIframeElement` on the navigated-to document that has the same `src` value and the same `id`.

For UX reasons, the iframe would need to be matched and reattached before the new document's first render. If for some reason developers would need to add that iframe using script or at the bottom of their HTML, they'd be able to delay the first render using [`<link rel=expect>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel#expect).

For security reasons, persistent iframes will be limited to same-origin navigations.

## FAQs

### What happens if the navigated-to page has no matching iframe?

If by first render the iframe finds no match, it'd get killed at that point.
Up until that point it'll remain alive and would be able to e.g. play audio or fetch content.

### What happens when the navigation is to a cross-origin document?
In that case, the iframe would be killed immediately, similar to when it'd be killed today.

## Open questions
* Would limiting this to cross-origin or process-isolated iframes help simplify the spec/implementation?
  - It seems like it would, as we won't need to worry about dangling references to destructed objects in the navigated parent document
* Would we be able to expand this to also tackle cross-origin top-level navigations, using explicit opt-in from the parent documents and from the iframe?
