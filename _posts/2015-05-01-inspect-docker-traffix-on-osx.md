---
layout: post
title: Inspecting Docker Traffic on OS X
date: 2015-05-01 19:51:27
tags:
  - osx
  - docker
  - devops
  - wireshark
  - networking
---
Recently I was trying to debug an application I wrote that interfaces with Docker’s API. I realized a simple HTTP capture would be ideal for figuring out what was happening. My first instinct was to fire up [Wireshark](https://www.wireshark.org/), an amazing tool for doing network analysis. However, getting it (or other packet capture tools) to work with the virtual networking of VirtualBox or VMWare can be a huge pain; why bother dealing with that when we can capture the traffic right at the destination?

The de-fecto setup for Docker on OS X is to use [boot2docker](https://github.com/boot2docker/boot2docker) to initialize a small in-memory Linux VM. The VM is based on Tiny Core Linux and has a very small suite of tools available. Luckily, tcpdump is packaged for it and can write output compatible with Wireshark! The following commands install and start tcpdump within the boot2docker VM:

```bash
$ boot2docker ssh
$ tce-load -wi tcpdump.tcz
$ sudo tcpdump -s0 -w /Users/hexedpackets/docker.pcap -i eth1 '(((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

First we connect to the VM running in VirtualBox using SSH. tce-load then fetches the tcpdump package from the Internet and installs it locally. The last command, which must be run as root, creates the actual packet capture.

Breakingdown the tcpdump command, it will listen on the eth0 interface and write all output to /Users/hexedpackets/docker.pcap. This relies on a little boot2docker magic, since the OSX users folder is mounted into the VM by default. It is filtered to only capture packets containing data; the filter used here is taken directly from [the tcpdump manual](http://www.tcpdump.org/tcpdump_man.html).

With tcpdump running in the VM, I was able to make my Docker API calls and open the resulting capture with Wireshark for easy analysis. Bug spotted and squashed.
