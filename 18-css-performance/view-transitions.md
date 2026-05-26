# View Transitions API

## Browser Support

| Browser | Version | Notes |
|---------|---------|-------|
| Chrome / Edge | 111+ | Same-document (SPA) |
| Chrome / Edge | 126+ | Cross-document (MPA) |
| Safari | 18+ | Same-document stable |
| Firefox | 134+ | Same-document stable |

Source: MDN/caniuse — accessed 2026-05-26

## Same-Document (SPA) Transition

```js
// Check support first
if (!document.startViewTransition) {
  // Fallback: instant swap
  updateDOM();
  return;
}

// Wrap DOM update in transition
const transition = document.startViewTransition(() => {
  // Any DOM mutation inside this callback
  mainContent.innerHTML = newPageHTML;
  document.title = newTitle;
});

// Optional: wait for completion
await transition.ready;       // CSS animations can start
await transition.finished;    // Transition fully done
await transition.updateCallbackDone;  // Callback resolved
```

## CSS Control

```css
/* Default cross-fade — ::view-transition-old and ::view-transition-new */
::view-transition-old(root) {
  animation: 200ms ease out fade-out;
}
::view-transition-new(root) {
  animation: 200ms ease in fade-in;
}

@keyframes fade-out { to { opacity: 0; } }
@keyframes fade-in  { from { opacity: 0; } }

/* Named elements — slide instead of fade */
.hero-image {
  view-transition-name: hero;  /* Must be unique per page */
}

::view-transition-old(hero) {
  animation: 300ms ease slide-out-left;
}
::view-transition-new(hero) {
  animation: 300ms ease slide-in-right;
}
```

## Cross-Document (MPA) Transition

```css
/* page-a.css — outgoing */
@view-transition {
  navigation: auto;  /* Enable cross-doc transitions */
}

/* Customize the outgoing page animation */
::view-transition-old(root) {
  animation: 250ms ease slide-out;
}
```

```css
/* page-b.css — incoming */
@view-transition {
  navigation: auto;
}

::view-transition-new(root) {
  animation: 250ms ease slide-in;
}
```

## Transition Types (Chrome 125+)

```js
document.startViewTransition({
  update: () => updateDOM(),
  types: ['slide', 'forwards']
});
```

```css
/* Style based on transition type */
:active-view-transition-type(slide) {
  &::view-transition-old(root) { animation: slide-out; }
  &::view-transition-new(root) { animation: slide-in;  }
}
```

## Reduced Motion Respect

```css
@media (prefers-reduced-motion: reduce) {
  ::view-transition-group(*),
  ::view-transition-old(*),
  ::view-transition-new(*) {
    animation: none !important;
  }
}
```

## Sources
- Chrome developers: https://developer.chrome.com/docs/web-platform/view-transitions/
- MDN: https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API
- Jake Archibald explainer: https://jakearchibald.com/2024/view-transitions-handling-aspect-ratio-changes/
