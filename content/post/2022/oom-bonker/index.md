---
title: Taking a whack at custom Prometheus alerting
date: 2022-05-11T22:00:00-04:00
draft: false
tags: ["kubernetes","raspberry-pi"]
categories: ["Kubernetes"]
---
After many years of being on-call under my belt, I never thought I'd say I have a favorite alerting method. But that changed after watching one of Justin Garrison's videos which had [an excellent depiction of how Linux's Out-of-Memory Killer works](https://www.youtube.com/watch?v=KNexvhb_DuY&list=PLehXSATXjcQHGYufa__n1y9WIUZjyNMEw&t=43s). I was no stranger to the [OOM Killer](https://docs.rackspace.com/support/how-to/linux-out-of-memory-killer/) visiting my Kubernetes clusters, so this gave me a dumb idea for a (perhaps) fun alerting mechanism: the [OOM Bonker](https://github.com/patrickeasters/oom-bonker).

<video width="600" playsinline autoplay loop muted>
  <source src="{{% imgref "img/bonker.webm" %}}" type="video/webm" />
</video>

<!--more-->

## How It Works
This project mostly consisted of hardware I had laying around, the Prometheus monitoring stack for Kubernetes, and a tiny bit of code to tie it all together. At a high level, I have a Raspberry Pi running a webserver waiting for alerts from Prometheus--at which point it commences bonking a poor container.

<img src="{{% imgref "img/flowchart.svg" %}}" width="600">

## Software
We'll configure a Prometheus alert to detect our out-of-memory conditions and then fire a webhook to a Raspberry Pi when comntainers are killed.

### Prometheus Configuration
To get started, we'll need a Prometheus instance configured to scrape kube-state-metrics, which is where we'll get metrics for container status. I use the [Prometheus Operator](https://prometheus-operator.dev/), but you can adapt the config to your environment as needed.

We'll be using the `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` query to find containers that were last terminated due to being OOM Killed. One caveat to this search is that it may not capture containers with child processes that were killed. I found [a good post](https://songrgg.github.io/operation/how-to-alert-for-Pod-Restart-OOMKilled-in-Kubernetes/) that gives a few alternatives using cadvisor metrics. I had difficulty getting those to work in my OpenShift cluster, so I fell back to using the query above from kube-state-metrics.

Since software evolves over time, I'll link you to the [patrickeasters/oom-bonker](https://github.com/patrickeasters/oom-bonker#prometheus-configuration) GitHub repo which contains the steps needed to configure Prometheus.

### Alert Webhook
With the Prometheus configuration out of the way, we need something to receive alerts. Prometheus includes some default integrations like Slack and PagerDuty, but webhooks are still the king of compatibility. All it takes is a web app that can accept a POST request with some JSON.

The Prometheus docs give a [full example](https://prometheus.io/docs/alerting/latest/configuration/#webhook_config) of what a webhook POST body looks like, but I've included a snippet below of what a webhook looks like for the alert we just configured.

```json
{
	"receiver": "oom-bonker",
	"status": "firing",
	"alerts": [{
		"status": "firing",
		"labels": {
			"alertname": "OOMKilled",
			"container": "eater",
			"endpoint": "https-main",
			"job": "kube-state-metrics",
			"namespace": "bonk",
			"pod": "memory-eater",
			"prometheus": "openshift-monitoring/k8s",
			"reason": "OOMKilled",
			"service": "kube-state-metrics",
			"uid": "49cc552f-3c00-4275-babb-e9598c7fec61"
		},
		"annotations": {},
		"startsAt": "2022-04-26T13:59:56.33Z",
		"endsAt": "0001-01-01T00:00:00Z",
		"fingerprint": "6e4ca3f5c5f99172"
	}]
  ...
}
```

I wrote a super simple Flask app that accepts the webhook requests and uses the [gpiozero](https://gpiozero.readthedocs.io/en/stable/) Python library to control a servo connected to the Raspberry Pi.

The [GitHub repo](https://github.com/patrickeasters/oom-bonker#webhook) contains the steps for copying the webhook app and configuring systemd to run it as a service.

## Hardware
As I mentioned before, most of the hardware was picked from things I had laying around. We all have that drawer with old electronics in our house, right?

### Electronic Components
I used the following components that I had on hand:

* Raspberry Pi 1B (yes, it's old--but it's hard to find a new Pi amidst our supply chain crisis)
* Generic servo ([purchased from SparkFun](https://www.sparkfun.com/products/11965))

A wiring diagram is included below, though it may likely vary if you're using a newer Raspberry Pi which has 40 pins instead of 26.

<img src="{{% imgref "img/hookup.svg" %}}" width="600">

### 3D Printed Enclosure
I didn't _have_ to 3D print an enclosure for this, but I wanted an excuse to exercise my CAD skills. I won't divulge how many Fusion 360 tutorials I had to lookup on YouTube, but I was able to fumble my way through it and only printed 2 prototypes.

I used self-tapping M2.3x6 screws for the Raspberry Pi and M3x6 screws for the servo.

{{% img "img/enclosure.png" %}}

Both the CAD model and ready-to-print STL file are [are available on Printables](https://www.printables.com/model/185578-oom-bonker)

### Other components
In the spirit of grabbing what I had around my house, the remaining components we need are:
* Tux penguin stress reliever (thanks for the swag, Microsoft!)
* A toothpick (close enough in scale to the 4x4 Justin used)
* One of an [assortment of springs from Home Depot](https://www.homedepot.com/p/Everbilt-Spring-Assortment-Kit-84-Pack-13554/203133714)
* A [3D printed shipping container](https://www.thingiverse.com/thing:1561981) scaled to about 40mm long

## Testing it out
Once it's wired up, there's no better way to test it out than by summoning the OOM Killer. Aside from the obvious answer of running Chrome in a container, I turned to [Stack Overflow](https://askubuntu.com/questions/1188024/how-to-test-oom-killer-from-command-line) and found a cool one-liner to quickly consume an infinite amount of memory: `tail /dev/zero`. In this project's Git repo, I provided a [spec for a pod](https://github.com/patrickeasters/oom-bonker/blob/13a9a69a38b0de7f8aaeca0ea4dfaadfbb185e49/k8s/hungry_pod.yml) that crashes 10 seconds after startup. Assuming it's all wired up correctly, poor Tux should be bonked momentarily.

Despite being a niche project, hppefully you were able to glean something from it. Let me know if you make something inspired from this—I'd love to see it!
