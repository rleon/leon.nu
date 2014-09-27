+++
date = "2014-03-09"
draft = false
title = "hotplug external monitor"
slug = "hotplug-external-monitor"
desription = "very short howto connect external monitor"
tags = ["linux", "tips", "script"]
+++

If you are happy user of fully featured DE ([desktop environment](http://en.wikipedia.org/wiki/Desktop_environment)) like Gnome/KDE/Unity, you
will find this short howto totaly useless.

In case, you are using WM ([window manager](http://en.wikipedia.org/wiki/Window_manager)), you will be required to implement hotplug of external 
monitors by yourself.

This mini-howto can help you to do it:

1. Create new udev rule in `/etc/udev/rules.d`.
```udev
ACTION== "change", KERNEL=="card0", SUBSYSTEM=="drm", RUN+="/usr/local/bin/set-display.sh"
```
2. Place [set-display.sh](https://github.com/rleon/utils) script in `/usr/local/bin` folder.
3. Set execute bits: `sudo chown a+x /usr/local/bin/set-display.sh`
