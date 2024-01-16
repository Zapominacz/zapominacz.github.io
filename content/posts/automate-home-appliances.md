---
title: "Automate home appliances"
date: 2024-01-17T21:00:00+02:00
authors: ["Mikołaj Wilczek"]
---
Smart home and Internet of Things (IoT) devices are more present in our homes today. This process started several years ago and will continue with the old devices being changed.

**TLDR: Smart sockets and power consumption analysis.**

### Non-smart appliances
As I've already automated half of my home, I look at my 3 years old dishwasher with sadness, knowing a year later a "smart" version was released. But there is no reason to change the dishwasher only because newer has an app provided.

### Not-so-smart appliances
On the other hand, smart home appliances are equipped with lots of unnecessary bloatware. My washing machine has a website wrapped in a native app to manage. Some of the functions are great - sending notifications about the finished cycle, own washing program or diagnostics. However, there are tons of unnecessary features that I've never used: current washing cycle, washing machine state, spin speed etc. Surely, they may have their target, but whatever you do, with the current state of technology you need to:
* load your laundry manually
* unload it manually
Due to this fact some of these functionalities are unnecessary because you will either want to have as small interactions with the washing machine as possible or it is quicker to operate it manually from the appliance's buttons.
There are also security concerns related to smart functionalities architecture. Usually, all the data is forwarded to the vendor's cloud. This is great for fast setup, but if you already have a smart home hub (like Home Assistant) - this is another uncontrolled device that communicates from your home.
Additionally, usually, there are no integrations available (due to security concerns...). My washing machine has an integration that simulates a smartphone.

## How to automate them
In both cases, you want to provide smart functions as simple as possible.

You need a smart socket. I've got plenty of Zigbee sockets that identify as `TS011F`. 

![TS011F](/automate_home_appliances/socket_TS011F.jpg)

The socket should feature:
* Active power measurement
* Child lock function (if have a button - I usually turn it off by accident)
* Power-on configuration

### First run - measure
After the socket is plugged - leave it for 1-3 use cycles to measure active power in particular stages (at least idle and active). The bigger difference the better. Always remember to have a margin due to power oscillations.

### Implement automations
After measurements implement automations. In my case (Home assistant, `TS011F`), there is a handy trigger that activates when power is lower/greater than X in Y seconds.

After a trigger is activated, it is a good idea to send a notification to the smarthome. Notice that other uses will probably need a little different triggers (notification versus power off).

### 3d printer power saving
My Prusa printers have Home Assistant integrations, but they don't have strict information (for now) if the heatend fan is disabled - and this is a problem because disabling the printer while cooling off can damage your extruder. I've identified that:
* MK3 with Rpi uses ~150-250W during use, ~15W when cooling and no more than 11W in idle (when no cooling is needed)
* MK4 uses ~150-250W during use, ~20W when cooling and no more than 14W in idle (when no cooling is needed)
  
![TS011F](/automate_home_appliances/mk4_power.png)

It is a great way to save the power :)


