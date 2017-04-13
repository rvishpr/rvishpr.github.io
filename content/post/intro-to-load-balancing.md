---
author: ""
categories: ["systems", "networking"]
date: "2017-04-12T15:05:32-07:00"
description: ""
featured: ""
featuredalt: ""
featuredpath: ""
title: "Intro to Load Balancing"
linktitle: "Intro to Load Balancing"
---

Load balancing at its simplest is defined as distributing load across a group of servers, so they appear to the clients/users as one big server. Load balancing does more than simply distribute load. It helps with redundancy. It helps with scaling the service provided, without the users having to know about it. And more. I’ll attempt to capture some of it here. This can serve as an overview to load balancing.

First, some definitions.

**Scaling**. This is more important now than ever considering the size of the internet. This refers to the ability to increase/decrease the capability to handle load (by processing power and other resources) according to demand, without any disruption of the service provided.

**Redundancy/High-Availability**. This refers to having more resources to take over in case of failures, to provide seamless service to the users. It can be provided by adding servers that wait on the sidelines to take over when needed. Or, it can be done by designing a stateless system where there are multiple servers running, and one can simply take over from another in case of failure.

**Client**. The entity which requests a resource/service.

**Server**. The entity which provides the resource/service.

People often wonder what makes a site like Facebook tick. Why is it hard to design a site like that? Anyone with enough web-development chops can put together a website. Why can’t we?

What makes it difficult is the ability to scale to millions, why billions, of users, while providing uninterrupted service under any type of load. An analogy is that anyone can make a dozen cookies, and make them taste better than the ones bought at a store. But can they make thousands of them at a time? Millions of them? That then brings into the question of how to source the ingredients, how to make the dough for such numbers of them, how to bake them all, and how to store and deliver them. That is what makes the problem interesting. Sites like Facebook are like that. While anyone can create the next great social networking site, handling the sheer scale of the internet is what makes a site tick or fail and crumble like a thousand others.

Not long ago, creating a website meant buying servers and hosting the website on them. But now, tools like AWS, Google Cloud, etc make such an effort unnecessary. But it is still important to understand how to design a scalable system.

Anyone wanting to create a website can buy a domain and a few servers and put together a website with a system that can then be used. But, when it starts to become popular and gets more traffic, it will need better and more powerful servers to handle the load. Replacing the old ones would mean downtime and more expense every time it came to replacing them. But, if the servers are designed to be run in parallel, we can simply place another server in front of them to distribute the traffic across the servers now running in parallel. We can increase resources by simply adding more servers to the pool now, without having to bring down or replace the entire system. This new server in front then becomes the load balancer.

This changes the question now to – how to distribute the load across the servers? First let’s see what types of load balancing are out there.

## Types of Load balancing

### Round Robin DNS

Load Balancing comes in several types. The simples of them is Round Robin DNS. This is simply using the fact whenever a client needs to access a website, it needs to resolve the domain name (such as http://www.google.com) into reachable IP addresses. In simple terms, the DNS server which will return the IP addresses can decide in which order to return them in. For Client A, it can return 1, 2, 3 and for Client B, it can return 2, 3, 1. The clients can choose the first address on the list and hence this can function as a simple load balancer.

```
$nslookup www.wordpress.com
Server: 10.0.2.3
Address: 10.0.2.3#53

Non-authoritative answer:
www.wordpress.com canonical name = lb.wordpress.com.
Name: lb.wordpress.com
Address: 192.0.78.13
Name: lb.wordpress.com
Address: 192.0.78.12
```

This can also be leveraged to implement client-side load balancing where the client can try each IP in turn in case of failures. This provides some level of redundancy too.

This scheme, however simple, has problems. What happens when one of the servers on the list is down and the DNS administrators for the domain fail to update it in time? Also, how do we account for DNS caching behavior? Despite such problems, this is popular for its simplicity and can, even now, help in geographically distributing the load across data centers.

Inside a data center, however, other schemes must be used.

### L3/L4 Load balancing

This is server side load balancing done at the network and transport layers respectively of the networking stack.

L3 load balancing is when routing decisions are taken based on the packet’s source and destination IP addresses. L4 Load balancing is when the source and destination IP address and port information is used to route traffic across server instances. For example, HTTP(80) traffic and HTTPS(443) traffic can be redirected differently from, say SIP traffic on port 5060.

Servers can be chosen in different ways. Round robin is a simple way here. Or, the IP+Port information can be hashed to decide the server. This can at least ensure some level of predictability in allocating clients to servers. More on this later.

Now the load balancer can act like a proxy, where it terminates TCP connection and establishes one with the server chosen or do NAT (Network Address Translation) where it simply forwards requests to the chosen server and receive the responses and sends them back to the client. In these cases, the IP address of the packet is changed accordingly. Or it can run in a mode called Direct Server Return (DSR), where the load balancer simply chooses the server and forwards the request after changing only the MAC address in the packet, and the server then directly responds to the client. In this case, the load balancer and the servers are all assigned the same Virtual IP Address(VIP), but only the load balancer will respond to ARP requests. This can be done by assigning the VIP in the loopback interface in the servers.

This type of load balancing is relatively inexpensive as compared to L7, although the differences are diminishing with all the improvements in technology.

### L7 Load balancing

This operates at the highest layer(s) of the networking stack. Here decisions on how to route the traffic are taken based on the application payload. For example, for HTTP requests, the request (URL or cookie for example) can be parsed and based on the headers/payload routing can be performed. This server should terminate the TCP connection from the client and either re-originate one with the server chosen or re-use from a pool of connections maintained.

More functionality can be added here, such as redirecting requests to the right server based on the request, such as for images or videos or quote requests, etc.

### More to load balancing

Load balancers are simply servers that sit in front of a cluster of different servers to distribute the load. This shouldn’t become a ‘single point of failure’ (SPOF). The load balancers need to be run in high availability mode with more servers to take over in case of failures, while remaining in sync with the active server.

Load balancing also provides high availability to the servers in the cluster. When the servers can run in parallel, where many run the same backend, failure of one means the requests can simply be redirected to others. Also scaling here becomes easier, since we simply need to add more servers in case of an increase in load. But, the servers here need to be stateless, so it wouldn’t matter which server serves a request. This is not always the case.

#### Persistence/Stickiness

A lot of requests need to preserve state, such as a shopping cart, or a wish list for examples.

Some implementations use ‘stickiness’ as a factor when distributing load, where requests from the same client go to the same server. This can be done by storing information in cookies, or URL rewriting(this may not be secure, since malicious clients can change URLS and send requests), or decide based on the client’s IP+Port information. Usually some combination of these in addition to others is adopted to implement this stickiness. This should still lead to the question of how HA is now possible if a single server is attached to a session. What happens when this server fails? The session information could then be lost.

In this case the state needs to be in a persistent store, so failure of a server doesn’t mean loss of data. Some in memory databases such as redis or memcached can be leveraged for this.  This is not trivial since database read and write operations are not usually expensive and even for in-memory databases, we need to replicate/backup the data often to prevent the database from becoming a SPOF.

Usually, just one form of load balancing is not enough. L3 load balancers sit in front of a cluster of L4 balancers which in turn distribute to a cluster of L7 balancers which eventually distribute traffic to the servers. For an example of this, we can look at Facebook’s implementation which is presented in a nice talk at Building a billion user load balancer.