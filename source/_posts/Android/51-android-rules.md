---
title: 51-android-rules
permalink: 51-android-rules
categories:
  - Android
tags:
  - android
  - rules
date: 2020-02-01 11:37:02
---

```xml
SUBSYSTEM=="usb" ENV{DEVTYPE}=="usb_device", MODE="0666"
```