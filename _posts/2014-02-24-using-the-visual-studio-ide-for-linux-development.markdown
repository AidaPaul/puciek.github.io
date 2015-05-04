---
layout: post
title:  "Using the Visual Studio IDE for Linux C++ development"
date:   2014-02-24 12:15:0
---

Before we will get to the more technical topics, I figured out that I will share one amazing way to develop C++ code for Linux easier and more pleasant than you ever thought it can be. And you will achieve that by getting Windows with latest Visual Studio. If you want to skip the boring personal story, search down to “oh boy” ;).

<!--more-->

I develop a lot of C++ code, a big chunk of it for Linux, and yet I cannot stand the switch between Visual Studio and “the others”. Netbeans is slow and cumbersome; CodeBlocks “quirky” in it’s own special way and Codelite is just a very wrong tool for any plus size project. And then there is Visual Studio, which supports all the cool tools (IntelliTrace is a beast), works at blazing speed and just gets better and better with every year. And let’s not forget about plethora of add-ons you can get for it, including language kits that allow you to move your Python development into Visual Studio, which works greatly but let’s not get sidetracked here.

So one day I was at home and was left with a choice – I could either once again fire up my local Gentoo VM and do what I always do, or seek another solution. And that particular day I was mad at CodeBlocks, so I started googling for an alternative. After going through life lessons of “learn to use the console”, “use cygwin to build the libraries on windows” (that won’t work for me as I often deal closely with hardware and kernel) and other unimpressive workarounds.
And then I ran into VisualGDB by Sysprogs.

Since they offer a free trial, I decided to give it a try. I won’t go over the tutorial since it’s already on their website, and you can check it out yourself, what I will tell you about it is how it turned out. I must admit that it took it’s sweet time ferrying over the whole environment to my windows box, but once he was done setting up I was stunned. IntelliSense worked without any lag or issues, debugging feels just as good as you would expect, so all breakpoints are honored, you can capture exceptions, browse the stacks and essentially all you would expect from a modern debugger.

On top of that it got a bit of fluff that makes remote work just that much easier. For example you can quickly embed a ssh console (based on SmarTTY) which will allow you to quickly, and painlessly run commands, browse logs and transfer files. All this keeps in the spirit of Visual Studio changes which slowly, but surely, turns it into a single, and only, application you will ever need for whole SDLC.

I could just go on, and on, and on but why bother? We are all coders – fire up your Windows box, download a trial and fall in love with your new way to develop C++ for Linux. A much more pleasant way.