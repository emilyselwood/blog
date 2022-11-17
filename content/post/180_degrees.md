---
title: "180 Degrees"
date: 2022-11-14T19:48:00+01:00
author: "Emily Selwood"
draft: false
slug: "180_degrees"
tags: ["geospatial"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

Note: This post started out as a thread on mastodon. Please follow me there if you'd like to see this sort of thing early

Lets start talking about something I wish I didn't know as much about as I do.

The 180 degree line! +/-180 degrees longitude. 

![180 degrees](/img/shorts/180_degrees.png)

It is not the same thing as the international date line, that's defined politically which lots of wiggles through the pacific islands.

The 180 degree line only crosses two countries, Russia and Fiji.

This means that you can go from +180 degrees to -180 degrees with a single step.

However most software that deals with maps doesn't understand this. "Not many people live there, it doesn't matter" people say. it is not unusual to end up with polygons going the wrong way around the world, when you ask for Fiji's borders. 

The other quite common solutions is to split the polygon in two, so there is a tiny sliver down the middle of Fiji that doesn't apparently belong to Fiji, at least according to the map.

The way the Fijians solve this is they have their own National Coordinate system that just covers the islands. 

The problem of course is converting into and out of this coordinate system, which you almost invariably need to do to plot things on a web map, as they commonly use a coordinate system based on latitude and longitude called WGS 84

For real fun you need to get a satellite image that crosses over the 180 degree line. The most common software for processing Sentinel 1 radar scenes attempts to allocate an array of 10m pixels all the way around the entire globe. This will almost always fail, due to running out of memory.

oh and if you think this sounds bad you should try working at either of the poles. Tip of the hat to my collogues who work with the Antarctic surveys