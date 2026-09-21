# Raspberry Pi USB Kiosk

- [Preamble](#preamble)
- [Expectations](#expectations)
- [Hardware Requirements](#hardware-requirements)
- [Software Configuration](#software-configuration)
- [USB RNDIS Gadget Activation Process](#usb-rndis-gadget-activation-process)
- [Locking It Down](#locking-it-down)
- [Credits](#credits)

## Preamble

Recently I found a need to render a web page full screen (aka Kiosk Mode) on a HDMI LCD panel that was built / modded into a PC case. 
This screen would need to display various system metrics (usages, temperatures etc) on a high resolution screen and update regularly.

The solution to render this would need to tick a few boxes:

1. Must be able to drive high resolution displays (1080p or greater) using HDMI
2. Must not take one of the video cards display outputs
3. Must still work when moving location (ie. LAN party)
4. Must be resilient to sudden power off events (read only filesystem)
5. Must run for hours / days on end without issue

It seemed a Raspberry Pi would be the best fit for the above, but there's an added bonus; some Raspberry Pi models support a feature called *USB-OTG* that allows the Pi to function as a USB device rather than USB host.

Once in device mode, you can then change this to function as a USB RNDIS Ethernet Gadget (like a crossover network cable between two computers).

For the software, I chose an [Aida64 RemoteSensor dashboard](https://www.aida64.com/products/features/external-display-support) because it would generate a simple HTML page with a JavaScript EventStream polling for updates.

## Expectations

In following this guide, it is expected that you;

* Are comfortable working with Raspberry Pi's and Linux
* Are comfortable SSH'ing into a Linux computer
* Are comfortable editing shell scripts with an editor that honours line endings
* Are comfortable writing out an SD card using the [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
* Have a suitable USB card reader for the above
* Have the correct hardware below

## Hardware Requirements

See the [Hardware Requirements Page](raspberry-pi-usb-kiosk-hardware.md)

## Software Configuration

See the [Software Configuration Page](raspberry-pi-usb-kiosk-software.md)

## USB RNDIS Gadget Activation Process

See the [USB RNDIS Gadget Activation Process Page](raspberry-pi-usb-kiosk-gadget.md)

## Locking It Down

See the [Locking It Down Page](raspberry-pi-usb-kiosk-overlay.md)

## Credits

* zram-config script - [https://github.com/ecdye/zram-config/tree/main](https://github.com/ecdye/zram-config/tree/main)
* Pi 4 Model B usb.h script - [https://www.hardill.me.uk/wordpress/2019/11/02/pi4-usb-c-gadget/](https://www.hardill.me.uk/wordpress/2019/11/02/pi4-usb-c-gadget/)
* Surf launch script - myself