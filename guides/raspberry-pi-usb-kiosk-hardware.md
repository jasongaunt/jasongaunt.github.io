# Raspberry Pi USB Kiosk - Hardware Requirements

- [Back to main page](raspberry-pi-usb-kiosk.md)

### Choosing the correct Raspberry Pi

Due to some hardware limitations, some Raspberry Pi models can only drive up to 1080p screens. I recommend one of the following, based on ease of setup and performance:

* For display resolutions up to 1080p: Raspberry Pi Zero 2 W
* For display resolutions above 1080p: Raspberry Pi 4 Model B

This guide will work for *all* Raspberry Pi models, but *only the below support USB-OTG*:

* Raspberry Pi Zero
* Raspberry Pi Zero W
* Raspberry Pi Zero WH
* Raspberry Pi Zero 2 W
* Raspberry Pi 1 Model A (special cable required)
* Raspberry Pi 1 Model A+ (special cable required)
* Raspberry Pi 3 Model A+ (special cable required)
* Raspberry Pi 4 Model B

If you can live with networking them "traditionally" then you may use another model (and skip the gadget mode setup).

Although not strictly required unless you overclock, your Raspberry Pi should have a heatsink. Keeping it cool can make it run faster.

Additionally, models with at least 512 MB RAM will perform best, but 256 MB models will work too (dependent on the web page you load).

### Choosing a good SD Card

With exception to the Raspberry Pi 4 Model B, all of the above *must* boot from SD card, so you'll need a suitable SD card (16 GB or greater, class 10 or better).

Although we're going to almost completely stop writes to the SD card, it's best to buy a good card to be safe. You want this to last for years, right?

Here are some recommendations on reliable SD card brands and product series that are designed for endurance / significant amounts of written data:

* Western Digital Purple
* Samsung Pro Endurance
* Sandisk Extreme Pro

### USB cable choices for USB-OTG

#### Pi Zero and derivatives

The Pi Zero's (including the 2 W) can all be powered and send network traffic over one USB cable (even directly from a motherboard USB header which is what I did)..

![USB Micro A to USB Header](/assets/USB%20Micro%20A%20to%20USB%20Header.jpg)

#### Pi 4 Model B

The Pi 4 Model B can also be powered and network over its Type C connector so you'll need something like this (or Type C USB header, either the USB 2.0 4 or 5 pin or full Type-C will work):

![USB Type C to USB Type A](/assets/USB%20Type%20C%20to%20USB%20Type%20A.jpg)

#### Pi 1 / 3 Models A and A+

The Pi 1 Models A, A+ and the Pi 3 Model A+ however have more peculiar requirements, you'll need to find a suitable USB Type A Male to USB Type A (or USB header) cable like this:

![USB Type A Male to USB Type A Male](/assets/USB%20Type%20A%20Male%20to%20USB%20Type%20A%20Male.jpg)

#### Direct motherboard connections

In the cases of connecting that cable plus the Pi 4 Model B to a motherboard header, you may need something like this:

![USB Type A Female to USB Header](/assets/USB%20Type%20A%20Female%20to%20USB%20Header.jpg)
