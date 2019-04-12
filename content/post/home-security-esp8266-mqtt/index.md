---
title: "Integrating existing home security sensors with MQTT"
date: 2017-03-28T00:00:00-04:00
tags: ["home-assistant","mqtt"]
categories: ["Home Automation"]
---
When my wife and I bought a house a couple years back, I knew it would only be a matter of time before I started getting into home automation. My house, like many built in the late 90s, was pre-wired for an alarm system. While I had no desire to revive a 20-year-old alarm panel, it did mean all my exterior doors were pre-wired with inconspicuous sensors. I already run [Home Assistant](https://home-assistant.io) on a Raspberry Pi, so I was looking for a way to integrate these hard-wired door sensors with what I already have. I had read about these cheap WiFi-enabled [ESP8266 boards](https://www.amazon.com/gp/product/B010O1G1ES), so I decided this would be a simple project to try it out with.
<!--more-->

## Breaking into the existing alarm system

The brains of my old alarm system were tucked away in a wall-mounted cabinet in my laundry room.

{{% img "img/panel.jpg" "The old alarm system (with bonus lead-acid battery that I need to get rid of)" %}}

All the sensor wires were cut back, meaning I had to strip each one and trace what they did. It was decently quick work with my multimeter and a wife to open/close doors for me. Since the existing sensors were just simple [reed switches](https://learn.sparkfun.com/tutorials/reed-switch-hookup-guide/reed-switch-overview), the two wires from each door would short when the door was closed, and open when the door opened. I guessed correctly that the small 2-conductor wires went to my three exterior doors, but honestly, I’m still not sure where the rest of them go.

{{% img "img/connections.jpg" "While I typically don’t use wire nuts in my projects, it made the most sense here since I was dealing with pre-existing structured wiring from the old security system." %}}

## Making it work on a breadboard

Now the fun part: building it. This is a super simple circuit, with just a microcontroller and 3 switches. I used 3 of the GPIO pins to connect to the 3 door switches and connected the other side of the switches to ground. One nice feature of this board was that most of the pins had a built-in weak pull-up resistor. This meant that when a switch was open, the pin would be pulled high (3.3V).

{{% img "img/breadboard.png" "The components labeled S1-S3 represent the reed switches already installed in the doors. When a door opens, the magnet in the door causes the switch to open (or disconnect)." %}}

## Adding some code

[All of my source code is on GitHub](https://github.com/patrickeasters/nodemcu-sensor-mqtt), so feel free to check out that repo and see how it works there. This is my first time using Lua, so I’m sure there are things that can be optimized in this code. Feel free to submit an issue or PR if you find anything that stands out!

*   **init.lua**: the simple startup script. Best practice is to keep this file relatively simple and give yourself an opportunity to break out of the boot cycle should your app have an issue.
*   **secrets.lua**: secret variables such as WiFi credentials
*   **config.lua**: just some simple configuration variables
*   **sensor.lua**: the brains of the project. All of the real logic lives in this file.

I won’t get too detailed here, but after establishing the initial WiFi and MQTT connections, the application code is triggered asynchronously by an interrupt when a change on one of the door sensors is detected.

## Preparing the NodeMCU board

Before I could run my code on the NodeMCU board, I needed to flash it with updated firmware that contained the modules I needed. I used this handy [Cloud Build Service](http://nodemcu-build.com/) to build a firmware image. For my needs, I simply needed the following modules: file, gpio, mqtt, net, node, tmr, uart, wifi.

{{% img "img/pyflasher.png" "" %}}

To actually flash the firmware, I used an open-source Python tool called [nodemcu-pyflasher](https://github.com/marcelstoer/nodemcu-pyflasher). It worked great on my Mac and should on most other platforms as well.

{{% img "img/esplorer.png" "" %}}

Now that I was ready to upload my code, I used a tool called [ESPlorer](https://github.com/4refr0nt/ESPlorer) to handle that portion. It provides a pretty handy console that lets you run commands ad-hoc as well as edit your code. I personally found it easiest to just use my usual text editor and the “Upload” button in the left pane.

## Integrating with Home Assistant

Like I mentioned before, my home automation platform of choice is an open-source app called [Home Assistant](https://home-assistant.io). It has a strong community backing and the maintainers are constantly pushing new releases. While I could have used the built-in REST API, I opted to use the [MQTT](https://home-assistant.io/components/mqtt/) protocol for a couple of reasons: first, MQTT is super lightweight and perfect for the modest compute resources of the NodeMCU board; and second, it’s a more universal protocol that can be used outside of the Home Assistant ecosystem without changes. If you use Home Assistant, I configured my door sensors as **binary_sensors**. The snippet for one sensor is below, but you can look at the [rest of them](https://github.com/patrickeasters/smart-house/tree/master/binary_sensors) in the Github repo for my config.

```yaml
- platform: mqtt
  state_topic: "pat/alarm/Front Door"
  name: "Front Door"
  payload_on: "open"
  payload_off: "closed"
  device_class: opening
```

#### Wrapping it up

Thanks for reading this far! Hopefully this project inspired you to build something of your own or saved you a bit of time as you tackle something similar. I’m happy to clarify anything or answer questions, so feel free to reach out here or on GitHub.

#### Resources

[NodeMCU Documentation](http://nodemcu.readthedocs.io/en/master/) (super thorough and helpful)
[Project Code](https://github.com/patrickeasters/nodemcu-sensor-mqtt)
[Home Assistant Config](https://github.com/patrickeasters/smart-house/blob/master/packages/mqtt_doors.yaml)
