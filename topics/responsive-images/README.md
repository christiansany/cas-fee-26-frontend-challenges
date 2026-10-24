# Responsive images: the implementation on post.ch and css.ch

> A 400 px wide image on a phone with a 3x display needs 1200 real pixels, and a wrong `srcset` sends a desktop-sized file over a mobile connection. We look at density and width descriptors, the `sizes` attribute, and `<picture>` for art direction and modern formats like AVIF. Then we look at how post.ch and css.ch calculate `sizes` for images in a 12-column grid. You practise the calculation yourself in a sandbox.

**Duration:** about 90 minutes

## Contents

* [Intro](#intro)
* [Responsive images](#responsive-images)
* [Implementation](#implementation)

## Intro

In this chapter we look at the responsive images implementation used on [post.ch](https://post.ch) and [css.ch](https://www.css.ch). The examples use code excerpts from both projects.

## Responsive images

### Recap: high-resolution displays

You might assume that smaller devices should always get smaller images. That is not always the case.

> "A pixel is not a pixel"

There is a difference between **CSS pixels** and **screen pixels**. CSS pixels are the unit for dimensions on our website. Screen pixels are the physical pixels the device uses.

![Pixel Density](./assets/pixel-density.jpg)

**Device pixel ratio (DPR) examples**

| Device | Native pixels | CSS pixels | Device pixel ratio (DPR) |
|---|--:|--:|:-:|
| iPhone 6, 6s, 7, 8, SE (2020) | 750 x 1334 | 375 x 667 | 2 |
| iPhone 11 Pro, X, XS | 1125 x 2436 | 375 x 812 | 3 |
| iPad Air 2020 | 2360 x 1640 | 1180 x 820 | 2 |

**Delivering sharp images**

What does this mean? How large do our images have to be so they don't look blurry on these devices?

| CSS pixels | DPR 1.0 | DPR 2.0 (4x px) | DPR 3.0 (9x px) | DPR 4.0 (16x px) |
|--:|--:|--:|--:|--:|
| 40px | 40px | 80px | 120px | 160px |
| 150px | 150px | 300px | 450px | 600px |
| 400px | 400px | 800px | 1200px | 1600px |

The table shows that at a pixel ratio of **3x** or **4x**, **the image size grows quickly** if the image is to look sharp. That is not ideal, because mobile devices usually have weaker hardware than desktop devices. On top of that, the mobile connection may be slower, which makes the problem worse.

**Unusual DPRs**

So far every device had a whole-number DPR (2.0, 3.0, and so on). Is there also a DPR of, say, 1.5? Yes, there is.

![MacBook 16" DPR](./assets/macbook-16-dpr.png)

For example, you can set up a 16" MacBook Pro so that it has a DPR of **1.5** instead of 2.0.  
This means that to show an image at 40px on this laptop, the "perfect" image would be 60px wide, not 80px as with a DPR of 2.0.

### The difficulties

With responsive images, watch out for these difficulties:

- Smaller devices should generally get smaller images
- Use the device pixel ratio so that images are as sharp as possible, without the image payload exploding
- Depending on the module or use case, the image's aspect ratio can change from device to device
- Browsers support different file formats. How can we handle and use that?

### The display density descriptor method

You can use the density descriptor inside a `srcset`. It gives the browser metadata so it can decide which image to download.  
In the example below, an iPhone 6 (DPR 2) would load `cat-400px.jpg`. An iPhone X (DPR 3) would load `cat-600px.jpg`.

*Note that the descriptor only tells the browser which image is meant for which DPR. It tells the browser nothing about how wide the image is.*

**Example**

```html
<img src="cat-200px.jpg" width="200" alt=""
     srcset="cat-200px.jpg 1x,
             cat-400px.jpg 2x,
             cat-600px.jpg 3x,
             cat-800px.jpg 4x">
```

**Pros**

This descriptor lets us force an image on the browser. Images look sharp, no matter which device we are on.

**Cons**

A device with a DPR of, say, 1.5 always downloads the next larger image. On our 16" MacBook with a DPR of 1.5, the browser has to download an image that is too large to show it sharply.

**Demo** 🤯

- [Display Density Descriptor](https://codesandbox.io/s/yb5t4)

**Useful links**

* [Use density descriptors](https://web.dev/articles/codelab-density-descriptors)

### The width descriptor method with the `sizes` attribute

Here we combine the width descriptor with the `sizes` attribute.  
The width descriptor lets us tell the browser how wide each image file is.  
This method is most useful when the width of an image changes with the size of the viewport. **This is the most common case for responsive images**.

**Example**

```html
<img src="cat-200px.jpg" alt=""
     srcset="cat-200px.jpg 200w,
             cat-400px.jpg 400w,
             cat-600px.jpg 600w,
             cat-800px.jpg 800w">
```

On its own, this information doesn't help the browser much, because it doesn't know yet how wide the image will be displayed. Because the displayed width changes with the width of the viewport, we can't use a `width` attribute either.

#### The `sizes` attribute

The `sizes` attribute tells the browser how wide an image is actually displayed. It takes either one value or several, separated by a **,** (comma). You can use media queries inside the attribute. With several values, the browser evaluates them from *left to right*. As soon as a media query matches, it uses that size as the display width of the image.

![Sizes Attr](./assets/sizes-attr.png)

**Image size condition**

There are three ways to give the width:
- In `vw`
- In `px`
- A combination of both, wrapped in `calc()`

**Example**

```html
sizes="(max-width: 399px) 50vw,
       (min-width: 400px) and (max-width: 900px) calc(30vw - 40px), 
       100vw"
```

The next example shows an image whose width is always relative to the viewport, until the viewport reaches a certain size. From there the image width is fixed and doesn't grow any further.

```html
<img src="cat.jpg" alt="" 
     srcset="cat-200px.jpg 200w, 
             cat-400px.jpg 400w, 
             cat-600px.jpg 600w"
      sizes="(min-width: 1400px) 340px,
             (min-width: 1200px) 33vw,
             (min-width: 800px) 50vw,
             100vw">
```

With the information about the **available images** and the **width** at which the image is displayed, the browser can now decide which image to download so it looks as sharp as possible. For this it also uses the device's **DPR** and, for example, the connection speed.

**Avatar example**

```html
<img src="avatar.jpg" alt="" 
     srcset="avatar-40px.jpg 40w, 
             avatar-60px.jpg 60w, 
             avatar-80px.jpg 80w, 
             avatar-100px.jpg 100w, 
             avatar-120px.jpg 120w, 
             avatar-160px.jpg 160w"
      sizes="(min-width: 1024px) 60px,
             40px">
```

| Viewport width | DPR | Display width | Image |
|---|--:|--:|:--|
| `300px` | 2 | `40px` | `avatar-80px.jpg` |
| `600px` | 3 | `40px` | `avatar-120px.jpg` |
| `1400px` | 1 | `60px` | `avatar-60px.jpg` |
| `1400px` | 1.5 | `60px` | `avatar-100px.jpg` |
| `1400px` | 2 | `60px` | `avatar-120px.jpg` |

**Demo** 🤯

- [Width Descriptor](https://codesandbox.io/s/0erxg)

### The `<picture>` element

The `<picture>` element gives us even more options for art direction. For example, you can show an image in a 1:1 aspect ratio on mobile and in 16:9 on desktop. It also lets us offer several file formats, so browsers can pick the best format they support, with a fallback for the rest.

### Art direction with the `<picture>` element

With the `<source>` element you can list several different images inside the `<picture>` element, each for different devices. You can use as many `<source>` elements as you like.

![Picture Sources](./assets/sources.png)

With several `<source>` elements, the browser evaluates them from *first to last*. As soon as it finds a `<source>` that meets all criteria, it uses that image.

**Example**

```html
<picture>
   <source media="(min-width:600px) and (max-width:1200px)"
           srcset="cat-medium.jpg">
   <source media="(min-width:1201px)"
           srcset="cat-large.jpg">
   <img src="cat-small.jpg" alt="">
</picture>
```

### Supporting different file formats

A `<source>` also lets us offer the same image in different file formats.  
If a browser doesn't support the given `type`, it ignores the whole source.

AVIF usually gives the smallest files, followed by WebP. Both work in every current browser: AVIF has been Baseline since January 2024, WebP since 2020. The JPEG in the `<img>` stays the fallback for everything else.

**Example**

```html
<picture>
  <source srcset="cat-avif.avif" type="image/avif">
  <source srcset="cat-webp.webp" type="image/webp">
  <source srcset="cat-jpg.jpg" type="image/jpeg">
  <img src="cat-jpg.jpg" alt="">
</picture>
```

Docs: [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types), [MDN: `<picture>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/picture)

**Useful links**

* [AVIF has landed](https://jakearchibald.com/2020/avif-has-landed/)
* [Using WebP Images](https://css-tricks.com/using-webp-images/)

### Loading and priority

The right file is only half of the work. The browser also needs to know when to load it.

- **Reserve the space.** Set `width` and `height` to the intrinsic size of the image. The browser derives the aspect ratio from them and reserves the space before the file arrives, so the page doesn't shift.
- **Lazy-load what is below the fold.** `loading="lazy"` defers the download until the image comes close to the viewport. Every current browser supports it.
- **Never lazy-load the largest image on first load.** That image is often your Largest Contentful Paint (LCP). Leave it eager and add `fetchpriority="high"`, so the browser fetches it before other images.
- **Let the browser measure lazy images.** For lazy images you can start `sizes` with `auto`: the browser uses the image's actual layout width. Chrome, Edge and Firefox support it, and Safari 27 added it with a known bug, so keep your calculated `sizes` after `auto` as the fallback.

```html
<img src="cat-400px.jpg" alt="" width="800" height="600" loading="lazy"
     srcset="cat-400px.jpg 400w, cat-800px.jpg 800w, cat-1600px.jpg 1600w"
     sizes="auto, (min-width: 800px) 50vw, 100vw">
```

Docs: [MDN: `<img>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/img), [web.dev: Browser-level image lazy loading](https://web.dev/articles/browser-level-image-lazy-loading), [web.dev: Fetch Priority](https://web.dev/articles/fetch-priority)

### Practice 🔥

Open this [**CodeSandbox**](https://codesandbox.io/s/xc76o) as your starting point.

The exercise already includes the Bootstrap grid system.

- [ ] Add a responsive image, using the image files in `./images/`
  - [ ] The image should be 12 columns wide on xs, 6 columns on rg, and 4 columns from lg upwards
  - [ ] Calculate the sizes correctly, so the size is right on every breakpoint.  
  Look at the CSS as well, to see how much padding the container, the row and the col have. Take these values into account when you calculate the size.

Time: about 15 min

## Implementation

Today we look at the specific implementation on [css.ch](https://www.css.ch).

### Modularity

In our implementation we want a generic component, so we avoid duplication and keep the quality of the image HTML consistent.  
The component should behave like an **atom** and be usable in every other module or component.

### Art direction

On the one hand, we have many different aspect ratios that can appear in the various modules.  

![Image-Ratios](./assets/design-ratios.png)

*Screenshot from the design*

On the other hand, some modules have a different aspect ratio on breakpoint x than on breakpoint y.

![Design Art Direction](./assets/design-art-direction.png)

*Screenshot from the design*

### `srcset` & `sizes`

#### Working with a grid

With a grid, images often span a number of columns.  
In this example, the second module is 8/8 columns wide on tablet, 6/8 columns on smaller desktops, and finally 8/12 columns on larger desktops.

![Example Grid](./assets/image-ratio-grid.png)

To calculate the `sizes` correctly, we have to take several parameters into account:
- The width of the container
- The maximum width of the container
- The number of columns on each breakpoint
- The gutter between the columns

With all these parameters, a size could look like this:

**Structure**

```html
sizes="(min-width: {breakpoint-width}) calc(({base} - 2 * {container-padding} + {gutter}) * ({columns}/{max-columns}) - {gutter})"
```

**Example**

```html
sizes="(min-width: 1441px) calc((1440px - 2 * 128px + 16px) * (3/12) - 16px)"
```

**Demo** 🤯

- [Proof of Concept](https://codepen.io/christiansany/pen/jQGjwd)

*The proof of concept was built for post.ch. Its sizes don't match those of css.ch exactly, but it illustrates the principle.*

#### Calculating the `sizes`

**Calculating the sizes inside a grid is not always straightforward.** That is why we built a **helper**: you pass the number of columns the image spans, and the helper returns the correct `sizes`.  

```typescript
export type BreakpointName = "large" | "medium" | "small";

export type Breakpoint = {
  readonly name: BreakpointName;
  readonly width: string;
  readonly containerPadding: string;
  readonly gutter: string;
};

const breakpoints: Breakpoint[] = [
  {
    name: "large",
    width: "1000px",
    containerPadding: "64px",
    gutter: "16px",
  },
  {
    name: "medium",
    width: "440px",
    containerPadding: "32px",
    gutter: "16px",
  },
  {
    name: "small",
    width: "0px",
    containerPadding: "16px",
    gutter: "16px",
  },
];

export const getBreakpointByName = (name: BreakpointName): Breakpoint => {
  const breakpoint = breakpoints.find((breakpoint) => breakpoint.name === name);
  if (!breakpoint) {
    throw new Error("Given BreakpointName was not found");
  }
  return breakpoint;
};

const widthCalc = (
  width: string,
  containerPadding: string,
  gutter: string,
  columns: number,
  fixGutter?: number
) => {
  let calc = `calc((${width} - 2 * ${containerPadding} + ${gutter}) * (${columns}/12) - ${gutter})`;

  if (fixGutter) {
    calc = `calc(((${width} - 2 * ${containerPadding} + ${gutter}) * (${columns}/12) - ${gutter}) - ${
      fixGutter * 2
    }px)`;
  }

  return calc;
};

export const mapImageSizesToColumns = (
  configs: { name: BreakpointName; columns: number }[]
): string => {
  return configs
    .map((config, index) => {
      const breakpoint = getBreakpointByName(config.name);
      if (breakpoint) {
        if (breakpoint.name === "large") {
          // "large" is the largest breakpoint and should not use 100vw as calc base
          return `(min-width: ${breakpoint.width}) ${widthCalc(
            "1000px",
            breakpoint.containerPadding,
            breakpoint.gutter,
            config.columns,
            undefined
          )}`;
        } else if (
          breakpoint.name === "small" ||
          configs.length === index + 1
        ) {
          // "small" is the smallest breakpoint and should not have a media query
          return widthCalc(
            "100vw",
            breakpoint.containerPadding,
            breakpoint.gutter,
            config.columns,
            undefined
          );
        }
        // Default has media-query and 100vw as base width
        return `(min-width: ${breakpoint.width}) ${widthCalc(
          "100vw",
          breakpoint.containerPadding,
          breakpoint.gutter,
          config.columns,
          undefined
        )}`;
      }
    })
    .join(",");
};
```

**Example**
  
```ts
const sizes = mapImageSizesToColumns([
  { name: "large", columns: 4 },
  { name: "medium", columns: 6 },
  { name: "small", columns: 12 },
]);

// sizes
// (min-width: 1000px) calc((1000px - 2 * 64px + 16px) * (4/12) - 16px),
// (min-width: 440px) calc((100vw - 2 * 32px + 16px) * (6/12) - 16px),
// calc((100vw - 2 * 16px + 16px) * (12/12) - 16px)
```

**Demo** 🤯

- [🌟 React Demo](https://codesandbox.io/p/sandbox/n8mczn?file=%2Fsrc%2FApp.tsx)
- [Playground Demo](https://codesandbox.io/s/o04l7)
- [Live Demo post.ch](https://www.post.ch)
- [Live Demo css.ch](https://www.css.ch)

### Types

We made life a little easier and leave the choice of file format to the backend, or rather to the CDN that [css.ch](https://www.css.ch) uses. When the browser requests a .jpg, for example, the CDN reads the `accept` header to see whether `webp`, `avif` or other formats are allowed. If they are, it returns that format in its response.

This way we don't have to deal with the different file types in the frontend, and we don't need a `<source>` for each of them.

![Accept Headers](./assets/accept-headers.png)
