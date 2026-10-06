+++
title = "On private IP addresses"
date = 2026-10-07
+++

# Motivation

While debugging logs, I've noticed the following pattern quite often:

```
read tcp 10.0.0.77:60254->192.168.0.158:27017: i/o timeout
```

Obviously, the IO timeout is concerning, but this post is going to be about this pattern of IP addresses. Kinda niche, but I also found out recently that I can configure my router if I simply visit `http://192.168.0.1/`. Clearly I know very little about this subject and need a refresher from my UG Computer Networking course. This is going to be a fairly short post which you may find very basic. But I swear, I seem to have forgotten all about this.

# Example

If you're reading this from a computer in your home, it is highly likely that you're part of a "private" network and your exposure to the outside internet is only through your router. For example, if I type the following command in my terminal, I get:

```
~ ❯ ipconfig getifaddr en0
192.168.0.100
```

But if I go to a website like [whatismyipaddress.com](https://whatismyipaddress.com)., I get a totally different IP address. What is going on?

# Explanation

The smart people who designed the internet later came up with an idea called "Network Address Translation (NAT)" when they realised that we may soon run out of IPv4 addresses and saw that devices that are part of a local network like an office or home need not each have a globally unique IP address. Instead, each device in the local network can get a private address and its communication with the outside world can happen through the router as an alias. They assigned specific IP address ranges for private addresses so that there are no conflicts with public IP addresses. From [RFC-1918](https://www.rfc-editor.org/info/rfc1918):

```
10.0.0.0        -   10.255.255.255  (10/8 prefix)
172.16.0.0      -   172.31.255.255  (172.16/12 prefix)
192.168.0.0     -   192.168.255.255 (192.168/16 prefix)
```

These are the IP addresses that you see in the logs. They are all private IP addresses. In the example, my service pod is getting a timeout from Mongo (which you might have guessed with the 27017 port). They are not really in the same private network as my pod is trying to reach out to Mongo Atlas which we don't even host in our infra. This is achieved through VPC peering which I'm not going to get into now.

When a device in the private network needs to make a request to the outside world, the router replaces the private address (which no one outside can understand) with its own public facing IP address. So as far as the outside world is concerned, the request is actually coming from the router. Apart from the IP address, the router may also replace the port number. Why? Let's say your laptop and phone, both on the private network, are making a request to public IP address from the same port. If the router uses the same port as the device, there's no way for it to distinguish replies coming for the laptop/phone from outside. Internally, the router uses something called the "NAT table" to figure out where to forward replies it's getting from the outside world. 


## Configuring my router through `http://192.168.0.1/`

My router is also obviously part of the network and has a private IP address which happens to be `192.168.0.1`. If I type this address in my browser, my router has enough handling to allow me to configure it after some auth. My router actually has a web server running to serve HTML pages that allow me to configure it, which I found to be pretty cool.
