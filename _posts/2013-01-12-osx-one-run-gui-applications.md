---
layout: post
title: OS X One-Run GUI Applications
date: 2013-01-12 19:51:27
tags:
  - desktop development
  - python
  - gui
  - osx
---
File this one under “Things Apple Would Not Approve Of.”

Attempting to create an installable .app bundle for OS X based on a Python application is surprisingly hard. For all the focus on making things easy for consumers, Apple has made things really hard for developers.

The problem I was having is that I needed a few variables to be input by the user once. Really easy on Windows - just pop it into the installer, the user has to enter the values before the application is installed.

The equivalent of an msi or exe installer for OS X is a pkg, so I tried that. There’s no way to add in any custom forms other than writing your own plugin in Objective C to integrate with the installer. That’s shit. A quick search didn’t turn up any existing plugins, either.

I resigned myself to making a GUI element. The application runs completely in the background, so I have to add in a the component solely for the purpose of collecting these three strings. Ok, fine, whatever. Except my first choice, Tkinter, didn’t work - the dialog appears but can’t get a keyboard focus so no text can be entered. My next choice, wxPython, works fine outside of a bundled app. Once it is compiled with py2app OS X starts throwing a cryptic error about not using a Python framework.

After a lot of debugging, I realized the error is causing by my plist entry stating that the application runs in the background. Ok, update that to False. Now I have a working dialog to collect the input and after the initial one-time setup, there is an icon in the dock that never goes away and doesn’t have any application to show.

My solution is to dynamically update the plist file within my own application bundle. Super hacky, but it seems like the best (and maybe only) way to get the behavior I want. On first load after an install or update, the application will check for a config file and show an text entry dialog if needed. It then updates the plist file to mark it as a background only application and restarts. Once it has restarted, it will be invisible to the user.
