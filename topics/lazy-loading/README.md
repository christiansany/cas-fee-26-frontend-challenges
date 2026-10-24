# Loading behaviour: lazy loading with React

> Your page sends every component, image and data request at once, even the parts nobody scrolls to. Let's load those parts only when the user is about to see them, with the `IntersectionObserver`, the native `loading="lazy"` attribute and React's `lazy`. We fix the layout shifts this causes with skeletons. Then we turn it around and load the next page before the user clicks.

**Duration:** about 45 minutes

## Contents

* [The problem](#the-problem)
* [Lazy loading](#lazy-loading)
* [Lazy loading in React](#lazy-loading-in-react)
* [Putting it all together](#putting-it-all-together)
* [Load before the click: prefetching](#load-before-the-click-prefetching)
* [Summary](#summary)

## The problem

Our websites and apps contain many components. When a user visits our site, they wait until everything is loaded, parsed and executed. On slow connections and older devices this can take a long time. That hurts the user experience. The longer the page takes to load, the higher the bounce rate. This lowers the conversion rate and also hurts SEO, because search ranking takes the [Core Web Vitals](https://web.dev/explore/learn-core-web-vitals) into account.

Websites and apps also depend on internal communication between services, especially during server-side rendering (SSR). If one service is slow, it can slow down the whole app. This happens, for example, when a service runs a slow database query. The app has to wait until the query finishes before it can continue. The result is a long loading time.

![Loading Problem](./assets/loading-problem.png)

## Lazy loading

### What is lazy loading?

Lazy loading is a technique that loads resources such as images or JavaScript only when they are needed, for example when they appear in the visible area of the page (the viewport). This shortens the initial loading time and improves the user experience, because the browser does not load large files up front.

![Loading Solution](./assets/loading-solution.png)

### Native lazy loading for images and iframes

For images and iframes you don't need JavaScript. Add `loading="lazy"` and the browser defers the request until the element comes within a certain distance of the viewport. Every current browser supports it for both elements.

```html
<img src="item-photo.jpg" alt="Camera" width="800" height="600" loading="lazy">
<iframe src="https://example.com/map" title="Pick-up location" loading="lazy"></iframe>
```

Set `width` and `height`, so the browser can reserve the space before the image arrives. Never lazy-load the images visible on first load, especially the largest one: that delays your Largest Contentful Paint (LCP). For that image, `fetchpriority="high"` tells the browser to fetch it before other images.

Docs: [MDN: Lazy loading](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Lazy_loading), [web.dev: Browser-level image lazy loading](https://web.dev/articles/browser-level-image-lazy-loading), [web.dev: Fetch Priority](https://web.dev/articles/fetch-priority)

For everything else (JavaScript modules, data, components) you need to know yourself when an element enters the viewport. That is the job of the `IntersectionObserver`.

### IntersectionObserver

The `IntersectionObserver` is a JavaScript API that lets you watch when an HTML element enters or leaves the viewport. You can use it to trigger actions at exactly that moment.

```html
<div class="page page-1">
  <h1>Page 1</h1>
</div>
<div class="page page-2">
  <h1>Page 2</h1>
</div>
<div class="page page-3">
  <h1>Page 3</h1>
</div>
<div class="page page-4">
  <h1>Page 4</h1>
</div>
```

```js
// Create the IntersectionObserver instance
// We provide a callback that is run when the Observer detects a change 
// (and when an element is first observed)
const observer = new IntersectionObserver((entries) => {
  // entries contains all IntersectionObserver Entries where a change was detected
  entries.forEach(entry => {
    // Toggles the class in-view, depending on entry.isIntersecting
    entry.target.classList.toggle("in-view", entry.isIntersecting)
  })
})

// Tell our IntersectionObserver to observe our elements
document.querySelectorAll(".page").forEach(page => {
  observer.observe(page);
})
```

**Demo** 🤯

- [IntersectionObserver Basic](https://codesandbox.io/s/2sk7t8?file=/src/index.mjs)

#### The [`rootMargin`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver/IntersectionObserver#rootmargin) option

The `rootMargin` option defines an offset around the viewport. Use it to tell the `IntersectionObserver` to run the callback before the element actually enters the viewport. At that moment the element already counts as `isIntersecting`.

```js
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    entry.target.classList.toggle("in-view", entry.isIntersecting);
  });
}, {
  // Element is considered intersecting 200px before it enters the viewport
  // This works for both directions (top and bottom)
  rootMargin: "200px",
});
```

**Demo** 🤯

🚧 CodeSandbox has problems with `rootMargin`.
To see the effect, open the sandbox result in a new window.

- [IntersectionObserver rootMargin (sandbox)](https://codesandbox.io/s/28rdfr?file=/src/index.mjs)
- [IntersectionObserver rootMargin (result)](https://28rdfr.csb.app/)

#### The [`threshold`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver/IntersectionObserver#threshold) option

The `threshold` option defines how much of the element, as a fraction, must be in the viewport before it counts as `isIntersecting`. It also accepts an array of values to define several thresholds.

```js
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    entry.target.classList.toggle("in-view", entry.isIntersecting);
  });
}, {
  // Element is considered intersecting even when only one pixel is visible.
  // This is the default value.
  threshold: 0,

  // Element is considered intersecting when 25% of it is visible.
  threshold: 0.25,

  // Element is considered intersecting when 99% of it is visible.
  threshold: 0.99,
});
```

**Demo** 🤯

- [IntersectionObserver threshold](https://codesandbox.io/s/d2xxff?file=/src/index.mjs)

#### Observing only once

You can also use the `IntersectionObserver` to observe an element only once. As soon as the element enters the viewport for the first time, you stop observing it.

```js
const observer = new IntersectionObserver((entries, observer) => {
  entries.forEach(entry => {
    entry.target.classList.toggle("in-view", entry.isIntersecting);
    console.log({ element: entry.target.className, isIntersection: entry.isIntersecting});

    // Stop observing the element, once it is intersecting
    if(entry.isIntersecting) {
      console.log(`No longer observing ${entry.target.className}`);
      observer.unobserve(entry.target);
    }
  });
}, {
  threshold: 0.25,
});

document.querySelectorAll(".page").forEach(page => {
  observer.observe(page);
});
```

**Demo** 🤯

- [IntersectionObserver Once](https://codesandbox.io/s/ldppv2?file=/src/index.mjs)

> **Key points**
>  
> * The `IntersectionObserver` runs a callback when an element enters or leaves the viewport
> * With `rootMargin` you can run the callback before the element enters the viewport
> * With `threshold` you define how much of the element must be in the viewport before it counts as `isIntersecting`
> * You can use the `IntersectionObserver` for all kinds of side effects, tracking for example

**Useful links**

* [IntersectionObserver](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver)

### Lazy loading in action

With the `IntersectionObserver` you can now implement lazy loading. You load resources such as JavaScript modules, data or images only when they are needed. This reduces the initial loading time and takes load off the main thread, which has less to process at the start. All of this improves the user experience.

**Demo** 🤯

- [Lazy Loading in action](https://codesandbox.io/s/nmm4kc?file=/src/index.js)

## Lazy loading in React

In React, `lazy` lets you load a component's code the first time it renders. Instead of importing the component directly with `import LazyComponent from './LazyComponent.tsx'`, you create a new component with `lazy` that runs a dynamic import of `LazyComponent`.

Three rules apply. The module must export the component as its `default` export. Declare the lazy component at the top level of the module, never inside another component, or its state resets on every render. And while the code is loading, the component suspends, so wrap it in a `<Suspense>` boundary with a fallback.

```tsx
import { lazy, Suspense } from "react";

const LazyComponent = lazy(() => import("./LazyComponent.tsx"));

export default function App() {
  return (
    <div className="App">
      <Suspense fallback={<p>Loading…</p>}>
        {/* When this is rendered, the Component is downloaded */}
        <LazyComponent />
      </Suspense>
    </div>
  );
}
```

Docs: [react.dev: lazy](https://react.dev/reference/react/lazy), [react.dev: Suspense](https://react.dev/reference/react/Suspense)

Instead of working with the `IntersectionObserver` directly in React, you can use the [`react-intersection-observer` package](https://www.npmjs.com/package/react-intersection-observer). It is actively maintained and supports React 19. It uses the `IntersectionObserver` under the hood and gives you a hook that tells you whether an element is in the viewport.

```tsx
import { lazy, Suspense } from "react";
import { useInView } from "react-intersection-observer";

const LazyComponent = lazy(() => import("./LazyComponent.tsx"));

export default function App() {
  const { ref, inView } = useInView({
    triggerOnce: true
  });

  return (
    <div className="App">
      <div className="page page-1">
        <h1>Page 1</h1>
      </div>

      {/* The ref has to be placed on the element we want to observe with the 
          IntersectionObserver */}
      <div ref={ref}>
        {/* LazyComponent will render if the parent is intersecting */}
        <Suspense fallback={null}>
          {inView ? <LazyComponent /> : null}
        </Suspense>
      </div>

      <div className="page page-3">
        <h1>Page 3</h1>
      </div>
    </div>
  );
}
```

**Demo** 🤯

- [React Lazy with IntersectionObserver (sandbox)](https://codesandbox.io/s/dvjl33?file=/src/App.tsx)
- [React Lazy with IntersectionObserver (result)](https://dvjl33.csb.app/)

### Layout shifts

As the demo shows, Page 3 behaves strangely: it moves down as soon as the component on Page 2 has loaded. This is called a layout shift. Layout shifts hurt the user experience a lot, because the content of the page suddenly jumps. They also cost you in SEO ([Cumulative Layout Shift](https://web.dev/articles/cls)).

Layout shifts happen when elements on the page are not ready yet when the user first sees them. But sometimes we don't want to show content from the start. So what can we do against these layout shifts?

**Demo** 🤯

- [Pokémon list with layout shifts (sandbox)](https://codesandbox.io/s/kzdh37?file=/src/App.tsx)
- [Pokémon list with layout shifts (result)](https://kzdh37.csb.app/)

The demo shows that the many Pokémon names on screen trigger our `useInView` hooks, so the app loads data for far too many Pokémon. That is not what we want.

The solution is to reserve the space for the content from the start. One way to do this is with skeletons. A skeleton looks roughly like the real content, without the details. Because we don't know what content the backend will return, we can only reserve the space approximately. That is not perfect, but much better than nothing.

![Skeletons](./assets/skeletons.png)

**Useful links**

- [Skeleton Screen - Galaxus](https://www.galaxus.ch/designsystem/patterns/skeleton-screen)

## Putting it all together

Let's combine everything. The result performs well and gives the user a good experience.

- [Pokémon list with skeletons (sandbox)](https://codesandbox.io/s/58tdkj?file=/src/Pokemon.tsx)
- [Pokémon list with skeletons (result)](https://58tdkj.csb.app/)

## Load before the click: prefetching

Lazy loading saves work on the current page. It has a cost on the next one: when the user clicks a link, the code and data for that page have not been loaded yet, and the user waits. Prefetching means loading the resources for a likely next step in the background, before the user asks for it.

### Prefetch on intent

The strongest signal is the user's own behaviour. When the pointer rests on a link, or the link gets keyboard focus, a click is likely. Start a short timer (about 100 ms) on `mouseenter` and `focus`, and cancel it on `mouseleave` and `blur`. The delay filters out the pointer passing over a link on its way somewhere else. On touch screens there is no hover, so prefetch on `touchstart`: it fires a moment before the click.

```tsx
import { useEffect, useRef } from "react";

const INTENT_DELAY_MS = 100;

export function usePrefetchOnIntent<T extends HTMLElement>(prefetch: () => void) {
  const ref = useRef<T>(null);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const controller = new AbortController();
    const { signal } = controller;
    let timer: ReturnType<typeof setTimeout> | undefined;

    const cancel = () => clearTimeout(timer);
    const run = () => {
      cancel();
      controller.abort(); // 👇 prefetch once, then remove all listeners
      prefetch();
    };
    const start = () => { timer = setTimeout(run, INTENT_DELAY_MS); };

    element.addEventListener("mouseenter", start, { signal });
    element.addEventListener("focus", start, { signal });
    element.addEventListener("mouseleave", cancel, { signal });
    element.addEventListener("blur", cancel, { signal });
    element.addEventListener("touchstart", run, { signal, passive: true });

    return () => {
      cancel();
      controller.abort();
    };
  }, [prefetch]);

  return ref;
}
```

Pass a stable function, for example one wrapped in `useCallback`, and put the returned `ref` on the link. Inside the function you call whatever loads the next step: a `fetch()` for the data of the next page, or the `import()` of a lazy component.

### Prefetch when the browser is idle

Some links are likely without any hover, such as the "next page" link of a list. You can prefetch them when the browser has nothing else to do. `requestIdleCallback` runs a callback in an idle period of the main thread. Give it a `timeout`, or the callback may wait for seconds. Safari does not support it yet, so fall back to `setTimeout`.

Combine both conditions: prefetch a link only when it is in the viewport *and* the browser is idle. Observe the link with an `IntersectionObserver`. When it enters the viewport, schedule the prefetch with `requestIdleCallback`. When it leaves before the callback has run, cancel it with `cancelIdleCallback`. This way, a user who scrolls quickly through a long list does not trigger a prefetch for every link on the way.

Docs: [MDN: requestIdleCallback](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback)

### Why prefetching everything hurts

Prefetching is a bet: you spend the user's bandwidth, data plan and CPU on a page they might open. Every prefetch that is never used is waste. Worse, prefetches compete with the requests that matter now: the LCP image, the data for the page the user is looking at, the JavaScript that makes it interactive. On a slow mobile connection, a page with 50 prefetched links can feel slower than one with none. Your servers also pay: every prefetch is a request they have to answer, often a full page render.

That is why large sites don't guess. They run A/B tests that compare prefetching strategies against real user metrics and server cost. Often the result is to turn blanket link prefetching off and keep only targeted prefetching on intent or for a few high-value links.

### The platform option: Speculation Rules API

The browser can do this for you. With the Speculation Rules API you describe in JSON which links the browser may prefetch (download the page's HTML) or prerender (load and render the whole page in a hidden tab). `eagerness` decides when: `conservative` on pointer or touch down, `moderate` after about 200 ms of hover on desktop, `eager` and `immediate` earlier.

```html
<script type="speculationrules">
  {
    "prefetch": [{
      "where": { "href_matches": "/items/*" },
      "eagerness": "moderate"
    }]
  }
</script>
```

The rules apply to full document navigations, so they suit multi-page sites more than client-side navigation in a single-page app. As of October 2026, Chrome and Edge support them fully. Safari 26.2 and later supports prefetch only behind a feature flag, and Firefox only in preview builds. Browsers without support ignore the script, so you can add it safely as a progressive enhancement.

Docs: [MDN: Speculation Rules API](https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API), [Chrome for Developers: Prerender pages](https://developer.chrome.com/docs/web-platform/prerender-pages)

## Summary

* Load only what the user needs now. Defer the rest until it is about to enter the viewport.
* For images and iframes, use `loading="lazy"`, but never for the content visible on first load.
* For components and data, detect the viewport with the `IntersectionObserver` (or `react-intersection-observer`), and load the code with `lazy` inside a `<Suspense>` boundary.
* Reserve the space for lazy content with dimensions or skeletons, so the page does not shift.
* Prefetch the next step on intent or when the browser is idle, not everything at once, and measure whether it helps.
