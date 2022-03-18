---
title: "Feed the dog and close the door with an open source home automation system"
date: 2018-03-20T00:00:00-04:00
tags: ["home-assistant"]
categories: ["Home Automation"]
---
As voice assistants, smart bulbs, and other devices increasingly become household staples, more people than ever are bringing smart technology into their homes. But the bewildering assortment of products on the market can present challenges: Remembering which app to use and trying to link things together with automation can get complicated quickly. In this article, I’ll show you a few ways I used an open source home automation platform, Home Assistant, to bring all my devices together.
<!--more-->

## Getting started with Home Assistant
When looking for a hub, I wanted something easily extensible, with a strong community and support for devices. After considering various hubs and platforms, I decided [Home Assistant](https://home-assistant.io/) was the winner. It has an extremely active community, is lightweight enough to run on a Raspberry Pi, and best of all, it’s open source.

I won’t get into the setup process here; check out Home Assistant’s [Getting Started](https://www.home-assistant.io/getting-started/) guide to get up and running.

## Tapping into my home’s alarm system
My wife and I bought our first house several years ago. Though this wasn't exactly a selling point, it came pre-wired with an alarm system that featured all the best technology from the 1990's. I had no intention of reviving the alarm panel, but I did want to take advantage of all the door sensors it included.

After cracking open the alarm panel box in my laundry room, I decided this would be a great first project for an ESP8266 microcontroller. Using MQTT (a lightweight message queuing protocol), I was able to get instant status updates whenever my outside doors opened or closed.

## Automating the lights
Though quite a few devices were controllable through my Home Assistant instance, I still needed to control or trigger most things manually. The whole point of automation is to accomplish tasks with little or no human interaction. Using a Z-Wave motion sensor and the custom MQTT door sensors mentioned above, I decided to automate a few scenarios.

My first task was [turning on my front porch lights](https://github.com/patrickeasters/smart-house/blob/master/packages/auto_porch_light.yaml) when the door is opened after daylight hours. I used my front door as the trigger and took into account the current state of the sun to ensure I’m not turning on the lights when it’s still daylight.

The second task I automated was turning on a lamp when I come downstairs to get ready for work. Since I’m an early riser, it’s often quite dark when I am getting ready for my day. Using a motion sensor covering my stairs, I configured an [automation to turn on a lamp](https://github.com/patrickeasters/smart-house/blob/master/packages/auto_living_room_lamp.yaml) as I come downstairs. It’s also intelligent enough to turn the lamp off once there’s enough daylight outside.

## Closing doors
Since we have a dog who goes outside frequently and my wife and I are trying to wrangle an infant, it’s not uncommon for a door to be left open or not completely shut. Using a [simple automation](https://github.com/patrickeasters/smart-house/blob/master/packages/mqtt_doors.yaml#L23) with my MQTT door sensors, Home Assistant automatically sends a push notification to our phones reminding us to close the door.

I also have a [similar automation setup for my garage door](https://github.com/patrickeasters/smart-house/blob/master/packages/garage_door.yaml#L18), which will alert us when the garage is open after 8:30 pm. While my OpenGarage controller is also able to close the door, I opted to have it notify us in case someone is still working outside, or for safety reasons—for example, if something is obstructing the door and preventing it from closing.

## Feeding the dog
As our lives got more hectic with the arrival of our first kid, my wife and I found ourselves asking one particular question more often: “Did we feed the dog?” Staring at the food bin one morning trying to remember if I’d fed her already, I knew there had to be a better way. Using an extra Z-Wave door sensor attached to the lid of the dog food bin, I used Home Assistant to keep track of when we open the container. [Using a few automations](https://github.com/patrickeasters/smart-house/blob/master/packages/dog_food.yaml), I set up a system to notify us if we forget to feed our dog after a certain time each morning and evening. Though the dog is a bit disappointed to no longer enjoy “bonus” meals, it’s helped us keep our sanity.

## Going forward
You probably won’t have the same devices or problems to solve in your home, but I hope some of the things I’ve done will help you find ways to make your home smarter. The possibilities truly are endless.

[Follow my Home Assistant config repo on GitHub](https://github.com/patrickeasters/smart-house) to keep up with my latest additions or find some inspiration. It’s constantly evolving!

You can also find me on Twitter or the [Home Assistant community forums](https://community.home-assistant.io/). I love answering questions and seeing what other ideas folks in the community come up with.

> This article was originally published on [Opensource.com](https://opensource.com/article/18/3/smart-home-assistant) and is licensed under Creative Commons SA-BY 4.0.
