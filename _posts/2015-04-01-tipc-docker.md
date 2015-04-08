---
layout: post
title: "TIPC and docker"
description: ""
category: 
tags: [docker, tipc]
---
{% include JB/setup %}

As of kernel 4.0, TIPC supports network namespaces and can be used as an IPC bus for docker containers.
This is just a series of commands that spawns 4 opensuse docker containers and enables TIPC
communication across these.

First, start the containers:  
<PRE>
linux-q8su:~ # docker run -d --cap-add all -v /root/docker_mnt/:/mnt opensuse /bin/sh
804737043c041ff659e01f91b8b3d28d177e1f9c7974a8f7f44439b856182087
linux-q8su:~ # docker run -id --cap-add all -v /root/docker_mnt/:/mnt opensuse /bin/sh
668c363b73bb72f4618e167716eddeaea6aecb36afe804d73015d32599ad0f26
linux-q8su:~ # docker run -id --cap-add all -v /root/docker_mnt/:/mnt opensuse /bin/sh
6eee345202cf241fc67856588207ed834bb48f5037f5da7229559afe71297d62
linux-q8su:~ # docker run -id --cap-add all -v /root/docker_mnt/:/mnt opensuse /bin/sh
61eeeeb0094228470e735f9ec9fdac9d7bac7d90b0ae0ade029b9edc745aa14f
</PRE>
We need the TIPC userspace tools to configure TIPC, the docker containers usually don`t include this so i have placed it in /root/docker_mnt on the docker host and bind mount that directory in the containers.
A better approach is to boot up the container base image, install the tipcutils package and commit the
change to a new image.

Set a TIPC address and enable eth0 as bearer:  
<PRE>
linux-q8su:~ # docker exec 61eeeeb00942 /mnt/tipc-config -a=1.1.1
linux-q8su:~ # docker exec 6eee345202cf /mnt/tipc-config -a=1.1.2
linux-q8su:~ # docker exec 668c363b73bb /mnt/tipc-config -a=1.1.3
linux-q8su:~ # docker exec f635a4af155e /mnt/tipc-config -a=1.1.4
linux-q8su:~ # docker exec 61eeeeb00942 /mnt/tipc-config -be=eth:eth0
linux-q8su:~ # docker exec 6eee345202cf /mnt/tipc-config -be=eth:eth0
linux-q8su:~ # docker exec 668c363b73bb /mnt/tipc-config -be=eth:eth0
linux-q8su:~ # docker exec f635a4af155e /mnt/tipc-config -be=eth:eth0
</PRE>

The containers can now speak TIPC to eachother:  
<PRE>
linux-q8su:~ # docker exec 61eeeeb00942 /mnt/tipc-config -l
Links:
broadcast-link: up
1.1.1:eth0-1.1.2:eth0: up
1.1.1:eth0-1.1.3:eth0: up
1.1.1:eth0-1.1.4:eth0: up
</PRE>

The manual address/bearer configuration is a little bulky, and we see a need for an automatic
configuration utility.


