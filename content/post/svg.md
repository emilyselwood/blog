---
title: "SVG"
date: 2022-11-23T06:45:00+00:00
author: "Emily Selwood"
draft: false
slug: "svgs"
tags: ["file formats"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

lets talk about SVGs as an image format. 

You may have heard of them called a vector format. This means that instead of defining a grid of pixels and having about what colour every pixel is, it defines the features of the image. 

There is a red line from 12,45 to 26,78
The text "hello" is in front ariel, 14pt and blue
etc.

The advantage of this is you can scale the image with out losing any quality. Hence the name Scalable Vector Graphics

If you double the size of the image you still have a red line, the edges of it will still be just as crisp, unlike if you double the size of a raster image, you end up with 4 pixels for every one in the original, it starts to look blocky unless you do some magic maths to add extra information where there wasn't before. AKA, make stuff up.

Internally an SVG file is an xml document. Just like any webpage you view its a series of "tags" which tell your computer what to display. There are a lot of similar concepts too, both have text, links, ids, even CSS.

SVGs have lines, rectangles, circles. You can draw almost anything as an SVG, but it can be a little more complex than painting a raster image.

![A screen shot of the internal xml structure of a SVG file.](/img/shorts/svg_internals.png)

Instead of div tags to group bits of an html document you have g or group tags. This is almost always just an ease of use thing rather than actually meaning anything in the picture. You can apply transformations and styles at the group level though to make your life easier. "rotate all of these things by 60 degrees" or "make all the lines red and fill with green"

Where the SVG language gets away from xml is in the definitions of paths and lines. Paths are a series of points joined by lines or curves. Once you start to deal with curves in computers things start to get complex. Mathematically there are a bunch of different ways to define curves. Some based on circles, some based on Bezier curves, which a bunch of helpful shortcuts.

If you want to know more [Mozilla's documentation](https://developer.mozilla.org/en-US/docs/Web/SVG/Tutorial/Paths) on SVG's is excellent.

Like HTML I'd argue that anything more complex than a simple image, you should not be hand crafting it. Inkscape is an excellent and free opensource tool to edit SVG files. Because they are basically an XML document internally there are libraries available for almost every programming language that exists. Even if you just end up editing the raw XML

One thing to keep in mind when working with SVG images is eventually it will be pixels on a screen or on a page. When you get there you will need to define your Pixels per inch. There is no defined standard, Inkscape uses 96ppi by default. The Cricut used 72. This causes no end of headaches. You can use physical units inside an SVG if you want. You can also set the size of the image, e.g. to the size of an A4 page.

While I said they will be pixels on a screen, the other really useful thing about SVGs is because they are made up of paths mostly they can be fed into things like laser cutters, plotters, and draw knife machines like the Cricut. The thing to watch out for though is the level of detail. You usually have a minimum feature size so you want to make sure your detail isn't below that.

