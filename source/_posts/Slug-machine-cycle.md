title: Slug machine cycle
date: 2015-12-15 18:06:06
tags:
- Slug

thumbnailImage: thumb.jpg
thumbnailImagePosition: left
coverImage: cover.jpg
coverMeta: out
---
While CPU instruction set defines what CPU can do, the micro-code defines how it does this in hardware. Each Slug/4 instruction code is combined with CPU state (FLAGS) to form 10-bit address of 23-bit CONTROL word. Not every CONTROL word is legal. For example an illegal CONTROL word can create bus conflict by enabling multiple devices to drive at once. Another example is when one or more devices are expected to be driven but none are actually driving a bus. Testing these conditions in real hardware is not easy. Solution is to aid validation by having software simulator for Slug. Slug simulator is a [nodejs](https://nodejs.org) program that runs on conventional PC and allows to do the following.

* Validate machine cycle;
* Validate micro-code;
* Test and debug Slug/4 programs.

<!-- more -->
Before I can write Slug simulator I need to understand what happens within a single machine cycle. In general machine cycle consists of following stages:

* 0: _Program counter_ (PC) is active at program ROM address inputs;
* 1: Instruction plus FLAGS bits are active at micro-code ROM address inputs;
* 2: CONTROL word is active;
* 3: ADDRESS bus is active if _addressing unit_ (AU) is output enabled;
* 4: DATA bus is active if _execution unit_ (EU), input I/O port, RAM or immediate operand are output or read enabled;
* 5: RAM is written from DATA bus if write enabled;
* 6: EU, AU registers, I/O output registers are loaded from DATA bus if load enabled; PC register is incremented or loaded from ADDRESS bus.

In simulator program stages have strict order from 0 to 7. This order is needed to satisfy input/output dependencies and also makes writing simulator easier. In real hardware things are more complicated. Dependent signals are ordered by nature of [propagation delay](https://en.wikipedia.org/wiki/Propagation_delay#Electronics), but otherwise stages happen in undefined order.

![](naive_cycle.png)

If all stages are simply ran inside one _clock_ (CLK) cycle stages 0, 1 and 2 will happen sequentially. Stages 3, 4 and 5 will be initiated at the same time by active CONTROL word. Given propagation delays of activated CPU devices there will be some sequence in which they happen---a highly unpredictable one. Stage 6 is synchronized on raising edge of CLK signal. At first it seems that this machine cycle might work. DATA bus is active when it needs to drive loading AU and EU registers. Also ADDRESS bus is active when it needs to drive loading PC or accessing RAM. Problems begin when RAM is enabled for writing. RAM is using ADDRESS bus for selecting memory location to write value provided on DATA bus. Yet ADDRESS bus and DATA bus may not be set when RAM writing is already enabled. This will result in invalid writes using random data and addresses. Another problem may occur when EU/AU registers and PC register are loaded from DATA and ADDRESS buses correspondingly. Writing to these registers happens on raising CLK and requires certain [hold time](https://en.wikipedia.org/wiki/Hold_time). This hold time races with propagation delays of DATA and ADDRESS. Usually hold time of a flip-flop is a lot less than propagation delay of EEPROM, but relying on this can be risky.

![](real_cycle.png)

Solution to both races is to split machine cycle into 3 phases.

* Setup phase: accommodate propagation delays to reliably form CONTROL, DATA and ADDRESS;
* Write phase: write enable RAM;
* Hold phase: keep holding CONTROL, DATA and ADDRESS for PC and EU/AU hold times plus respect propagation delay to read next instruction.

Base CLK is divided by 2 to form _fetch clock_ (FCLK) full period of which is new machine cycle. CLK skipping every other pulse forms write clock (WCLK). In order to align active instruction and FLAGS with machine cycle I added 8-bit FETCH buffer at program ROM output and 2-bit FFLAGS buffer at FLAGS output. FETCH and FFLAGS buffers are the only devices in CPU clocked with FCLK. Therefore it holds instruction for the entire duration of machine cycle---CONTROL, DATA and ADDRESS follow with corresponding delays. Setup phase should be long enough to fit micro-code ROM read delay plus RAM or register read delays so that CONTROL, DATA and ADDRESS become valid. Write phase should long enough for RAM to write. Finally hold phase should be long enough to fit hold time for AU/EU and PC registers---which load at raising WCLK---and for program ROM delay to read next instruction. Next instruction is loaded into FETCH at raising FCLK while program ROM output is holding for the duration of the next cycle's setup and write phases.

With CLK divider based on a single flip-flop I get CLK/2 and ~CLK/2 (inverted). If FCLK=CLK/2 with positive OR logic I can make WCLK=(~CLK/2)|CLK. This will have setup phase in the first 1/4 of machine cycle, write phase in the second 1/4 and hold phase in the remaining 1/2. At 2MHz CLK machine cycle will be **1000 ns** long, which translates into 1 million Slug/4 instructions per second. Phases will have following durations and correspondingly accommodate longest delays from parts data sheets.

* Setup phase: **250 ns** > [CAT28C16A](https://www.jameco.com/Jameco/Products/ProdDS/74691.pdf) micro-code EEPROM read cycle (**200 ns**) + SRAM read cycle (**20 ns**);
* Write phase: **250 ns** > [CY7C168A](http://www.cypress.com/file/103106/download) SRAM write cycle (**20 ns**);
* Hold phase: **500 ns** > [AT28C64B](http://www.atmel.com/Images/doc0270.pdf) program EEPROM read cycle (**150 ns**).

By using more ICs I could get shorter write and hold phases and raise CLK to 4MHz and 2 million Slug/4 instructions per second. After that I hit EEPROM speed limits. 
