# Lazy load images and `<iframe>` elements

Images and `<iframe>` elements often consume more bandwidth than other types of
resources. In the case of `<iframe>` elements, a fair amount of extra processing
time can be involved in loading and rendering the pages within them.

In the case of lazy loading images, deferring the loading of images that are
outside of the initial viewport can be helpful in reducing bandwidth contention
for more critical resources within the initial viewport. This can improve a
page's Largest Contentful Paint (LCP) in some cases where network connections
are poor, and that reallocated bandwidth can help LCP candidates load and
paint faster.

Where `<iframe>` elements are concerned, a page's Interaction to Next Paint
(INP) can be improved during startup by lazy loading them. This is because an
`<iframe>` is a completely separate HTML document with its own subresources.
While `<iframe>` elements can be run in a separate process, it's not uncommon
for them to share a process with other threads, which can create conditions
where pages become less responsive to user input.

Thus, deferring the loading of off-screen images and `<iframe>` elements is a
technique worth pursuing, and requires fairly low effort for a reasonably good
return in terms of performance. This module covers explains to lazy load these
two types of elements for a faster and better user experience during the page's
critical startup period.

## Lazy load images with the loading attribute

The loading attribute can be added to `<img>` elements to tell browsers how
they should be loaded:


- "eager" informs the browser that the image should be loaded immediately,
even if it's outside the initial viewport. This is also the default value for
the loading attribute.
- "lazy" defers loading an image until it's within a set distance from the
visible viewport. This distance varies by browser, but is often set to be
large enough that the image loads by the time the user scrolls to it.



> [!IMPORTANT]
> As stated previously, the distance from the viewport at which the browser decides that the image is needed when using the loading="lazy" attribute varies by browser. Factors involved may be the effective connection type, as well as the type of image.


It's also worth noting that if you're using the `<picture>` element, the
loading attribute should still be applied to its child `<img>` element, not
the `<picture>` element itself. This is because the `<picture>` element is a
container that contains additional `<source>` elements pointing to different
image candidates, and the candidate that the browser chooses is applied directly
to its child `<img>` element.

### Don't lazy load images that are in the initial viewport

You should only add loading="lazy" attribute to `<img>` elements that are
positioned outside the initial viewport. However, it can be complex to know the
precise position of an element relative within the viewport before the page is
rendered. Different viewport sizes, aspect ratios, and devices have to be
considered.

For example, a desktop viewport can be quite different from a viewport on a
mobile phone as it renders more vertical space which may be able to fit images
in the initial viewport that wouldn't appear in the initial viewport of a
physically smaller device. Tablets used in portrait orientation also display
a considerable amount of vertical space, perhaps even more than some desktop
devices.

However, there are some cases in which it's fairly clear that you should avoid
applying loading="lazy". For example, you should definitely omit the
loading="lazy" attribute from `<img>` elements in cases of hero images,
or other image use cases where `<img>` elements are likely to appear above the
fold, or near the top of the layout on any device. This is even more important
for images that are likely to be LCP candidates.

Images that are lazy loaded need to wait for the browser to finish layout in
order to know if the image's final position is within the viewport. This means
that if an `<img>` element in the visible viewport has a loading="lazy"
attribute, it's only requested after all CSS is downloaded, parsed, and
applied to the page—as opposed to being fetched as soon as it is discovered by
the preload scanner in the raw markup.

Since the loading attribute on the `<img>` element is supported on all major
browsers there is no need to use JavaScript to lazy load images, as adding
extra JavaScript to a page to provide capabilities the browser already provides
affects other aspects of page performance, such as INP.


> [!NOTE]
> The loading attribute does not affect the image's network priority. To adjust the network priority, you can use the Fetch Priority API. Be aware that an image in the visible viewport with fetchpriority="high" and loading="lazy" still waits until all CSS is downloaded and parsed.


### Image lazy loading demo

## Lazy load `<iframe>` elements

Lazy loading `<iframe>` elements until they are visible in the viewport can save
significant data and improve the loading of critical resources that are required
for the top-level page to load. Additionally, because `<iframe>` elements are
essentially entire HTML documents loaded within a top-level document, they can
include a significant number of subresources—particularly JavaScript—which can
affect a page's INP considerably if the tasks within those frames requires
significant processing time.

Third-party embeds are a common use case for `<iframe>`elements. For example,
embedded video players or social media posts commonly use `<iframe>`elements,
and they often require a significant number of subresources which can also
result in bandwidth contention for the top-level page's resources. As an
example, lazy loading a YouTube video's embed saves more than 500 KiB during
the initial page load, while lazy loading the Facebook Like button plugin
saves more than 200 KiB—most of which is JavaScript.

Either way, whenever you have an `<iframe>`below the fold on a page, you should
strongly consider lazy loading it if it's not critical to load it up front, as
doing so can significantly improve the user experience.

### The loading attribute for `<iframe>`elements

The loading attribute on `<iframe>`elements is also supported in all major
browsers. The values for the loading attribute and their behaviors are the
same as with `<img>`elements that use the loading attribute:


- "eager" is the default value. It informs the browser to load the `<iframe>`
element's HTML and its subresources immediately.
- "lazy" defers loading the `<iframe>`element's HTML and its subresources
until it is within a predefined distance from the viewport.



> [!NOTE]
> Chrome reserves space and shows a placeholder when a lazy loaded `<iframe>` is still being fetched in an effort to avoid layout shifts. However, you should still consider using the `<iframe>` element's width and height attributes as well as additional styling in CSS to minimize layout shifts.


### Lazy loading iframes demo

## Facades

Instead of loading an embed immediately during page load, you can load it on
demand in response to a user interaction. This can be done by showing an image
or another appropriate HTML element until the user interacts with it. Once the
user interacts with the element, you can replace it with the third-party embed.
This technique is known as a facade.

A common use case for facades is video embeds from third-party services where
the embed may involve loading many additional and potentially expensive
subresources—such as JavaScript—in addition to the video content itself. In such
a case—unless there's a legitimate need for a video to autoplay—video embeds
require the user to interact with them before playback by clicking the play
button.

This is a prime opportunity to show a static image that is visually similar to
the video embed and save significant bandwidth in the process. Once the user
clicks on the image, it's then replaced by the actual `<iframe>`embed, which
triggers the third-party `<iframe>`element's HTML and its subresources to begin
downloading.

In addition to improving initial page load, another key upside is that if the
user never plays the video, the resources required to deliver it are never
downloaded. This is a good pattern, as it ensures the user only downloads what
they actually want to, without making possibly faulty assumptions about the
user's needs.

Chat widgets are another excellent use case for the facade technique. Most
chat widgets download significant amounts of JavaScript that can negatively
affect page load and responsiveness to user input. As with loading anything up
front, the cost is incurred at load time, but in the case of a chat widget, not
every user never intends to interact with it.

With a facade on the other hand, it's possible to replace the third-party "Start
Chat" button with a fake button. Once the user meaningfully interacts with
it—such as holding a pointer over it for a reasonable period of time, or
with a click—the actual, functional chat widget is slotted into place when the
user needs it.

While it's certainly possible to build your own facades, there are open source
options available for more popular third parties, such as lite-youtube-embed
for YouTube videos, lite-vimeo-embed for Vimeo videos, and React Live Chat
Loader for chat widgets.

## JavaScript lazy loading libraries

If you need to lazy load `<video>`elements, `<video>`element poster images,
images loaded by the CSS background-image property, or other unsupported
elements, you can do so with a JavaScript-based lazy loading solution, such as
lazysizes or yall.js, as lazy loading these types of resources is not a
browser-level feature.

In particular, autoplaying and looping `<video>`elements without an audio track
are a much more efficient alternative than using animated GIFs, which can
often be several times larger than a video resource of equivalent visual
quality. Even so, these videos can still be significant in terms of bandwidth,
so lazy loading them is an additional optimization that can go a long way to
reducing wasted bandwidth.

Most of these libraries work using the Intersection Observer API—and
additionally the Mutation Observer API if a page's HTML changes after the
initial load—to recognize when an element enters the user's viewport. If the
image is visible—or approaching the viewport—then the JavaScript library
replaces the non-standard attribute, (often data-src or a similar attribute),
with the correct attribute, such as src.

Say you have a video that replaces an animated GIF, but you want to lazy load it
with a JavaScript solution. This is possible with yall.js with the following
markup pattern:

```html
<!-- The autoplay, loop, muted, and playsinline attributes are to
     ensure the video can autoplay without user intervention. -->
<video class="lazy" autoplay loop muted playsinline width="320" height="480">
  <source data-src="video.webm" type="video/webm">
  <source data-src="video.mp4" type="video/mp4">
</video>
```

By default, yall.js observes all qualifying HTML elements with a class of
"lazy". Once yall.js is loaded and executed on the page, the video doesn't
load until the user scrolls it into the viewport. At that point, the data-src
attributes on the `<video>`element's child `<source>`elements are swapped out
to src attributes, which sends a request to download the video and
automatically begin playing it.

### Test your knowledge

- Which is the default value for the loading attribute for
both `<img>`and `<iframe>`elements?  
  ◯ "lazy" 
  ◯ "eager"

- When are JavaScript-based lazy loading solutions reasonable to use?  
  ◯ For resources in which the loading attribute isn't
supported, such as in the case of autoplaying videos intended to replace
animated images, or to lazy load a `<video>`element's
poster image.  
  ◯ For any resource that can be lazy loaded.

- When is a facade a useful technique?  
  ◯ For any third-party embed where the resources required to load are not only substantial, but there is a decent probability that not all users may interact with them.  
  ◯ For any third-party embed that consumes significant data, regardless of the user's needs.



## Up next: Prefetching and prerendering

Now that you have a handle on lazy loading images and `<iframe>`elements,
you're in a good position to ensure that pages can load more quickly while
respecting the needs of your users. However, there are cases in which
speculative loading of resources can be desirable. In the next module,
learn about prefetching and prerendering, and how these techniques—when used
carefully—can substantially speed up navigations to subsequent pages by loading
them ahead of time.
