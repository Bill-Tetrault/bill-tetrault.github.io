---
layout: post
title: "Monoprice Whole Home Audio App"
date: 2026-07-25
categories: [projects, home-audio]
tags: [monoprice, whole-home-audio, ios]
---

I have been running Monoprice multi‑zone amps for whole‑home audio for a while, and wanted a simple way to manage zones and sources from my phone instead of walking around to keypads.[web:16] The Monoprice Whole Home Audio app ties into those controllers and gives a straightforward interface for turning zones on and off, changing sources, and tweaking basic audio settings.[web:6]

![Zones overview](/assets/images/monoprice-amp-zones.jpg)

## Zones view

The Zones screen lists each room with its current state, source, and volume. From here it is easy to:

- Turn individual zones on or off  
- See which source each room is using  
- Make quick volume changes  

This is mostly what I use day‑to‑day: check which rooms are active and shut things down when nobody is listening.[web:16]

![Zone controls](/assets/images/monoprice-amp-zoneadv1.jpg)

## Per‑zone controls

Tapping into a zone brings up more detailed controls. You can:

- Change the source for that room  
- Adjust volume  
- Enable a sleep timer so the zone turns off after a set interval  

For spaces like bedrooms, the sleep timer is handy so music does not stay on all night.[web:6]

![Advanced zone settings](/assets/images/monoprice-amp-zoneadv2.jpg)

The app also exposes bass, treble, and balance for each zone, which maps to the controls on the amplifier. That makes it easy to dial in rooms that need a little extra low end or to shift balance when speakers are not centered.[web:16]

![Sources view](/assets/images/monoprice-amp-sources.jpg)

## Sources view

On the Sources tab you can rename inputs to match your setup: TV, Radio, Aux, etc. Clear labels make it easier for other family members to pick the right input without remembering which numbered source is which.[web:6]

![Server settings](/assets/images/monoprice-amp-serversettings.jpg)

## Server settings

The Server tab is where you configure the system name and zones and point the app at the IP‑to‑serial adapter that talks to the amplifier.[web:6] Once that is in place, the app handles the day‑to‑day control and you do not need to think about the serial details again.[web:10]

## Install guide and repo

If you want to see how this fits together, I documented the setup and requirements in the GitHub repo:

[Monoprice Whole Home Audio App – GitHub](https://github.com/Bill-Tetrault/monoprice-amp)[web:12]

The install guide walks through supported amps, the IP‑to‑RS‑232 hardware, and the basic configuration needed to get the app talking to your controller.[web:6] It should be enough to get a similar system up and running if you already have the hardware in place.[web:10]