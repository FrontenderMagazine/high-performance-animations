# High Performance Animations

We’re going to cut straight to the chase. Modern browsers can animate four things really cheaply: **position**, **scale**, **rotation** and **opacity**. If you animate anything else, it’s *at your own risk*, and the chances are you’re not going to hit a silky smooth 60fps.

![figure][Cheap to animate]

Take a look at this side-by-side slow motion video of the same animation:

<iframe style="position:absolute;top:0;left:0;width:100%;height:100%;" src="//www.youtube.com/embed/-62uPWUxgcg?rel=0" allowfullscreen="" frameborder="0"></iframe>

One is done with transforms, the other isn’t. You can see what a difference it makes, so let’s take look at why that is.

## From DOM to Pixels in DevTools

When you use the Chrome DevTools timeline you should see a pattern similar to this:

![screenshot][DevTools waterfall] 

*Chrome DevTools frame mode. The higher up the waterfall you start, the more work the browser does.*

The process that the browser goes through is pretty simple: calculate the styles that apply to the elements (Recalculate Style), generate the geometry and position for each element (Layout), fill out the pixels for each element into [layers][1] (Paint Setup and Paint) and draw the layers out to screen (Composite Layers).

To achieve silky smooth animations you need to avoid work, and the best way to do that is to only change properties that affect compositing -- transform and opacity. **The higher up you start on the timeline waterfall the more work the browser has to do to get pixels on to the screen.**

> This advice is broadly cross-browser compatible. Chrome, Firefox, Safari and Opera all hardware accelerate transforms and opacity. Unfortunately it is not clear what criteria Internet Explorer 10+ uses to determine if hardware acceleration is suitable, but hopefully that will become clearer when the F12 tools in IE11 are shipped.

## Animating Layout Properties

When you change elements, the browser may need to do a layout, which involves calculating the geometry (position and size) of all the elements affected by the change. If you change one element, the geometry of other elements may need to be recalculated. For example, if you change the width of the `<html>` element any of its children may be affected. Due to the way elements overflow and affect one another, changes further down the tree can sometimes result in layout calculations all the way back up to the top.

The larger the tree of visible elements, the longer it takes to perform layout calculations, so you must take pains to avoid animating properties that trigger layout.

> Are you storing app state in the or elements with classes? When these elements are changed, the browser may have to redo style calculations and layout. Watch out for inadvertent layout triggers in your apps; they may not animate but they can be very expensive!

Here are the [most popular CSS properties][2] that, when changed, trigger layout:

**Styles that affect layout**

<table>
<tr><td>width</td><td>height</td></tr>
<tr><td>padding</td><td>margin</td></tr>
<tr><td>display</td><td>border-width</td></tr>
<tr><td>border</td><td>top</td></tr>
<tr><td>position</td><td>font-size</td></tr>
<tr><td>float</td><td>text-align</td></tr>
<tr><td>overflow-y</td><td>font-weight</td></tr>
<tr><td>overflow</td><td>left</td></tr>
<tr><td>font-family</td><td>line-height</td></tr>
<tr><td>vertical-align</td><td>right</td></tr>
<tr><td>clear</td><td>white-space</td></tr>
<tr><td>bottom</td><td>min-height</td></tr>
</table>

Source: [http://goo.gl/lPVJY6][3]

## Animating Paint Properties

Changing an element may also trigger painting, and the majority of painting in modern browsers is done in software rasterizers. Depending on how the elements in your app are grouped into layers, other elements besides the one that changed may also need to be painted.

> If you’re new to the concept of layers you should read Tom Wiltzius’s [introduction to the topic][4].

There are many properties that will trigger a paint, but here are the most popular:

**Styles that affect paint**

<table>
<tr><td>color</td><td>border-style</td></tr>
<tr><td>visibility</td><td>background</td></tr>
<tr><td>text-decoration</td><td>background-image</td></tr>
<tr><td>background-position</td><td>background-repeat</td></tr>
<tr><td>outline-color</td><td>outline</td></tr>
<tr><td>outline-style</td><td>border-radius</td></tr>
<tr><td>outline-width</td><td>box-shadow</td></tr>
<tr><td>background-size</td><td></td></tr>
</table>

Source: [http://goo.gl/lPVJY6][5]

If you animate any of the above properties the element(s) affected are repainted, and the layers they belong to are uploaded to the GPU. On mobile devices this is particularly expensive because CPUs are significantly less powerful than their desktop counterparts, meaning that the painting work takes longer; and the bandwidth between the CPU and GPU is limited, so texture uploads take a long time.

## Animating Composite Properties

There is one CSS property, however, that you might expect to cause paints that sometimes does not: opacity. Changes to opacity can be handled by the GPU during compositing by simply painting the element texture with a lower alpha value. For that to work, however, the element must be the **only one in the layer**. If it has been grouped with other elements then changing the opacity at the GPU would (incorrectly) fade them too.

In Blink and WebKit browsers a new layer is created for any element which has a CSS transition or animation on opacity, but many developers use `translateZ(0)` or `translate3d(0,0,0)` to manually force layer creation. Forcing layers to be created ensures both that the layer is painted and ready-to-go as soon as the animation starts (creating and painting a layer is a non-trivial operation and can delay the start of your animation), and that there's no sudden change in appearance due to antialiasing changes. Promoting layers should done sparingly, though; you can overdo it and having [too many layers can cause jank][6].

> In Chrome, non-root, non-opaque layers use [grayscale antialiasing rather than subpixel antialiasing][7], which can be very noticeable, especially if the antialiasing method changes suddenly. If you are going to promote an element don’t wait until the animation starts, do it ahead of time.

Changing the transform of an element boils down to changes to its position, rotation or scale. Often, position is animated by setting the `left` and `top` properties. The problem is, as shown above, `left` and `top` both trigger layout operations, and that's expensive. The better solution is to use a `translate` on the element, which does not trigger layout.

> In Chrome Canary and Safari you can also animate filters, as these are handled off the main thread, are accelerated and generally perform very well. But since filters are not yet supported in Internet Explorer or Firefox you should proceed with caution.

## Imperative vs Declarative Animations

Developers often have to decide if they will animate with JavaScript (imperative) or CSS (declarative). There are pros and cons to each, so let’s take a look:

### Imperative

The main pro of imperative animations happens to also be its main con: it’s running in JavaScript on the browser’s main thread. The main thread is already busy with other JavaScript, style calculations, layout and painting. Often there is thread contention. This substantially increases the chance of missing animation frames, which is the very last thing you want.

Animating in JavaScript does give you a lot of control: starting, pausing, reversing, interrupting and cancelling are trivial. Some effects, like [parallax][8] scrolling, can only be achieved in JavaScript.

### Declarative

The alternative approach is to write your transitions and animations in CSS. The primary advantage is that the browser can optimize the animation. It can create layers if necessary, and run some operations off the main thread which, as you have seen, is a good thing. The major con of CSS animations for many is that they lack the expressive power of JavaScript animations. It is very difficult to combine animations in a meaningful way, which means authoring animations gets complex and error-prone.

## Looking to the future

As web standards evolve, some of the limitations around animation will go away. There is a proposal by Google’s Ian Vollick that investigates the concept of allowing [JavaScript animations via workers][9], providing the animation does not trigger layout or style recalculations.

For those interested in a more declarative approach to animation there is the [Web Animations specification][10], which [Jake Archibald has written about extensively][11].

## Conclusion

Animating well is core to a great web experience. You should always look to avoid animating properties that will trigger layout or paints, both of which are expensive and may lead to skipped frames. Declarative animations are preferable to imperative since the browser has the opportunity to optimize ahead of time.

Today transforms are the best properties to animate because the GPU can assist with the heavy lifting, so where you can limit your animations to these, do so.

* opacity
* translate
* rotate
* scale

In the future there may be new ways of animating that allow you to be as expressive as you can be with JavaScript, but without the associated main thread cost; or as optimized as CSS animations but without the restrictions, but until then plan your animations for a silky smooth experience.

## Resources

* [The current status of animated properties on the web][12].
* [Paul Irish’s deck on performance tooling][13].
* [Antialiasing 101][14]
* [We’re Gonna Need A Bigger API!][15]

[1]: http://www.html5rocks.com/en/tutorials/speed/layers/
[2]: https://docs.google.com/spreadsheet/pub?key=0ArK1Uipy0SbDdHVLc1ozTFlja1dhb25QNGhJMXN5MXc&single=true&gid=0&output=html
[3]: https://docs.google.com/spreadsheet/pub?key=0ArK1Uipy0SbDdHVLc1ozTFlja1dhb25QNGhJMXN5MXc&single=true&gid=0&output=html
[4]: http://www.html5rocks.com/en/tutorials/speed/layers/
[5]: https://docs.google.com/spreadsheet/pub?key=0ArK1Uipy0SbDdHVLc1ozTFlja1dhb25QNGhJMXN5MXc&single=true&gid=0&output=html
[6]: http://wesleyhales.com/blog/2013/10/26/Jank-Busting-Apples-Home-Page/
[7]: http://www.html5rocks.com/en/tutorials/internals/antialiasing-101/
[8]: http://www.html5rocks.com/en/tutorials/speed/parallax/
[9]: https://github.com/ianvollick/animation-proxy/blob/master/Explainer.md
[10]: http://dev.w3.org/fxtf/web-animations/
[11]: http://www.smashingmagazine.com/2013/03/04/animating-web-gonna-need-bigger-api/
[12]: http://www.chromestatus.com/metrics/css/animated
[13]: https://docs.google.com/presentation/d/19R_E5B__kdE55L1bTpS6IFKbYbHq-PQKKky4do5Yc6A/edit#slide=id.g105c64d69_170
[14]: http://www.html5rocks.com/en/tutorials/internals/antialiasing-101/
[15]: http://www.smashingmagazine.com/2013/03/04/animating-web-gonna-need-bigger-api/

[Cheap to animate]: img/cheap-operations.jpg
[DevTools waterfall]: img/devtools-waterfall.jpg