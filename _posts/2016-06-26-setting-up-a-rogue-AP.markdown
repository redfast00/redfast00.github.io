---
layout: post
title:  "Set up a hotspot to MitM on Kali Linux Rolling"
date:   2016-06-26 00:00:00 +0000
category: Hacking
tags: [Kali, Wireless]
---
## What?
In this tutorial, I will show how to set up a wireless access point on Kali Linux Rolling. This
will enable you to MitM any device that connects to that hotspot, without having to manually set 
the gateway on the device. This can then be used with Wireshark or mitmproxy.

## How?
Plug in your second wireless card (I am using a TP-LINK TL-WN722N). Click the network icon in the top right corner of your screen, 
then click “Wi-Fi ”, then "Wi-Fi Settings", select the wireless card you want to use as a hotspot and click "Use as Hotspot".
Congratulations, you just set up a hotspot and some forwarding rules. You can now run Wireshark and other tools on the traffic.

## Setting up mitmproxy
First, enable IP forwarding in the kernel by running: `sysctl -w net.ipv4.ip_forward=1`.
Assuming that your hotspot uses `wlan1`, set the following iptables rules. They will redirect any
TCP traffic over port 80 and 443 to mitmproxy. If you want to intercept HTTPS, 
you will need to install the CA certificate on the device.

```
iptables -t nat -A PREROUTING -i wlan1 -p tcp --dport 80 -j REDIRECT --to-port 8080
iptables -t nat -A PREROUTING -i wlan1 -p tcp --dport 443 -j REDIRECT --to-port 8080
```
Now, you can run:

```
mitmproxy -T --host --port 8080
```