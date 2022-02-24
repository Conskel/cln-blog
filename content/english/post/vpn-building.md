+++
author = "King Consk"
title = "Building a VPN Service"
date = "2022-02-23"
description = "Create you own VPN service using headscale."
thumbnail = "images/vpn_thumb.jpg"
tags = [
    "VPN",
    "Headscale",
    "Tailscale",
]
draft = true
+++

I've been exploring using headscale to build my own VPN service.

<!--more-->

### Overview
There's a misconception that using a VPN will hide your activity on the Internet. While this is partially true the concern is around what information the VPN provider is collecting and how they [might be storing that information]( https://www.techradar.com/news/this-top-vpn-provider-may-have-leaked-millions-of-user-details).

It's simple enough to purchase a virtual private server from somewhere and run OpenVPN or Wireguard to host a single VPN server. But what if you wanted to mimic the ease of use that purchasing a third-party VPN provides? What if you wanted to be able to exit in many different countries around the world?

[Tailscale](https://tailscale.com/blog/how-tailscale-works/) is an interesting product that provides a key management layer to Wireguard along with some NAT traversal to allow hosts behind CGNAT to connect to one another. The control component is closed-source and while it has a free tier, you’re still placing trust in a third party. The client component on the other hand is open source. Enter [Headscale]( https://github.com/juanfont/headscale) – an open source project to mimic the functionality of Tailscale server component. 

One of the features of Tailscale is to allow a node on a Tailnet to advertise as an exit point. The intended use of this might be to allow access to resources in a non-Internet facing subnet by making the node a router. Nodes can advertise routes and access control policies can restrict which nodes can access what resources. Let's try and advertise a default route `0.0.0.0/0` with an allow all policy. 

### Architechture
We'll need the following components:
- [Headscale](https://github.com/juanfont/headscale/blob/main/docs/running-headscale-linux.md) server
- [DERP](https://tailscale.com/kb/1118/custom-derp-servers/#why-run-your-own-derp-server) server (Designated Encrypted Relay for Packets) - While not strictly necessary. Let's build one anyway.
- Exit points running [Tailscale](https://tailscale.com/download/linux)

Here's a diagram of what's going to be built. 
![vpn](/images/vpn.jpg)

And because we’re currently in the middle of a pandemic and I’m at home self-isolating, let’s automate the whole thing!