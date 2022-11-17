---
title: "InSAR"
date: 2022-11-17T08:17:00+01:00
author: "Emily Selwood"
draft: false
slug: "insar"
tags: ["geospatial"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

Today I'm going to do a thread on something I want to learn more about but have used a little in the past. So be warned this topic might not be 100% accurate but is my understanding.

Interferometric synthetic-aperture radar or InSAR for short because everyone loves acronyms.

Its a way of measuring how much something has moved between two radar "images" the something can be a building, a dam, a mountain. If you've ever seen those "fringe" maps with bands after an earthquake that's this.

![fringe map](/img/shorts/fringe.png)

To do this you need two things:
1: a radar satellite that has a very well known orbit and orbits consistently over the area of interest, you always need to be "looking" at the target from the same direction.
2: A time series of radar images. You need two images to compare, you cant do it with one.

My understanding of the way it works is by measuring the phase of the returned signal and comparing it between the two images. Movement changes the distance so the returned wave isn't the same.

Should probably do a thread on SAR at some point but that's for another day.

With a high resolution radar satellite you can measure individual parts of a structure and see if any part is moving compared to the rest. 

The amazing thing about this technology is you can measure movements of fractions of mm. (this is the part that makes this seem magic to me)

So I've mentioned measuring buildings a few times, but how is this use full. Say you own a bridge, over time that bridge needs maintenance, but its expensive to send people out regularly to look at your bridge and tell you if you need to do anything.  Also potentially dangerous.  

For my day job we worked on this with the Canadians. Where currently they get someone to walk across the bridge with a gps tool to measure where the bridge is. At best they get this result every 6 months.

So the system we build is able to look at radar images of the same bridge every couple of weeks (the orbits of the satellite mean it takes a couple of weeks or so to be back in the same place) 

It also gets way way more data due to the resolution of the satellite we were using. The screen shot shows the radar result. The colour's show how much each point on the bridge has moved.

But don't worry its meant to move like that.

![A screen shot of project brigital](/img/shorts/brigital.png)

Its a big metal structure. Metals change size as the temperature changes. On something the size of a bridge we have to take that into account. Meaning you need the local weather as well or you are going to unnecessarily panic. 

The maths involved between how you go from a pair of radar images to collection of points like this is where my knowledge ends. I helped load the results into the application and render them on the screen.

So in conclusion InSAR is awesome and something you should keep in mind for your remote monitoring needs. 

If you have any good articles on how the maths inside it works I'd love to see them, please send me a link on mastodon.