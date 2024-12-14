---
layout: post
title: Trunks - Satellite Link Simulator
date: 2024-12-14 10:54 +0100
categories: [Network]
tags: [network, satellite]
image: https://user-images.githubusercontent.com/41422704/111051142-37e67680-8451-11eb-9e1b-c3cdee7e0064.png
---

During my PhD, I had to work on something very amazing related to 5G-NTN and develop some Proof-of-Concept (PoC).
The funny part was when I had to work with containers and simulate/emulate a satellite link that could have variable parameters during a certain period of time (e.g., bandwidth increase/decrease, latency).
And I say funny because it was not in practice, trying to dockerize the incredible project opensand and adapt the code to something much more flexible...

So, I decided to develop something open-source and very simple that would only play with variable latency and bandwidth, the result: **Trunks - The Satellite Link Simulator 🛰️**.

The idea behind this was to leverage all the tools provided by the Linux kernel to build a very simple tool and easy to dockerize to simulate the behavior of a satellite link at the L3+ layer between a gateway and a satellite and as well between a satellite terminal and a gateway.

To achieve this, I had to implement the forward and return links of the satellite connectivity, which materialize in a different packet processing by the network stack, and that is quiet complex when you are not a linux Quality of Service (QoS) expert, but everyone learns right!

## First challenge: differentiate packets arriving on network interfaces and process them in different pipelines 🛣️

Using Linux, to be honest, there are not many options here for low overheads and efficient pipelines; I had to go play with netfilter and mark the packets arriving on each interface I could keep track of each packet in the network stack.

This can be done using the **MARK** target and the iptables command to match packets on specific criteria and mark them with a specific value.

I had to do this for all packets arriving on my interfaces, if they were entering *eth-a*, they were marked as belonging to the return links and if they were entering *eth-b*, the were marked as belonging to the forward link and I was able to process them in a different manner.

Now, first step done, using **MARK**, I could tag the packet and differentiate them into the linux kernel. Now the pipelines themselves!

## Second challenge: implement different processing pipelines ⚙️

Now that we are able to identify all packets of the forward/return links, I could go for implementing the different processing. Here again, linux embeds all the tools I needed to implement bandwidth limitation and delay simulation.

The famous traffic control (tc) module allows to modify the Queuing Discipline (qdisc) of each ethernet interface to change the default First-In-First-Out behavior of the packet scheduler. Here comes the real fun if you never had any experience with stateless/stateful qdisc, qdisc type, qdisc parameters, it rapidly becomes a nightmare and can take days to fully go through all the available disciplines and understand how they really works.

I will skip you that part and only jump to the selected qdiscs to configure the packet processing pipelines:
- Hierarchical Token Bucket (HTB) - Stateful: this one is used for bandwidth management to configure the maximum available bandwidth of the forward/return link. I am also skipping the stateful specificy here with the parent/leaf configuration, I will deep dive more into that later when I will focus on the Quality of Service (QoS).
- Network Emulator (Netem) - Stateless: this one is used for delay management and will be used to simulate the altitude of the satellite (LEO, MEO, GEO). Very simple and straighforward qdisc which allows the configuration of a fixed delay as well as an offset to replicate more real behavior of the latency with fluctuent values.

In a nutshell, using a chain of *HTB* and *NETEM*, it is possible to add delays to packet and limit cap the max bandwidth of certain flows that could match a certain amount of information.

**Ok then, but how can we chain HTB with NETEM?**

Here comes the magic of tc, it is possible to combine as many as stateful qdisc that you want to create a tree-like structure with each node being a qdisc configuration and the last node of the tree branches being the stateless qdisc. We can obtain something like this:

ROOT (HTB 1) -> CHILD (HTB 2) -> CHILD (HTB 3) -> CHILD AND LAST (NETEM)

In that case we will have 3 HTBs chained together that could apply different processing to the packets and then the last element (leaf) NETEM which adds the delay!

With this amount of flexibility offered by the Kernel, I was able to implement both return and forward pipelines with each packet processed according to the mark set in the previous step.

A full example (bash script) is given below to show how it works:

```shell
#/bin/bash

# ======= Trunks based model =======
ST_IFACE=enp0s8
GW_IFACE=enp0s9

UP=20mbit
RET=50mbit
DELAY=250ms
OFFSET=100ms

# configure redirection for tc
sudo iptables -F -t mangle
sudo iptables -t mangle -A PREROUTING -i $ST_IFACE -j MARK --set-mark 10
sudo iptables -t mangle -A PREROUTING -i $GW_IFACE -j MARK --set-mark 20
sudo iptables -L -t mangle -nv

# =========== Configure rules for return link ============
# configure TC
sudo tc qdisc del dev $GW_IFACE root
sudo tc filter del dev $GW_IFACE

# configure rules for TC
sudo tc qdisc add dev $GW_IFACE root handle 1:0 htb default 30
sudo tc class add dev $GW_IFACE parent 1:0 classid 1:1 htb rate $UP
sudo tc qdisc add dev $GW_IFACE parent 1:1 handle 2:0 netem delay $DELAY $OFFSET distribution normal
sudo tc filter add dev $GW_IFACE protocol ip parent 1:0 prio 1 handle 10 fw flowid 1:1
sudo tc -s qdisc ls dev $GW_IFACE
sudo tc -s class ls dev $GW_IFACE
sudo tc -s filter ls dev $GW_IFACE

# =========== Configure rules for forward link ============
# configure TC
sudo tc qdisc del dev $ST_IFACE root
sudo tc filter del dev $ST_IFACE

# configure rules for TC
sudo tc qdisc add dev $ST_IFACE root handle 1:0 htb default 30
sudo tc class add dev $ST_IFACE parent 1:0 classid 1:1 htb rate $RET
sudo tc qdisc add dev $ST_IFACE parent 1:1 handle 2:0 netem delay $DELAY $OFFSET distribution normal
sudo tc filter add dev $ST_IFACE protocol ip parent 1:0 prio 1 handle 20 fw flowid 1:1
sudo tc -s qdisc ls dev $ST_IFACE
sudo tc -s class ls dev $ST_IFACE
sudo tc -s filter ls dev $ST_IFACE
```

## Third challenge: dynamic changes ⧧

With the different pipelines implemented, now need time for variable parameters...

For this one, there was no choice, I was good for a good dev session to change the parameters dynamically, and the time was for a wrapper implementation.
I initially wanted to go for Python, but as I wanted it to be very light and compiled (remember, this is for container deployment), I went for a Golang wrapper.

Using a YAML configuration file, the wrapper would change the bandwidth and delay parameters over time, making the return/forward adapt to various situations.

The combination of the wrapper, tc, pipelines and netfilter is the core of **Trunks**! Its inner system is given below in the figure

[![satellite](https://user-images.githubusercontent.com/41422704/111052860-3fad1780-845f-11eb-9e6b-c24d55909ee1.png)](https://github.com/shynuu/trunks)
_Trunks Inner Gears with Traffic Control and Netfilter_


Thanks a lot for reading until the end, I tried to recap how Trunks was working internally in a simple manner but still giving sufficient details for people wanting to contribute. In the future, I will detail more the QoS aspect as I also had to work with fine-grained QoS…

The code is open-source and can be found on [Github]( https://github.com/shynuu/trunks).

I am currently working on a newer version that could also be used to simulate ISLs/OISLs between satellites and will soon give some updates on this one.


> Trunks has been used in the work Slice Aware Non-Terrestrial Network published in IEEE LCN 2021: Y. Drif et al., "Slice Aware Non Terrestrial Networks," 2021 IEEE 46th Conference on Local Computer Networks (LCN), Edmonton, AB, Canada, 2021, pp. 24-31, doi: 10.1109/LCN52139.2021.9524938.
{: .prompt-info }