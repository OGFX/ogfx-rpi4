# OGFX

The Open (Guitar) Audio Effects Processor

# Rationale

Since the new Raspberry Pi4 dropped I was amazed to find that it finally offers a working USB3 host which can drive an USB class audio 2.0 device. This repository serves as a collection of knowledge and software snippets to setup an Rpi4 as an open source (guitar) audio effects processor.

The base distribution is Archlinux-Arm (alarm) but the methods should be, in principle, applicable to any distribution for the Rpi4

# Kernel Setup

We need to apply two patches to the stock linux-raspberrypi4 kernel:

* RT-PREEMPT (full)
* Low-Latency USB 

We have prepared a PKGBUILD that has the required patches. It's found in the src/linux-raspberrypi4 folder.
