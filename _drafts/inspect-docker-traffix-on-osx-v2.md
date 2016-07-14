---
layout: post
title: Inspecting Docker Traffic on OS X - Part 2
data: 2016-07-14
tags:
  - osx
  - docker
  - devops
  - wireshark
  - networking
---

[Previously I had posted about install tcpdump in boot2docker]({% post_url 2015-05-01-inspect-docker-traffix-on-osx %})

Dockerfile

```docker
FROM alpine
RUN apk add --no-cache tshark
ENTRYPOINT ["dumpcap", "-P", "-w", "-", "-i", "eth0", "-"]
```

Command:

```bash
wireshark -k -i <(docker run --name netcap --net=host tshark); docker rm -f netcap
```

Running `docker rm` instead of just passing the `--rm` flag is necessary since the container inside the pipe is noot actually stopped when wireshark stops.

Filters can be added after tshark. For example, to capture traffic just on port 8080 you can change the command to `docker run --name netcap --net=host tshark -f 'tcp port 8080'`.
