---
layout: post
title: Sniffing network traffic remotely
date: 2016-07-14
tags:
  - docker
  - devops
  - wireshark
  - networking
---

This is a follow-up to a [previous post about install tcpdump in boot2docker]({% post_url 2015-05-01-inspect-docker-traffix-on-osx %}). On Tiny Core Linux tcpdump currently has a broken dependancy chain - netfilter fails to install with a 404 for the specific version. Since writing that post, I've been using tshark over SSH to debug remote hosts and realized the same technique can apply to a boot2docker VM.

This assumes you already have both wireshark and docker installed, so if you don't go do that. I'll wait.

# Create the image

An Alpine Linux image with tshark installed comes in at ~92mb. Its a little bit hefty for a tool like this; although tcpdump is much smaller, I prefer the expressiveness of the capture filters for tshark.

Save the contents below as a Dockerfile and run `docker build -t tshark .` from that directory.

```docker
FROM alpine
RUN apk add --no-cache tshark
ENTRYPOINT ["dumpcap", "-P", "-w", "-", "-i", "eth0", "-"]
```

# Run it

Now to start capturing all of traffic on your Docker VM, run:

```bash
wireshark -k -i <(docker run --name netcap --net=host tshark); docker rm -f netcap
```

Filters can be added after tshark. For example, to capture traffic just on port 8080 you can change the command to `docker run --name netcap --net=host tshark -f 'tcp port 8080'`. Killing wireshark will also stop the capture and delete the container.

# What it does

The tshark container is created with the host network. By default a separate bridged interface is used, which gives containers access to each other but not the host. `dumpcap` is run on the default eth0 interface and outputs all traffic in pcap format to stdout. The stdout/stderr for the container is passed through an anonymous FIFO pipe to wireshark, allowing you to monitor everything live.

Running `docker rm` instead of just passing the `--rm` flag is necessary since the container inside the pipe is not actually stopped when wireshark stops.
