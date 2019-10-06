---
layout: post
title: What are we trying to build? - Air conditioner remote control
author: fpicalausa
---

This is the first article in a series about building a remote control for a air
conditioner over the internet. Check the [figure out the link](overview article)
if you want more details.

# What are we trying to build?

Our system should be able to change the air conditioner settings from a web interface. I have a
 Raspberry Pi 3b, so I will be using it for this project. 
Now, changing the settings is useful, but without a good idea of the current temperature in
 the room, it is difficult to know remotely if turning the air 
conditioner on makes sense. So we'll also want to build that.

My air conditioner is a Fujitsu AS-E4007T. It doesn't come with a web interface (otherwise I
 wouldn't be writing this), but it does come with an infrared interrace (a plain looking 
remote control). Fun fact: most phone cameras will pick up infrared. Point your phone camera 
at the remote control LED, press a button on the remote, and theblive preview should show the
LED blinking.

Given these parameters, here is what I came up with:






