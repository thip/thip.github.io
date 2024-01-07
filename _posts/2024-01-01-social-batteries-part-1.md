---
layout: post
title:  "Manufacturing and Selling Electronic Badges Part 1: Designing and Prototyping the board"
image: /assets/SocialBattery-1.jpg
description: ""
permalink: /posts/social-batteries-part-1
tags:
---

>I wrote this series of posts primarily for myself, or at least someone who is trying to make something electronic to sell. If that's you then I hope this helps you on your journey!  If you just want to see the end product then you can find it at [https://hortus.dev/products/social-battery](). It's a long post so I have broken it up into chunks to make it a bit easier to digest. These are:
>
>1. [Designing and Prototyping the Board](/posts/social-batteries-part-1) (this article)
>2. [Selling the Prototypes Online](/posts/social-batteries-part-2)
>3. [Going from Prototype to Production](/posts/social-batteries-part-3)

![3D Render of the design in KiCad](/assets/SocialBattery-1.jpg)

## Introdution
I wanted to experiment with JLC PCB's assembly service - Whilst I've designed and manufactured bare PCBs before before placing and soldering components manually, I've got some future projects in mind that will be impractical to solder up by hand due to both the quantity and size of components. It's kind of amazing how cheap this service is when you think about what's involved but it's still expensive enough that it can be a bit daunting handing over your money then waiting to find out whether you've made some mistake that will ruin the end result.

I figured I'd get started with something simple, something I could test the waters of not just PCB assembly, but also e-commerce. My plan was to come up with a small item that I could realistically design and submit for manufacture in less than a day, then hopefully sell reasonably easily on a market place like Etsy. I was expecting the design and manufacture side of things to be the hard part and listing the product on Etsy to be easy, but it was very much the other way around as you will see if you keep reading!

## The product
I had a quick scan of Etsy to look at the kinds of things people were successfully selling that I could make. I found a couple of examples of people making electronic pin badges - the perfect project! These mostly consisted of LEDs with either a random or pre-set flashing pattern on a novelty shaped board. These looked alright to me, but I wanted to try something a bit more interactive and meaningful that I could sell for enough money to make a sensible margin on a small number of initial units. 

Looking at the regular pin badges being sold on Etsy I saw a bunch of enamel 'Social Battery' pins with a sliding indicator. These immediately jumped out at me as something that a) I could identify with personally (people make endless jokes about my social battery...), b) would be really fun as an electronic version, and c) would be easy to design - just a few LEDs, a switch, and a microcontroller to tie it all together!

## Designing and prototyping the board
Confident in my skills I threw together a quick circuit diagram in Kicad. I decided to use an ATtiny13A - mainly because I had a few on hand from a previous project, and because I  have a decent amount of experience with similar chips. If you're not familiar with it, the ATtiny13A is a small 8bit microprocessor with 6 IO pins and is part of the AVR family of MCUs. It's similar to the ATmega chips that have historically been at the core of most Arduinos, except its capabilities are much more limited. The benefit is that the ATtiny range of chips are smaller and cheaper, so if you don't need a lot of memory or perhiperals then they are great! (although possibly a bit dated now with the endless variety of ARM chips that are available).

I laid out the PCB for my circuit in KiCad then got it to spit out the gerbers and drill files (which are used to manufacture the PCB), and the bill of materials and placement files (which are used to assemble components onto PCBs). I submitted them to JLC PCB to see if they were able to process them correctly. The BoM and placement files needed a bit of tweaking from the default to get them in the right format (turns out I did this the hard way, and theres a much easier plugin for KiCad that does everything perfectly in a single click). 

JLC PCB maintains a pretty large library of components that they keep in stock for assembly orders. However in my case the ATtiny13a was not available so I had to order them in. This was pretty straight forward using their global sourcing service. I was able to find the supplier with the best price for the quantity I needed and then let JLC PCB order them to their warehouse on my behalf. 

Whilst I waited I figured I might as well breadboard my design and start working on the code, and I'm glad I did because I immediately discovered an issue! My design used  five of IO pins available on the the ATtiny13a to drive the LEDs directly (with the sixth being used to monitor the button). What I didn't realise/remember from the last time I made this mistake (yes, It's happened before) was that one of those IOs is also the reset pin. You can use it, but it's not able to supply much current, and by tying it to ground through an LED I was keeping the chip in a permanent reset state.

One way to get around this is to burn a fuse on the chip that permanently disables the reset functionality of the pin transforming it into a regular IO. The issue with this though is that you can only program the chip once (unless you own a high voltage programmer, which I don't) and given my propensity for learning things the hard way, that seemed potentially quite wasteful!

The other option is to find a way to do more with fewer pins, so that the reset pin can be left alone. This can be achieved using a technique called [charlieplexing](https://en.wikipedia.org/wiki/Charlieplexing) which allows you to address many more LEDs than the number of available pins. You can then scan through these LEDs, turning them on and off individually at a high rate to make it appear as if multiple are on at once through [persistence of vision](https://en.wikipedia.org/wiki/Persistence_of_vision). In my case I'm driving the four green LEDs from 3 pins, and the red LED with a dedicated pin. This isn't the most efficient example of charlieplexing, as I could drive all the LEDs from the three pins, but keeping red on a dedicated pin allows for simpler code when it comes to programming. 

I revised my circuit diagram and PCB design, then re-exported the necessary files and sent them off to JLC PCB to manufacture and assemble an initial set of five prototypes. 

A little over a week later I received my prototypes and I was really pleased with them! There were a few things that I could see I needed to change:

1. I hadn't really planned how I was going to program the boards. I had just broken out the AVR programming pins and hoped for the best. This was fine for five prototypes as I could tack the programming wires on with the tiniest bit of solder but  this would get very tedious very quickly at scale.
2. The spike for the fastener on the back was soldered onto a ground pad. This would have been fine except the thermal mass of the spike and the ground plane on the board made it difficult to make a good join. It also sticks out very close to the positive metal cage of the battery holder which meant that there was a good chance of accidental shorts occurring if people placed/attached the badge on/to conductive surfaces.
3. The negative contact for the battery wasn't prominent enough so I had to add a dab of solder to it to make a good connection. Again - not the end of the world for a small number of prototypes, but a pain if I had to do this for loads.

These were all simple fixes to implement. I added a proper programming header that could be used with a pogo pin jig for quick and repeatable programming. I disconnected the spike from the ground plane so that it was its own small disconnected island of copper that it would heat up more easily, and not cause shorts. And I expanded the negative contact for the battery so that it would have more surface area to make a solid connection.

The code is pretty simple. I keep track of the mode the badge is in which is represented by an integer that gets decremented every time the button is pressed. I then loop and blink each LED on my way round as dictated by the mode. When mode 0 is reached I reset it back to the original number and then put the ATtiny into sleep mode. Pressing the button again triggers the interrupt which wakes the chip and starts the whole process again.

Overall I was really pleased with the results, which meant it was time to see if I could sell them!