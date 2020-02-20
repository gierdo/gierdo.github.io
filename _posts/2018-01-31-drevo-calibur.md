---
title: The drevo calibur 71-key BT keyboard with linux
---

I bought a drevo calibur, a nice 71-key (60%) bluetooth-capable keyboard with
mechanical switches.

As it is a 60% keyboard, it doesn't have any FX (F1, F2, F3, ..) keys, they are
supposed to be pressed by pressing FN + X ( FN + 1, FN + 2, ...).

However, this didn't work with linux (4.15.0 on (k)ubuntu 17.10) out of the box.

The hid-apple driver is used to run the keyboard. The default mapping of the
apple keyboard doesn't allow for pressing the FX buttons, but propper function
keys (brightness-down, brigthntess-up, ...), as provided by the original apple
keyboard.

The default hid_apple fn-mode has to be changed in order to map the FX keys as
intended. Assuming your kernel has hid-apple built as a module:

Create a conf file at `/etc/modprobe.d/hid-apple.conf`:

```bash
options hid_apple fnmode=2
```

Reload the hid_apple module or reboot, and your're done.


```bash
sudo rmmod hid_apple sudo modprobe hid_apple
```
