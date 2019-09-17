# OGFX

The Open (Guitar) Audio Effects Processor

# Rationale

Since the new Raspberry Pi4 dropped we were amazed to find that it finally offers a working USB3 host which can drive an USB class audio 2.0 device. This repository serves as a collection of knowledge and software snippets to setup an Rpi4 as an open source (guitar) audio effects processor.

The base distribution is Archlinux-Arm (alarm) but the methods should be, in principle, applicable to any distribution for the Rpi4

# Preliminaries

Install the alarm version for raspberrypi4 on your Rpi4 and then follow the instructions below. 

You will need to install the following packages:

* <code>base-devel</code>
* <code>git</code>
* <code>sudo</code>
* <code>htop</code>
* <code>jack2</code>
* <code>rtirq</code>

Add the <code>alarm</code> user to the <cide>wheel</code> group. Also change the <code>root</code> user's password to something of your liking.

# Kernel Setup

We need to apply two patches to the stock linux-raspberrypi4 kernel:

* RT-PREEMPT (full)
* Low-Latency USB 

We have prepared a PKGBUILD that has the required patches. It's found in the <code>src/linux-raspberrypi4</code> folder. It should be fairly easy to build this kernel with <code>makepkg</code>. Read up on the makepkg documentation if you run into trouble, but in principle it should be a three step process to install and boot into this kernel:

* <code>makepkg -s</code>
* <code>sudo pacman -U linux-raspberrypi4-4.19.69-2-armv7h.pkg.tar.xz</code>
* <code>sudo reboot</code>

Once rebooted checkout the output of e.g. <code>htop</code> (install it if necessary). Hit <code>F6</code>, select <code>PRIORITY</code> and you should now see some threads called <code>irq/53-PCIe PME</code>, and similar. These are the kernel IRQ handler threads which we want from our RT-PREEMPT system and which we will tune a little in the next step. But before that create a file calle <code>/etc/modprobe.d/00-snd-usb.conf</code> with the following content:

<pre>
options snd-usb-audio max_packs=4 max_packs_hs=4 max_urbs=12 sync_urbs=4 max_queue=18
</pre>

This tunes the parameters introduced by the Low-Latency USB patch.

# Setup RTIRQ

Edit <code>/etc/rtirq.conf</code> to look like this:

<pre>
RTIRQ_NAME_LIST="usb i8042"
RTIRQ_PRIO_HIGH=90
RTIRQ_PRIO_DECR=5
RTIRQ_PRIO_LOW=51
RTIRQ_RESET_ALL=0
RTIRQ_NON_THREADED="rtc snd"
</pre>

And enable the service with

<code>sudo systemctl enable rtirq</code>

and either reboot or start the service now (with <code>systemctl start</code>). Check with <code>htop</code> whether the <code>irq/54-xhci_hcd</code> kernel thread has priority <code>-91</code>. If so <code>rtirq</code> is doing its thing correctly.
