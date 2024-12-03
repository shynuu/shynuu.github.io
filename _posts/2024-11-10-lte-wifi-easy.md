---
layout: post
title: Seamless connectivity with LTE and WiFi - easy method
date: 2024-12-01 10:54 +0100
categories: [Network]
tags: [network]
---


I had to work on an R&D project where some devices embedding 4G-LTE (SIMCOM-4G) and WiFi chipsets were unstable and constantly disconnected from the 4G network.

I had to find a straightforward solution, using available tools to detect when the connection becomes unstable on the LTE interface and seamlessly connect to the available WiFi network.

A very simple way to do that is always to ping a “reliable” server in the background, and if the ping fails, we re-configure the default routes to use another available network interface. 

But before reconfiguring the default route, the device must connect to the WiFi network!

I have simple [tool](https://github.com/shynuu/if-reliability) (Golang) in order to do that.

The tool can be describe using this simple routine (in python)

```python
def reliability(wifi_if: str, wifi_ssid: str, wifi_pwd: str, server: str, max_retry: int = 5):
    reacheable: bool = True
    num_retry: int = 0
    while True and num_retry < max_retry:
        reacheable = ping_server(server)
        if not reachable:
            num_retry += 1
    connect_wifi(wifi_ssid, wifi_pwd)
    set_default_route(wifi_if)
```

I will work more on it and try to release a more elaborate tool which also take the physical layer properties as inputs.

> N.B: It only works on linux :)
{: .prompt-tip }