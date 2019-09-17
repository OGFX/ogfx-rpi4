# OGFX

The Open (Guitar) Audio Effects Processor

# Rationale

Since the new Raspberry Pi4 dropped we were amazed to find that it finally offers a working USB3 host which can drive an USB class audio 2.0 device. This repository serves as a collection of knowledge and software snippets to setup an Rpi4 as an open source (guitar) audio effects processor using a USB audio class 2.0 audio interface.

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
* <code>realtime-privileges</code>
* <code>stress</code>
* <code>tmux</code>

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

This tunes the parameters introduced by the Low-Latency USB patch. Either unload and reload the snd-usb-audio module (using <code>rmmod</code> and <code>modprobe</code> or just reboot once more..

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

# Setup the CPU Frequency Scaling

We want to disable all CPU frequency scaling to make the system behave more predictably. Edit the file <code>/etc/rc.local</code> and add into it:

<pre>
for n in 0 1 2 3; do echo performance > /sys/devices/system/cpu/cpu"$n"/cpufreq/scaling_governor; done
</pre>

Make sure it has a <code>#!/bin/bash</code> shebang at the top of the file and that it is executable. Then enable the <code>rc-local</code> service:

<code>sudo systemctl enable rc-local</code>

and either reboot or start it with <code>systemctl start...</code>.

# Setup RT Permissions 

Add the <code>alarm</code> user to the <code>realtime</code> group using

<code>sudo gpasswd -a alarm realtime</code>

And relogin that user.

# Setup JACK2

Now it's time to plugin your USB audio class 2.0 device (for example the Focusrite 2i2 2nd gen, or Tascam 2x2, etc). It should appear in <code>/proc/asound/cards</code>

<pre>
$ cat /proc/asound/cards 
 0 [USB            ]: USB-Audio - Scarlett 2i2 USB
                      Focusrite Scarlett 2i2 USB at usb-0000:01:00.0-1.2, high speed
</pre>

Run

<code>jackd -R -P 80 -S -d alsa -d hw:USB -p 64 -n 2</code>

It should startup fine (some warnings about libFFADO are to be ignored, they do not matter). Now open another terminal on the Rpi4 and run

<code>stress -c 4</code>

Open yet another terminal and run

<code>sudo watch /opt/vc/bin/vcgencmd measure_temp</code>

Now leave this running for a couple of hours. Make sure that in the <code>jackd</code> terminal no XRUNs are reported. Also check the terminal running <code>vcgencmd</code> from time to time to check whether your Rpi4 has adequate cooling. Ideally you don't want the temperature maxing out around 60 degrees to have some headroom for really hot summer days (at 80 degrees the cpu gets throttled ruining our RT-performance).

Once you are sufficiently satisfied that the system is stable, kill all three processes.

# Setup SYSTEMD Lingering User Session

We want our effects processor to start up once the device is powered. And we also want to run everything as the <code>alarm</code> user. To start long running services and processes we can make use of <code>systemd</code>'s user service handling. To enable user services started independent of logging in manually we have to enable <code>lingering</code>:

<code>sudo loginctl enable-linger alarm</code>

Some versions of <code>systemd</code> shipped with Alarm are buggy and you have to run the following command manually:

<pre>
sudo mkdir -p /var/lib/systemd/linger
sudo touch /var/lib/systemd/linger/alarm
</pre>
