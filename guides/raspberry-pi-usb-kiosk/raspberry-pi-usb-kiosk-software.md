# Raspberry Pi USB Kiosk - Software Configuration

- [Back to main page](raspberry-pi-usb-kiosk.md)

## Operating System Choice

This guide is based around using the Legacy Debian Bullseye release, if you're planning on driving a standard screen, the latest available Raspberry Pi Legacy OS should suffice.

If, however, you are planning on running a custom resolution LCD panel (such as the [1440p 5.5" LCD](https://www.aliexpress.com/item/32830430329.html) I chose) that requires specific HDMI timings you will need an even older release:

* [2020-05-27-raspios-buster-armhf.zip](https://downloads.raspberrypi.com/raspios_armhf/images/raspios_armhf-2020-05-28/2020-05-27-raspios-buster-armhf.zip) (Official link from Raspberry Pi Foundation website).

The reason for needing this older release is the Pi Foundation changed drivers after this release and it broke some custom HDMI timings.

**Please note the above image does not support Pi Zero 2 W devices but supports every other Raspberry Pi model up to and including the Pi 4 B**

## Package Choices

We will be installing the following packages to support our endeavour:

* `surf` - A simple lightweight web browser that works well with HTML, CSS and JS
* `unclutter` - A simple tool that hides the mouse cursor when not in use
* `zram-config` - Quick setup tool for zram that provides _compressed_ RAM Disks for swap and folders like `/var/log/`
* `curl` - Used to check the web server is online before starting the browser
* `git` - Used to get the `zram-config` code


## Install Process

### Prep the SD card with an OS

Firstly, if you need to, download the above older release zip file. If you're unsure, it's safe to start with this.

Plug your SD card into your computer and run the Raspberry Pi imager.

Select your device and then either select `Raspberry Pi OS (Legacy)` if you're planning on using the latest legacy release, or scroll down and select `Use custom` and select the zip file you downloaded (ignore the part about .img, it takes .zip files too).

Click `Next` and when prompted to apply OS customisations, click `Edit Settings` and then:

* On the `General` tab, give your Pi a hostname, username, password, WiFi details etc
* On the `Services` tab, enable SSH and `Use password authentication` (you can add an SSH key post-boot)
* On the `Options` tab, tick `Eject media when finished` and untick `Enable telemetry`

I *heartily* recommend you configure wireless settings even if you (currently) don't have a wireless adapter - you may need to add (a USB) wireless adapter in future for debugging!

Click `Save`, then click `Yes`, and then `Yes` again to write the image.

Once this completes, eject your SD card, place it in your Pi and boot it up.

### First boot and configuring the kiosk

On first boot the Pi will reboot a couple of times before it can be used. Once it's done that, you can either SSH in (preferred) or just use a keyboard and mouse and launch the terminal.

#### Install required software

Firstly, we'll want to run the following to install some required software packages;

```bash
sudo apt install surf unclutter git curl
```

Once that has completed, run the following to install `zram-config`;

```bash
git clone https://github.com/ecdye/zram-config
sudo ./zram-config/install.bash
```

#### Configure services

###### Configure zram

Unless you have a Raspberry Pi with greater than 512 MB RAM, you will need to perform these changes.

Edit `/etc/ztab` file as root with your favourite editor, `nano` is a good choice for beginners but you can use your favourite editor.

```bash
sudo nano /etc/ztab
```

Change all `mem_limit` values to `50M` and `disk_size` to `100M` to be compatible with every Pi listed above.

If you are running a Pi 1, you will also need to change all references to `lzo-rle` to `lzo`.

Save the file and close it (press Ctrl+X, then Y, then Enter).

###### Disable the swap file

Edit `/etc/dphys-swapfile` like the above with `sudo nano /etc/dphys-swapfile`, change `CONF_SWAPSIZE=100` to `CONF_SWAPSIZE=0`, save and close.

The `dphys-swapfile` service *is* disabled later and this isn't strictly required, but it's a good safety margin to do both.

###### Disable the LightDM / LXDE panels

Edit `/etc/xdg/lxsession/LXDE-pi/autostart` with `sudo nano /etc/xdg/lxsession/LXDE-pi/autostart` and comment out every line (add a `# ` at the start), it should look like this:

```bash
# @lxpanel --profile LXDE-pi
# @pcmanfm --desktop --profile LXDE-pi
# @xscreensaver -no-splash
```

Then save and close the file.

###### Disable the piwiz popup (initial Pi setup screen)

```bash
sudo rm /etc/xdg/autostart/piwiz.desktop
```

### Disable unnecessary services and set up surf

#### Save memory by disabling some services

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

#### Save memory and speed up boot by tweaking config.txt

Edit `/boot/config.txt` with `sudo nano /boot/config.txt` and add the following lines at the bottom of the file underneath the `[all]` marker:

```bash
gpu_mem=16
disable_splash=1
boot_delay=0
dtoverlay=disable-bt
```

Then save and close the file.

#### Reboot and test surf manually

Run `sudo reboot` and then when it boots back up you'll just get a black screen in place of the desktop.

It may take some time, and you may need to wait a minute after the screen goes black (as it may not be apparent `lightdm` has finished loading), but when that's done run the following;

```bash
export DISPLAY=:0.0
surf -F http://www.google.com/
```

It may also take some time (if you're running an older Pi) for the web page to load but it should load.

Once you've confirmed this works, press `ctrl+c` to close the browser.

#### Add the surf launch script

Create a new file `/boot/kioskhost.txt` using `sudo` and place the following contents inside of it:

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

        surf -b -d -g -m -t -X -F -z "${KIOSKZOOM}" "${KIOSKHOST}"
done
```

Make the file execute it with `chmod +x launch-kiosk.sh`

And then finally edit your users crontab entry with `crontab -e` and add the following line:

```bash
@reboot sleep 60 && /home/pi/launch-kiosk.sh &
```
