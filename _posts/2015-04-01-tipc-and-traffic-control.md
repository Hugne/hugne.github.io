---
layout: post
title: "TIPC and traffic control"
description: ""
category: 
tags: []
---
{% include JB/setup %}


TIPC have four different messaging importance levels that can be used to
give precedence to certain services. The importance levels are only honored
for messaging going out on the same link, which means that low importance data
sent out at a high rate on one link may starve higher importance data sent out
on a different one.
This article describes how to use TC to set up a priority queueing discipline for TIPC
traffic on your bearer device that will prevent the starvation scenario.

By default, an ethernet device will have a pfifo qdisc which is used for any packets
sent out to that device.

> tc qdisc show  
> qdisc pfifo_fast 0: dev eth0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1

In order to prioritize TIPC traffic, we first need to add a new 4-band (one for each importance level) prio qdisc on eth0. This will create 4 classes that we can attach further qdiscs to.


> tc qdisc add dev eth0 root handle 1:0 prio bands 4  
> tc class show dev eth0  
> class prio 1:1 parent 1:  
> class prio 1:2 parent 1:  
> class prio 1:3 parent 1:  
> class prio 1:4 parent 1:  

Band 0 is the highest priority and 3 lowest.  

We now need to add another qdisc layer, one for each priority class (indicated by parent 1:x)  
> tc qdisc add dev eth0 parent 1:1 handle 10: pfifo_fast  
> tc qdisc add dev eth0 parent 1:2 handle 20: pfifo_fast  
> tc qdisc add dev eth0 parent 1:3 handle 30: pfifo_fast  
> tc qdisc add dev eth0 parent 1:4 handle 40: pfifo_fast  

In this example, i am using pfifo_fast but you could of course use another. Remember though
that if TIPC traffic is shaped or policed it will cause retransmissions and the overall performance
of the links will deteriorate.

Our TC classifier tree now looks like this:  
<PRE>
	           { 1:  }                     (Root)  
	          + + + +  
	  +-------+ | | +-------+  
	  |         | |         |                           
	  |      +--+ +--+      |                           
	  |      |       |      |                           
	  |      |       |      |                           
	  +      +       +      +                           
	{1:1}  {1:2}   {1:3}  {1:4}           (Prio classes)
	  +      +       +      +                           
	  |      |       |      |                           
	  +      +       +      +                           
	{10:}  {20:}   {30:}  {40:}           (pfifo_fast qdiscs)  
	  ^      ^       ^      ^                           
	  |      |       |      |                           
	  +      +       +      +                           
	  0      1       2      3             (Bands)      
</PRE>

Add filters on the root that maps the TIPC importance levels to the
correct flow: 

TIPC_LOW_IMPORTANCE      ---> 1:4  
TIPC_MEDIUM_IMPORTANCE   ---> 1:3  
TIPC_HIGH_IMPORTANCE     ---> 1:2  
TIPC_CRITICAL_IMPORTANCE ---> 1:1  

> tc filter add dev eth0 protocol tipc parent 1: prio 1 u32 match u8 0 1E at 0 flowid 1:4  
> tc filter add dev eth0 protocol tipc parent 1: prio 1 u32 match u8 2 1E at 0 flowid 1:3  
> tc filter add dev eth0 protocol tipc parent 1: prio 1 u32 match u8 4 1E at 0 flowid 1:2  
> tc filter add dev eth0 protocol tipc parent 1: prio 1 u32 match u8 6 1E at 0 flowid 1:1  

The u32 match filters deserves a little explanation. What we are saying here is that if the packet
type is TIPC, map the packet to either flow 4,3,2 or 1 based on the value of MSB bits 3-6 in word 0 (the TIPC importance level field).

This takes care of the priority starvation problem, but might cause packet reordering if there is a mix
of different importance levels for data sent out on a link. The TIPC stack should handle that though.



