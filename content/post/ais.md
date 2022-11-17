---
title: "AIS"
date: 2022-11-16T06:27:00+01:00
author: "Emily Selwood"
draft: false
slug: "ais"
tags: ["geospatial"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

Today we'll talk about ais data. This is another post that started out as a thread over on mastodon

Sorry it's another acronym. Automatic Identification System. It's original purpose was for ships to be able to tell each other where they are and which way they are going to avoid collisions. It is a little radio box on the boat that sends periodic messages saying "my Id is 123456, I'm at this lat and long, I'm going this way at this speed" it also sends out other messages like "my Id is 123456 my name is see the wind, I'm a bulk grain transporter and 30m long"

There are many message types that can mean all sorts of different things. There are special ones for lighthouses, search and rescue aircraft, etc.

By maritime law any vessel over a defined size must have an aid transmitter on board. The fun thing is that these transmissions can be picked up by anyone. The transmissions are mostly line of sight (it's radio, that gets weird so I'm not going to go into it too much) but satellites are always over head so can see everything.

It's possible to buy a feed of the transmissions of every boat on the planet, it's quite a lot of data to process, but honestly only just into big data territory. 

The problem is the data is noisy. Very noisy.   Because the transmitters are on the vessels, the owner can do what they like, turn it off? Easy, unplug the power. Change the id and vessel details? Also easy. If someone is doing something they shouldn't they will often just sail out of port and turn off the transmitter.

Then there is the fun that because the messages are resent pretty often it doesn't matter if one or two are corrupt so there is very little error correction. Bit flips are possible and more than one is common, if that bit is in the position parts it can make a vessel look like it's jumped across the globe for a second and then jumped back. That's before hundreds of people put their Id in as 111111 or 123456

So let's dig in to some use cases: 

Say your country has a system to license vessels to fish in your waters, but also can't really afford a big navy. With ais data you can track vessels in your waters and deal with any that look like they are fishing.

Boats that are fishing move very differently to a cargo ship trying to get to the next port, so even if it lies about being a fishing vessel you can tell.

What about verification that the fish you are selling in your super market is caught legally? This is a good one because it's just making sure vessels are doing what they say they are, so they will want to keep their transmitters on to prove they are good and can sell their fish.

You can also keep track of your cargo containers and know where the shipments are out side of what the shipping company says. That snafu in the Suez was fun to watch.

Want to make an economic estimate of how much stuff is getting to a country? Reasonably easy to get a list of vessels who've arrived at ports and how big they are. From a humanitarian point of view I've helped figure out how much fuel was getting into Yemen.

Ais is an awesomely powerful dataset but also not easy to work with. The source data comes in as basically points and then you have to construct routes and so on from that.

If you want to have a look at some data check out [marine traffic](https://www.marinetraffic.com/en/ais/home/centerx:-12.0/centery:25.0/zoom:4). Who have a nice web interface. Also check out [Anita Graser's tutorial on processing ais data](https://github.com/anitagraser/EDA-protocol-movement-data)