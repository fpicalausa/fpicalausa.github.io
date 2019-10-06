---
layout: post
title: Controlling an air conditioner remotely
author: fpicalausa
---

In the hot and humid months of Japanese summer, a very common problem is to come
back home and waste 10 precious minutes of time until your air conditioning unit
had enough time to cool down the room. There is also the unsurmountable problem
of trying to remember if you turned the air conditioner off after leaving home.
The same problem also happens in winter, where your room gets way too cold for
any thinking.

One solution to this problem is to give up entirely on air conditioning. It's
somewhat painful, but certainly energy efficient. Another solution is to remote
control your air conditioner. From the Internet. This latter solution allows you
to turn your air conditioner on from the train, and to check on it any time you
left home.

I set out to build this latter solution with a Raspberry Pi, a infrared LED, a
bunch of wires, and a lot of time on my hands. In this and the following
articles, I want to explain step by step how I built my over-the-Internet
infrared remote control for my air conditioner.

## Table of Contents

(tentative)

1. [What are we trying to build? (and a bill of material)](/2019/10/07/Air-conditioner-what-are-we-trying-to-build.html)
2. Controlling an infrared LED from a raspberry pi
3. Decoding the air conditioner protocol (Fujitsu AS-E4007T)
4. Talking to the air conditioner
5. DHT11 for decision making
6. A web interface to tie it altogether
7. Conclusion
