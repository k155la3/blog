title: Slug micro-code
date: 2015-12-15 18:06:06
tags:
- Slug

thumbnailImage: thumb.jpg
thumbnailImagePosition: left
coverImage: cover.jpg
coverMeta: out
---
While CPU instruction set defines what CPU can do, the micro-code defines how it does it on given hardware. Each Slug/4 instruction code is combined with CPU state (FLAGS) to form 10-bit address of 23-bit CONTROL word. Not every CONTROL word is legal. For example an illegal CONTROL word can create bus conflict by enabling mupliple devices to drive at once. Another example is when one or more devices are expected to be driven but none are actually driving. Testing these conditions in real hardware is not easy. Solution is to aid validation by having software simulator for Slug. Slug simulator is a [nodejs](https://nodejs.org) program that runs on convetional PC and allows to do the following.

* Define structure of machine cycle;
* Validate micro-code;
* Test and debug Slug/4 programs.

<!-- more -->
Before I can write Slug simulator I need to understand what happens within Slug machine cycle. In general machine cycle consists of following stages:

0. _Program counter_ (PC) drives program ROM address inputs;
1. Slug/4 instruction plus FLAGS drives micro-code ROM address inputs;
2. CONTROL word enables CPU devices;
3. ADDRESS bus driven by _addressing unit_ (AU);
4. ADDRESS is used to load PC or to drive RAM address inputs;
5. DATA bus is driven by _execution unit_ (EU), input I/O port, RAM or immediate operand;
6. DATA is used to drive EU, load one of AU registers, output I/O port or write to RAM;
7. PC is incremented.

In simulator program stages have strict order from 0 to 7. This order is needed to satisfy input/output dependencies and also makes writing simulator easier. In real hardware things are more complicated. Input/output is ordered by nature of IC timing, but otherwise stages happen in parallel.
