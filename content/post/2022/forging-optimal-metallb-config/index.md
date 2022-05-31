---
title: Forging an optimal MetalLB configuration
date: 2022-05-30T22:00:00-04:00
draft: false
tags: ["kubernetes","metallb"]
categories: ["Kubernetes"]
---
As someone who's been playing with a lot of Kubernetes on bare metal lately, I've come to appreciate [MetalLB](https://metallb.io/) (a load balancer implementation for bare metal). Nothing is worse than blindly pasting YAML into your terminal, then seeing `Pending` next to all your newly created services expecting cloud load balancers.

{{% img "img/waiting.jpg" %}}

MetalLB was the last thing I needed to make my tiny home lab cluster feel like a real cloud. When I was first configuring it, the hardest thing to wrap my head around was how traffic flowed in the different modes and traffic policies. I spent a lot of time reading docs and experimenting, so hopefully this post will help you understand the different modes and how they work with service traffic policies.
<!--more-->

# Choosing an announcement mode
MetalLB discovers services needing load balancers, allocates IP addresses for them, and advertises them outside of the cluster. For the latter piece of the puzzle, there are 2 primary modes for announcing load balancers: **Layer 2** and **Layer 3** (BGP), both of which may sound familiar if you've ever seen the [OSI model](https://en.wikipedia.org/wiki/OSI_model). Each mode has its pros and cons, so it's a real balancing act as you decide which one to use.

## Layer 2 Mode
Layer 2 mode is the easiest to get started with and will work in any environment—no fancy routers needed. One Kubernetes node will assume the role of advertising your service to the rest of your network, behaving as if a secondary IP was added to its network interface. But underneath, MetalLB is responding to [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) (or [NDP](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol) in the case of IPv6) requests on its own. Should the node advertising your service fail, another node will be elected and use gratuitous ARP/NDP to alert the rest of your network that your service has a new home.

If you're thinking that sounds a lot like failover instead of load balancing, you're not wrong. Because an IP address can only map to one physical (MAC) address at time, incoming traffic for the service will all come through one node. This can potentially become a bottleneck for some services, but is likely not a deal breaker most of the time.

## BGP Mode
At the expense of a more involved initial setup, [BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) mode can avoid the single-node bottleneck that comes with Layer 2. Despite BGP [being hard sometimes](https://blog.cloudflare.com/october-2021-facebook-outage/), don't let it dissuade you from trying it out. 

In this mode, each Kubernetes node essentially becomes a router and advertises routes to your services to its peers. As long as your upstream routers support BGP multi-path, traffic will be evenly-ish distributed to each node that advertises your service.

The main gotcha with BGP mode stems from how the protocol's stateless load balancing works. Instead of tracking connections, BGP hashes headers from the incoming packet (like IPs and ports) to determine which of the available next hops to use. When nodes are added or removed, existing connections will be randomly rebalanced based upon the hash. This can lead to clients receiving a one-time connection reset error when nodes come and go. MetalLB's docs include some strategies to [mitigate this disruption](https://metallb.universe.tf/concepts/bgp/#limitations) if this becomes an issue for you.

# Understanding how traffic flows
MetalLB is really only concerned with getting traffic __to__ your nodes. Once it reaches the node's networking stack, it's up to `kube-proxy` to direct traffic to your service endpoints based on your [external traffic policy](https://kubernetes.io/docs/concepts/services-networking/service/#external-traffic-policy). Each combination of announcement mode and traffic policy has its tradeoffs, so there is not necessarily one correct option.

<table>
  <tr>
    <th></th>
    <th>Layer 2 Mode</th>
    <th>BGP Mode</th>
  </tr>
  <tr>
    <th>Global Traffic Policy</th>
    <td>Traffic enters <strong>one</strong> node and is distributed to all pods matching the service selector</td>
    <td>Traffic enters <strong>all</strong> nodes and is distributed to all pods matching the service selector</td>
  </tr>
  <tr>
    <th>Local Traffic Policy</th>
    <td>Traffic enters <strong>one</strong> node and is only distributed to pods on the same node.</td>
    <td>Traffic is distributed to <strong>all</strong> nodes with running pods and is then distributed to pods on the same node.</td>
  </tr>
</table>

## Layer 2 Global
Generally layer 2 mode should be paired with the `Global` external traffic policy. This allows MetalLB to advertise one node's MAC address and for `kube-proxy` to evenly distribute traffic to all service endpoints regardless of node.

<img src="{{% imgref "img/l2_global.svg" %}}" width="100%">

## Layer 2 Local
On the other hand, Layer 2 mode with the `Local` external traffic policy results in traffic entering one node and only reaching service endpoints on the same node. Traffic is also dropped entirely if there are no endpoints on the same node.

This configuration likely isn't ideal when you have a multi-node cluster and more than one pod backing a service. But on a single node cluster, this behaves the same as the `Global` traffic policy. If you think of a compelling reason to use this, let me know.

<img src="{{% imgref "img/l2_local.svg" %}}" width="100%">

## BGP Global
The real fun starts with BGP. Combined with the `Global` external traffic policy, traffic will enter all nodes and then be distributed across all service endpoints in the cluster. If a node becomes unreachable, its BGP session will drop and the upstream routers will rebalance traffic amongst the other nodes. Unlike layer 2 mode, all nodes will accept incoming traffic, which can be handy when you have high throughput services and want to avoid overwhelming a single node.

<img src="{{% imgref "img/bgp_global.svg" %}}" width="100%">

## BGP Local
Lastly we have my personal favorite (is that nerdy to say?): BGP mode with the `Local` external traffic policy. In this configuration, each node containing at least one service endpoint will advertise itself and then distribute traffic to endpoints on the same node. I like this for at least several reasons:  it minimizes extra hops and the source IP is preserved since packets don't have to leave the node. The latter is extremely helpful when you don't have a CDN or other reverse proxy in front of your service and really do need the source IP.

One thing to watch out for when using this combination is your traffic distribution. Because traffic is equally balanced between nodes, then yet again balanced between pods on the node, the spread can become uneven if some nodes have more pods than others. This can be mitigated by using pod anti-affinity policies to spread pods across nodes. If this is a pain point for you, it may be worth following a [related issue](https://github.com/metallb/metallb/issues/1) that proposes some BGP magic to equal out the distribution based upon endpoint count on each node.

<img src="{{% imgref "img/bgp_local.svg" %}}" width="100%">

# Wrapping up
Hopefully by now you've learned a little more about how MetalLB works and some of the tradeoffs to consider. Reach out if you have any questions or feedback. Happy load balancing!
