---
layout: post
title: "TIPC in Python"
description: ""
category: 
tags: [python tipc]
---
{% include JB/setup %}

A few days ago, i hacked together a small example how to do basic IPC using TIPC in a Python environment.
I didn't bother with an elaborate service addressing scheme, it's just using the local PID to
index services and the TIPC names are hardcoded.

You register a callback function as:

<PRE>
from tipc import tipc

def callback(msg):
	print msg

tipc.setup(callback)
</PRE>

Sending a message to another python process:

<PRE>
tipc.send("Hello unicast", os.getpid())
</PRE>

Or, send to everyone:
<PRE>
tipc.sendall("Hello multicast")
</PRE>

Full example can be found here:
<https://github.com/Hugne/tipc_python/tree/master/>
