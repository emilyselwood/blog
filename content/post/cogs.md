---
title: "COG - Cloud Optimized Geotiffs"
date: 2022-11-22T06:45:00+00:00
author: "Emily Selwood"
draft: false
slug: "cogs"
tags: ["geospatial", "cogs", "file formats"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

COGs, Cloud Optimized GeoTiffs, not the little things in clocks and gearboxes. 

Why do we need yet another data format? What is the point? Well dear #geospatial people. Lets dive in.

First of all, lets quickly go over what a GeoTiff is. A tiff file is a type of image. Really old format designed for fax machines, but it has an advantage of not requiring only red, green, blue or alpha bands like a lot of other image formats do so you can happily have a 12 band sentinel 2 image as a single tiff. It also handles 16bit values and floating point values. 

A geo tiff is a tiff file with a bunch of added geospatial information, so it can be placed in the world easily.

The one downside to a GeoTiff is because its a tiff and tiffs are very flexible you usually have to read the entire thing to understand it. This is fine with a fax machine, but with a 4gb multi band image its annoying. 

A COG is a GeoTiff that has been setup in a defined way. all of the metadata will be at the beginning and the bands will be laid out in a known order.

Why is it being in a known order useful? Because of a completely unrelated bit of magic tech. The HTTP range request. Normally when you use the web your browser issues lots of "get" requests, which ask for files. you usually want the entire thing. there is no point displaying half the mastodon logo usually. With a range request you can ask the web server for bytes 45 to 2367 of a file and it'll just get that. Assuming it supports range requests.

Combining this with a COG we can only pull down the bits of that 4Gb image that we actually need. Say we only want a small bit of the image to show one house on a map, or we only need two of the bands.

We can issue a couple of range requests to get the header information from the COG, and then work out where the data we want actually is and issue another range request for just that bit. Now we've not downloaded 3.9Gb of data we didn't need we've just pulled out the 100mb we do.

Cloud storage systems like Amazon S3 or azure Blob storage support these range requests already, and their ease of use with COGs is why they are called "Cloud Optimized GeoTiffs" 

They are designed to work on the cloud, but they don't have to be on the cloud. They work just like an existing GeoTiff, so you can use them anywhere. 

That is the magic, they don't break anything that already works with a GeoTiff input, but they let you do new fun things.

If you want to know more about tiff file internals I gave a talk at [Foss4guk](https://www.youtube.com/watch?v=Z5g5p4H5u58) a couple of years go on the the topic