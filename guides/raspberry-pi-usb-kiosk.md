- [Preamble](#preamble)
- [Expectations](#expectations)
- [Hardware Requirements](#hardware-requirements)
- [Software Requirements](#software-requirements)
- [Install Process](#install-process)


## Preamble

I found a need recently to render a web page full screen (aka Kiosk Mode) on an LCD panel that was built / modded into a PC case. 

Although the solution below will cater for **most** web pages, for me this screen would need to display various system metrics (usages, temperatures etc) on a high resolution screen and update regularly.

I chose an [Aida64 RemoteSensor dashboard](https://www.aida64.com/products/features/external-display-support) because it would generate a simple HTML page with JavaScript polling for updates.

The solution to render this would need to tick a few boxes:

1. Must be able to drive high resolution displays (1080p or greater) using HDMI
2. Must function independently to the main system (ie. not take one of the video cards display outputs)
3. Must function independently of any network environment (ie. still work when moved away from a wireless network)
4. Must be resilient to sudden power off events (read only filesystem)
5. Must run for hours / days on end without issue

It seemed a Raspberry Pi would be the best fit for the above and it just so happens that some models support a lovely feature called USB OTG that can be configured to be a USB RNDIS Ethernet Gadget (like a crossover network cable between two computers).

## Expectations

In following this guide, it is expected that you;

* Are comfortable working with Raspberry Pi's and Linux
* Are comfortable SSH'ing into a Linux computer
* Are comfortable editing shell scripts with an editor that honours line endings
* Are comfortable writing out an SD card using the [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
* Have a suitable USB card reader for the above
* Have the correct hardware below

## Hardware Requirements

### Choosing the correct Raspberry Pi

Due to some hardware limitations, some Pi's can only drive up to 1080p screens. Additionally, your Pi should have at least 512 MB RAM and ideally have a heatsink.

I recommend the following, based on ease of setup:

* For display resolutions up to 1080p: Raspberry Pi Zero 2 W
* For display resolutions above 1080p: Raspberry Pi 4 Model B

### Hardware

* Raspberry Pi Zero
* Raspberry Pi Zero W
* Raspberry Pi Zero WH
* Raspberry Pi Zero 2 W
* Raspberry Pi 1 Model A (special cable required)
* Raspberry Pi 1 Model A+ (special cable required)
* Raspberry Pi 3 Model A+ (special cable required)
* Raspberry Pi 4 Model B

### SD Card

With exception to the Raspberry Pi 4 Model B, all of the above *must* boot from SD card, so you'll need a suitable SD card (16 GB or greater, class 10 or better).

Although we're going to almost completely stop writes to the SD card, it's best to buy a good card to be safe.

Here are some recommendations on reliable SD card brands and product series that are designed for endurance / significant amounts of written data:

* Western Digital Purple
* Samsung Pro Endurance
* Sandisk Extreme Pro

### Cables

The Pi Zero's (including the 2 W) can all be powered and send network traffic over one USB cable (even directly from a motherboard USB header which is what I did)..

![USB Micro A to USB Header](https://m.media-amazon.com/images/I/51FLI9+cSRL._AC_UF1000,1000_QL80_.jpg)

The Pi 4 Model B can also be powered and network over its Type C connector so you'll need something like this (or Type C USB header, either the USB 2.0 4 or 5 pin or full Type-C will work):

![USB Type C to USB Type A](https://www.blackbox.co.uk/_AppData/Images/Full/10747.jpg)

The Pi 1 Models A, A+ and the Pi 3 Model A+ however have more peculiar requirements, you'll need to find a suitable USB Type A Male to USB Type A (or USB header) cable like this:

![USB Type A to USB Type A](https://90a1c75758623581b3f8-5c119c3de181c9857fcb2784776b17ef.ssl.cf2.rackcdn.com/432837_406645_01_front_comping.jpg)

And in the cases of connecting that cable plus the Pi 4 Model B to a motherboard header, something like this:

![USB Type A Female to USB Header](https://media.startech.com/cms/products/main/usbmbadapt.main.jpg)

## Software Requirements

### Operating System

This guide is based around using the Legacy Debian Bullseye release, if you're planning on driving a standard screen, the latest available Raspberry Pi Legacy OS should suffice.

If however you are planning on running a custom resolution LCD panel (such as the [1440p 5.5" LCD](https://www.aliexpress.com/item/32830430329.html) I chose) that requires specific HDMI timings you will need an even older release:

[2020-05-27-raspios-buster-armhf.zip](https://downloads.raspberrypi.com/raspios_armhf/images/raspios_armhf-2020-05-28/2020-05-27-raspios-buster-armhf.zip) (Official link from Raspberry Pi Foundation website).

The reason for needing this older release is the Pi Foundation changed drivers after this release and it broke some custom HDMI timings.

**Please note the above image does not support Pi Zero 2 W devices but supports every other Pi listed in the hardware section above.**

### Packages

We will be installing the following packages to support our endeavour:

* `surf` - A simple lightweight web browser that works well with HTML, CSS and JS
* `unclutter` - A simple tool that hides the mouse cursor when not in use
* `zram-config` - Quick setup tool for zram that provides _compressed_ RAM Disks for swap and folders like `/var/log/`


## Install Process

### Prep the SD card with an OS

Firstly, if you need to, download the above older release zip file. If you're unsure, it's safe to start with this.

Plug your SD card into your computer and run the Raspberry Pi imager.

Select your device and then either select `Raspberry Pi OS (Legacy)` if you're planning on using the latest legacy release, or scroll down and select `Use custom` and select the zip file you downloaded (ignore the part about .img, it takes .zip files too).

Click Next and when prompted to apply OS customisations, click `Edit Settings` and then:

* On the `General` tab, give your Pi a hostname, username, password, WiFi details etc
* On the `Services` tab, enable SSH and `Use password authentication` (you can add an SSH key post-boot)
* On the `Options` tab, tick `Eject media when finished` and untick `Enable telemetry`

Click Save, then click Yes, and then Yes again to write the image.

Once this completes, eject your SD card, place it in your Pi and boot it up.

### Boot it up and start configuring the kiosk

On first boot the Pi will reboot a couple of times before it can be used. Once it's done that, you can either SSH in (preferred) or just use a keyboard and mouse and launch the terminal.

#### Install required software

Firstly, we'll want to run the following;

```bash
sudo apt install surf unclutter git curl
```

Once that has completed, run the following;

```bash
git clone https://github.com/ecdye/zram-config
sudo ./zram-config/install.bash
```

#### Configure supporting services

###### Configure zram

Edit `/etc/ztab` file as root with your favourite editor, `nano` is a good choice for beginners, `vi` is my preference :)

```bash
sudo nano /etc/ztab
```

Change `mem_limit` to `50M` and `disk_size` to `100M` to be compatible with every Pi listed above. You can increase these values on systems with greater than 1 GB RAM.

If you are running a Pi 1, you may also need to change `lzo-rle` references to `lzo` as well.

Save the file and close it (Ctrl+X then Y then Enter).

###### Disable the swap file

Edit `/etc/dphys-swapfile` like the above with `sudo`, change `CONF_SWAPSIZE=100` to `CONF_SWAPSIZE=0`, save and close.

###### Disable the LightDM / LXDE panels

Edit `/etc/xdg/lxsession/LXDE-pi/autostart` with `sudo` and comment out every line (add a `# ` at the start), it should look like this:

```bash
# @lxpanel --profile LXDE-pi
# @pcmanfm --desktop --profile LXDE-pi
# @xscreensaver -no-splash
```

Finally disable the `piwiz` tool prompting to set up the Pi:

```bash
sudo rm /etc/xdg/autostart/piwiz.desktop
```

#### Disable unnecessary services

Run the following (ignore any errors about missing services);

```bash
sudo systemctl disable bluetooth
sudo systemctl disable avahi-daemon
sudo systemctl disable triggerhappy
sudo systemctl disable cups
sudo systemctl disable cups-browsed
sudo systemctl disable alsa-state
sudo systemctl disable hciuart
sudo systemctl disable dphys-swapfile
sudo systemctl disable plymouth
```

#### Reboot and test surf manually

Run `sudo reboot` and then when it boots back up you'll just get a black screen.

When you see that, run the following;

```bash
export DISPLAY=:0.0
surf -F http://www.google.com/
```

#### Add the surf launch script

Firstly, create a new file `/boot/kioskhost.txt` using `sudo` and place the following contents inside of it:

```
https://myhost:8080/path
1.0
```

The first line is the URL to load and the second number is the zoom level (1.0 = 100% zoom, 0.75 = 75% etc).

Save and exit that file, next create a new file in your user dir (`/home/pi` for most users) called `launch-kiosk.sh` with the following contents;

```bash
#!/usr/bin/env bash

KIOSKDATA=$(cat /boot/kioskhost.txt 2>/dev/null || cat /boot/firmware/kioskhost.txt 2>/dev/null)
KIOSKHOST=$(echo "${KIOSKDATA}" | head -n1)
KIOSKZOOM=$(echo "${KIOSKDATA}" | tail -n1)

if [[ "${KIOSKHOST}" == "" ]]; then echo "kioskhost.txt is missing from /boot, aborting"; exit 1; fi

while true; do
        sleep 2
        curl -svko /dev/null -f "${KIOSKHOST}" && echo "Kiosk web server is ready" || continue

        export DISPLAY=:0.0
        xset -dpms
        xset s off
        killall unclutter > /dev/null 2>&1
        unclutter -idle 0 &

        surf -b -d -F -z "${KIOSKZOOM}" "${KIOSKHOST}"
done
```

Make the file execute it with `chmod +x launch-kiosk.sh`

And then finally edit your users crontab entry with `crontab -e` and add the following line:

```bash
@reboot sleep 60 && /home/pi/launch-kiosk.sh &
```

### Enable USB RNDIS Ethernet Gadget mode

Amend or create the following files as before with `sudo` to enable this feature as well as speed up boot.

#### /boot/config.txt

Scroll down to the `[All]` section and add the following right at the bottom;

```bash
# Enable USB Gadget Ethernet
dtoverlay=dwc2,dr_mode=peripheral

# Speed up boot
disable_splash=1
boot_delay=0
dtoverlay=disable-bt

# Force HDMI output
hdmi_force_hotplug=1
```

Make any other changes to this file that you need to, such as HDMI tweaks etc.

#### /boot/cmdline.txt

Look for the phrase `rootwait` and then add `modules-load=dwc2,g_ether` before it (with spaces either side).

It should look *similar* to this (there will likely be other entries there):

```bash
boot=overlay console=serial0,115200 console=tty1 root=PARTUUID=c09dba46-02 rootfstype=ext4 fsck.repair=yes modules-load=dwc2,g_ether rootwait plymouth.ignore-serial-consoles ...
```

#### /etc/network/interfaces.d/usb0

This one will require **ONE** change from the below - **DO NOT ADD BOTH**.

###### Static USB IP

Change the IPs to suit your desired network. Bear in mind that it will *not* be the same network as your home LAN.

```bash
auto usb0
allow-hotplug usb0
iface usb0 inet static
  address 10.255.255.3
  netmask 255.255.255.0
  gateway 10.255.255.1
  dns-nameservers 10.255.255.1
```

###### Dynamic USB IP

```bash
auto usb0
allow-hotplug usb0
iface usb0 inet dhcp
```

#### /root/usb.sh

**Only required for Pi 4 Model B**

These steps require `sudo`. Create `/root/usb.sh` with the following contents:

```bash
#!/bin/bash
cd /sys/kernel/config/usb_gadget/
mkdir -p pi4
cd pi4
echo 0x1d6b > idVendor # Linux Foundation
echo 0x0104 > idProduct # Multifunction Composite Gadget
echo 0x0100 > bcdDevice # v1.0.0
echo 0x0200 > bcdUSB # USB2
echo 0xEF > bDeviceClass
echo 0x02 > bDeviceSubClass
echo 0x01 > bDeviceProtocol
mkdir -p strings/0x409
echo "fedcba9876543211" > strings/0x409/serialnumber
echo "Ben Hardill" > strings/0x409/manufacturer
echo "PI4 USB Device" > strings/0x409/product
mkdir -p configs/c.1/strings/0x409
echo "Config 1: ECM network" > configs/c.1/strings/0x409/configuration
echo 250 > configs/c.1/MaxPower
# Add functions here
# see gadget configurations below
# End functions
mkdir -p functions/ecm.usb0
HOST="00:dc:c8:f7:75:14" # "HostPC"
SELF="00:dd:dc:eb:6d:a1" # "BadUSB"
echo $HOST > functions/ecm.usb0/host_addr
echo $SELF > functions/ecm.usb0/dev_addr
ln -s functions/ecm.usb0 configs/c.1/
udevadm settle -t 5 || :
ls /sys/class/udc > UDC
ifup usb0
```

You'll then need to `sudo chmod +x /root/usb.sh` and then finally enable this at start up;

Edit `/etc/rc.local` and add `/root/usb.sh &` immediately before the last line `exit 0`.

That's it! Reboot and test the interface comes up.

### Finalising it

When you are happy everything is working as expected, it's time to enable the overlay (this makes the filesystem read-only).

To do this, run `sudo raspi-config` and choose one of these options (it moves depending on OS release):

* `Performance Options`->`Overlay File System`
* `Advanced Options`->`Overlay FS`

When prompted, enable the overlay file system but *do not* write-protect the boot partition. It may take some time for `update-initramfs` to complete (couple of minutes) but when that's done, `sudo reboot` and enjoy!

To undo the above, just re-run `sudo raspi-config`, follow the above menu options but choose *not* to enable it (and then reboot).

## Credits

* zram-config script - https://github.com/ecdye/zram-config/tree/main
* Pi 4 Model B usb.h script - https://www.hardill.me.uk/wordpress/2019/11/02/pi4-usb-c-gadget/
* Surf launch script - myself