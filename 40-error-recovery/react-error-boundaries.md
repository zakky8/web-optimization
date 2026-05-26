# React Error Boundaries for R3F Canvas

## Why Error Boundaries Are Necessary for R3F

React Three Fiber runs Three.js inside the React reconciler. Any exception thrown during rendering of an R3F component — including Three.js internal errors, shader compilation failures, missing textures, or geometry processing errors — propagates up the React component tree. Without an error boundary, a single bad component unmounts the entire React tree and leaves the user with a blank white page and a console error.

Error boundaries catch these during the React render phase. They do not catch:
- Errors in event handlers
- Errors in async code (`setTimeout`, `requestAnimationFrame` callbacks)
- Errors thrown by the error boundary itself
- Server-side rendering errors

Source: React documentation, `Component` reference — "Catching rendering errors with an error boundary", accessed 2026-05-26.

---

## The Canonical Error Boundary Class Component

React requires a **class component** to implement an error boundary. There is no hook-based equivalent in the standard React API (as of React 18). The `react-error-boundary` package provides a function-component wrapper.

```jsx
import React from 'react';

class CanvasErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  // (1) Called synchronously during rendering when a child throws.
  //     Return value is merged into state. Pure — no side effects here.
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // (2) Called after the render phase — use for side effects (logging, reporting).
  //     info.componentStack is the React component stack as a string.
  componentDidCatch(error, info) {
    console.error('[CanvasErrorBoundary] Three.js/R3F render error:', error);
    console.error('Component stack:', info.componentStack);

    // Pass to parent handler if provided
    if (typeof this.props.onError === 'function') {
      this.props.onError(error, info);
    }
  }

  render() {
    if (this.state.hasError) {
      // Render fallback UI — replace the canvas entirely
      return this.props.fallback ?? (
        <div role="alert" style={{ padding: '1rem', background: '#111', color: '#eee' }}>
          <p>3D rendering encountered an error and stopped.</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Retry
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

Source: React docs — `getDerivedStateFromError` and `componentDidCatch` patterns, accessed 2026-05-26.

---

## Wrapping the R3F `<Canvas>`

```jsx
import { Canvas } from '@react-three/fiber';

function App() {
  return (
    <CanvasErrorBoundary
      fallback={<StaticFallbackScene />}
      onError={(err, info) => reportError(err, info)}
    >
      <Canvas>
        <Scene />
      </Canvas>
    </CanvasErrorBoundary>
  );
}
```

### R3F's own `fallback` prop

R3F `<Canvas>` also accepts a `fallback` prop for the specific case where WebGL is not available (feature detection failure, not a runtime error). This is distinct from an error boundary — it handles the "no WebGL" path, not crash recovery. Source: R3F Canvas API, accessed 2026-05-26.

```jsx
<Canvas fallback={<div>WebGL not supported.</div>}>
  <Scene />
</Canvas>
```

For runtime errors (thrown during rendering), you still need an error boundary wrapping the `<Canvas>`.

---

## R3F `onError` Prop

R3F `<Canvas>` does not expose a top-level `onError` prop for component-tree errors as of the current API (R3F Canvas API docs, accessed 2026-05-26 — this prop is absent from the documented interface). Use an outer error boundary class component or the `react-error-boundary` package instead.

For errors during the Three.js GL context creation specifically, listen on the canvas element directly:

```js
// Access the underlying canvas from the R3F store
import { useThree } from '@react-three/fiber';

function ContextErrorLogger() {
  const gl = useThree((state) => state.gl);

  React.useEffect(() => {
    const canvas = gl.domElement;
    const handler = (event) => {
      console.error('WebGL context creation error:', event.statusMessage);
    };
    canvas.addEventListener('webglcontextcreationerror', handler);
    return () => canvas.removeEventListener('webglcontextcreationerror', handler);
  }, [gl]);

  return null;
}
```

---

## `react-error-boundary` Package

The `react-error-boundary` package (https://github.com/bvaughn/react-error-boundary) is the idiomatic modern approach. It wraps the class component pattern in a function-component-friendly API.

```bash
npm install react-error-boundary
```

```jsx
import { ErrorBoundary } from 'react-error-boundary';

function CanvasFallback({ error, resetErrorBoundary }) {
  return (
    <div role="alert">
      <p>3D scene crashed: {error.message}</p>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary
      FallbackComponent={CanvasFallback}
      onError={(error, info) => reportError(error, info)}
      onReset={() => {
        // Reset any application state that caused the error
        resetScene();
      }}
    >
      <Canvas>
        <Scene />
      </Canvas>
    </ErrorBoundary>
  );
}
```

The `resetErrorBoundary` function passed to `FallbackComponent` triggers `onReset` and clears the error state in one call. This allows users to retry without a page reload.

---

## Sentry Integration

Sentry's React SDK ships its own `ErrorBoundary` component with automatic capture. Source: Sentry React error boundary docs, accessed 2026-05-26.

```bash
npm install @sentry/react
```

```jsx
import * as Sentry from '@sentry/react';
import { Canvas } from '@react-three/fiber';

function App() {
  return (
    <Sentry.ErrorBoundary
      fallback={({ error, resetError }) => (
        <div role="alert">
          <p>Renderer error: {error.message}</p>
          <button onClick={resetError}>Retry</button>
        </div>
      )}
      onError={(error, componentStack) => {
        // Sentry automatically captures the error; this callback is for additional logic
        console.error('R3F error captured by Sentry:', error.message);
      }}
      showDialog // Shows Sentry user feedback widget
    >
      <Canvas>
        <Scene />
      </Canvas>
    </Sentry.ErrorBoundary>
  );
}
```

`Sentry.ErrorBoundary` automatically attaches the React component stack to the Sentry event. The `fallback` prop accepts either a React element or a function receiving `{ error, componentStack, resetError }`. Source: Sentry React error boundary docs, accessed 2026-05-26.

### Sentry initialization (required before using ErrorBoundary)

```js
Sentry.init({
  dsn: 'https://YOUR_DSN@sentry.io/YOUR_PROJECT',
  integrations: [Sentry.browserTracingIntegration()],
  tracesSampleRate: 0.1,
});
```

---

## Reporting Without Sentry

If you are not using Sentry, a lightweight approach sends errors to your own endpoint:

```js
async function reportError(error, info) {
  try {
    await fetch('/api/errors', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message:        error.message,
        stack:          error.stack,
        componentStack: info?.componentStack,
        url:            location.href,
        timestamp:      new Date().toISOString(),
        userAgent:      navigator.userAgent,
      }),
    });
  } catch {
    // Do not let error reporting itself crash the app
  }
}
```

---

## Reset Strategy

A "Retry" button works when the error was transient (network glitch, temporary GPU state). For structural errors (bad model, broken shader), retry will loop. Distinguish them:

```jsx
function CanvasFallback({ error, resetErrorBoundary }) {
  const isTransient = error.message.includes('context') || error.message.includes('network');

  return (
    <div role="alert">
      <p>Scene error: {error.message}</p>
      {isTransient && (
        <button onClick={resetErrorBoundary}>Retry</button>
      )}
      {!isTransient && (
        <p>This error requires a page reload to resolve.</p>
      )}
    </div>
  );
}
```

---

## Error Boundary Placement

Place the boundary as close to the Three.js code as possible, not at the app root. This lets the rest of your UI (navigation, forms) continue working when only the 3D canvas fails.

```
<App>
  <Header />          ← keeps working on canvas error
  <CanvasErrorBoundary>
    <Canvas>
      <Scene />
    </Canvas>
  </CanvasErrorBoundary>
  <Footer />          ← keeps working on canvas error
</App>
```

---

## Sources

| Source | URL | Tier | Accessed |
|---|---|---|---|
| React docs — Error boundary pattern | https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary | 1 | 2026-05-26 |
| Sentry React ErrorBoundary | https://docs.sentry.io/platforms/javascript/guides/react/features/error-boundary/ | 1 | 2026-05-26 |
| R3F Canvas API | https://r3f.docs.pmnd.rs/api/canvas | 1 | 2026-05-26 |
