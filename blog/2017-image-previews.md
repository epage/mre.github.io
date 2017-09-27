extends: default.liquid

title:      Lightning fast image previews with pure CSS and LQIP
date:       18 Sep 2017 00:00:00 +0000
humandate:  18th of September 2017
path:       2017/image-previews
social_img: 2017_image_previews.png
---

<figure>
            <div class="loader">
            <object data="/img/posts/2017/image-previews/factory.svg" type="image/svg+xml"></object>
            <img class="frozen" src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA8AAAAJCAMAAADaUYZlAAAA81BMVEX////84ub84eX84OT97vH++PnPsLns8PPl6+7t8fP//v396sO/nK7voq+jb4Devaafr7marbejjJraWnLyzsfv4832r7rjpJLBiXnHrqOltb2gsbulsLHVpqvcPVn808T/69bdusLRtKiipKa0ravz9/n9/v7O1tCqn6bEZnz22cj/7NXr0NXtyLjrwrLwy7vrvKjfuZtuk6eOqrl9na6shpvqeYr98fP72t/ik6Tloa3wvK/2zb/pqKGqd3eyfn94kKKxgJftYHb96+77+/vS1dvoxczoxc3u1cnlvLPbopvTn5rbopzj19bZ0tju3uD7+vrqrSoEAAAANklEQVR42mNhQAVAPiMCfAXxuYGMjwIIPheQwQ2Rfw2RhwMGNL75SSCfB8lAoLwLIyp/D4r9APY4BoRjYiX7AAAAAElFTkSuQmCC" />
        </div>
  <figcaption>
  Adapted from <a href="http://www.freepik.com/free-vector/industrial-machine-vector_753558.htm">Freepik</a>
  </figcaption>
</figure>

My website is reasonably fast.  

I hope that every page load feels snappy, no matter your device or location.
That should not come as a surprise. After all, I'm just using plain HTML and CSS.
JavaScript is avoided whenever possible.

There was one thing left which really annoyed me: layout reflow after images got loaded.

The problem is, that the image dimensions are not known when the text is ready to be displayed.
As a result, the text will be pushed down on the screen as soon as an image is loaded above.

Also, while an image is loading, there is no preview, just blank space.
Here's what that looks like on a slower connection:

![Illustration of a flash of unstyled content](/img/posts/2017/image-previews/fout.png)

I could fix that, by hardcoding the image width and height, but that would be tedious and error-prone.
And there would be no preview.

### Tiny image thumbnails

So I was wondering, what others were doing.
I vaguely remembered, that [Facebook uses tiny preview thumbnails in their mobile app](https://code.facebook.com/posts/991252547593574/the-technology-behind-preview-photos/).
They extract the quantization table from the JPEG header to render the preview. This information 
is stored on the client, so it doesn't need to be downloaded every time.
Unfortunately, this approach requires full control over the image encoder.
It works for apps, but hardly for websites.

The search continued.

Until my colleague [Tobias Baldauf](http://tobias.is/) introduced me to [LQIP (Low-Quality Image Placeholders)](http://www.guypo.com/introducing-lqip-low-quality-image-placeholders/).

Here's the idea:

* Load the page including inlined, low-quality image thumbnails.
* Once the page is fully loaded (e.g. when the [`onload` event](https://www.w3schools.com/jsref/event_onload.asp) is fired), lazy load full quality images.

Unfortunately, this technique requires JavaScript.
Nevertheless, I liked the idea, so I started experimenting with different image sizes and formats. My goal was to create the smallest thumbnails using any common image format.

### Benchmark

Here are 15 pixel wide thumbnails encoded in different file formats:

![Comparison of different image formats when creating thumbnails](/img/posts/2017/image-previews/thumbnails.jpg)

I used different tools to create the thumbnails.
For JPEG and PNG encoding, I used [svgexport](https://github.com/shakiba/svgexport).

```bash
svgexport img.svg img.png "svg{background:white;}" 15: 1%
```

For webp, I used [cwebp](https://developers.google.com/speed/webp/docs/cwebp):

```bash
cwebp img.png -o img.webp
```

The gif was converted using an online tool and optimized using [gifsicle](https://github.com/kohler/gifsicle):

```bash
gifsicle -O3 < img.gif > img_mini.gif
```

### Comparison

WebP is the smallest, but it's [not supported by all browsers](http://caniuse.com/#feat=webp).  
Gif was second, but when resizing the image and applying the blur filter, I was not happy with the result.  
In the end I, settled for PNG, which provided a nice tradeoff between size and quality.
I optimized the images even further using [oxipng](https://github.com/shssoichiro/oxipng).
With that, I kept my budget of around 200-250 bytes per thumbnail.

I integrated the thumbnail creation into my build toolchain for the blog.
The actual code to create the images is rather boring.
If you *really* want to have a look, [I've pushed it to Github](https://github.com/mre/lqip/).

### Avoiding JavaScript


Here is the skeleton HTML for the image previews:

```html
<figure>
  <div class="loader">
    <object data="image.svg" type="image/svg+xml"></object>
    <img class="frozen" src="data:image/png;base64,..." />
  </div>
</figure>
```

The trick is to wrap both the full-size image and the preview image into a `loader` div,
which gets a `width: auto` CSS attribute:

```css
.loader {
  position:relative;
  overflow: hidden;
  width: auto;
}
```

I wrap the SVG into an `object` tag instead of using an `img` element.
This has the benefit, that I can show a placeholder in case the SVG can't be loaded.
I position the `object` at the top left of the `loader` div.

```css
.loader object {
  position: absolute;
}

.loader img, .loader object {
  display: block;
  top: 0;
  left: 0;
  width: 100%;
}
```

Here's the placeholder *hack* including some references:

```css
// https://stackoverflow.com/a/29111371/270334
// https://stackoverflow.com/a/32928240/270334
object {
  position: relative;
  float: left;
  display: block;
  
  &::after {
    position: absolute;
    top: 0;
    left: 0;
    display: block;
    width: 1000px;
    height: 1000px;
    content: '';
    background: #efefef;
  }
}
```

The last part is the handling of the thumbnails.
Like most other sites, I decided to apply a blur filter.
In a way, it looks like the image is *frozen*, so that's what I called the CSS selector.
I also applied a scaling transformation to achieve sharp borders.

```
.frozen {
  -webkit-filter: blur(8px);
  -moz-filter: blur(8px);
  -o-filter: blur(8px);
  -ms-filter: blur(8px);
  filter: blur(8px);
  transform: scale(1.04);
  animation: 0.2s ease-in 0.4s 1 forwards fade;
  width: 100%;
}

@keyframes fade {
  0% {
    opacity:1;
  }
  100% {
    opacity:0;
  }
}
```

I use CSS animations instead of JavaScript.  
The duration of the animation is based on the 95% percentile load time of all visitors of the page. Although it's just an approximation, this should work for most readers.

### Result

* No JavaScript needed
* Supports both SVG and JPEG
* Standards-compliant and works on all modern browsers
* Provides a fallback in case the main image cannot be loaded
* Tiny overhead

### Resources

* [Introducing LQIP – Low Quality Image Placeholders](http://www.guypo.com/introducing-lqip-low-quality-image-placeholders/)
* [How Medium does progressive image loading](https://jmperezperez.com/medium-image-progressive-loading-placeholder/)
* [SQIP, a new preview technique using pure SVG](https://github.com/technopagan/sqip)