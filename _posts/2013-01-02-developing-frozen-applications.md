---
layout: post
title: Developing Frozen Applications
date: 2013-01-02 19:51:27
tags:
  - python
  - cloud
  - windows
  - saas
  - deployment
---
Designing a SaaS application is easy. You code, you test, you push. Done. If you want to instantly see the changes, add the step of restarting the web server. Putting everything into the magical cloud may be controversial, but there’s no denying that its taken away most of the pain of software deployment. Working on a hybrid application made me realize how valuable this is.

The frozen part of the application was really rough to deploy. Even though I’m still only testing it locally, my list was much longer than the web part: code, test, freeze, wrap frozen files in an installer, kill the old process, install, run. It sucked.

I looked into making the application auto-update. Despite the great freezing tools available for Python, there isn’t much in the way of auto-updating. Someone on Stack Overflow suggested using bsdiff/bspatch to manually patch the binary. That also sucks.

Finally I stumbled across a Pycon 2012 talk about esky, a real auto-update framework. Now my frozen deployment goes code, test, freeze with esky, drop the zip in my file hosting. The application will check within a few hours for any updates, find the new version, download and install it, and restart itself. Simple and effective, the way it should be.

What I didn’t expect is how much this would change my development strategy. I didn’t realize before how afraid I was of small updates. I would wait until there was something large to do in the local app before working on it, and try to do a bunch at once. Now I can do a quick one-line logging change and deploy it just as rapidly as I would update my SaaS code. Its truly invigorating.
