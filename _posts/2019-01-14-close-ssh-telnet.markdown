---
layout: post
title:  "How to quit console session established by SSH and telnet"
date:   2019-01-14 21:45:00 +0800
categories: telnet, ssh
---

# How to quit console session established by SSH and telnet

We can connect to console port of various devices via console server. Console server usually provides two methods for connection:
* SSH
* Telnet

To quit console session established via SSH and telnet, simply run `exit` or `quit` won't work. Run these commands usually logout from the remote device rather than terminating the current SSH/Telnet connection.

## Quit SSH session

Press `Ctrl + Z`, then type `exit`

## Quit telnet session

Press `Ctrl + ]`, then type `quit`
