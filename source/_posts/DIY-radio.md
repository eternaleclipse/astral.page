---
title: DIY radio
date: 2023-09-24 13:36:16
tags:
- electronics
- tech
- diy
---

After a very long time of not soldering anything, I finally got myself an [RDA5807 FM Radio Receiver DIY Kit](https://www.icstation.com/rda5807-radio-receiver-with-flashing-indicator-p-15922.html) from [Amazon](https://www.amazon.com/gp/product/B09Y1BJZ6F/). Spent 3 hours straight on this thing and my soldering-phobia is basically gone now. This included some SMD soldering, a bunch of DIP mount and pin headers, wiring, a very hacky antenna solder and frame assembly. I did miss something though because the frequency "Down" button doesn't work.

Some ideas for future improvements:
- Fix the "Frequency Down" button
- Add a 3.5mm headphone jack
- Some finer volume control (currently there are volume levels 0-10 where "0" is still quite loud)
- Replace the power socket with a USB-C socket
- Add an internal usb power bank
- Read the diagram, understand the purpose and function of each component
- The holes for the antenna and power socket could use some work too. I'm thinking about using a laser-cutter on the acrylic panels to make the holes larger
- Make a custom case that would fit the radio and the power bank
- Mess with the [STC89c52]() 8051 microcontroller that runs this thing (dumping, flashing, debugging the firmware)
- Reverse-engineer the firmware and make a custom one
    - The LCD screen says "STEREO RADIO" but it's actually mono - change to some custom text
    - Initial volume is really high, I'd like to change that to something more reasonable
    - Starting frequency is something like 80MHz, should be more like 100Mhz
    - Currently it scans until it finds a station. Explore the possibility of manually setting the frequency
    - Explore adding persistent memory for storing the last configured volume and frequency

But for now, I'm just so happy with it!

![](/images/diy-radio.jpg)