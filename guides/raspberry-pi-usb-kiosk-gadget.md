# Raspberry Pi USB Kiosk - USB RNDIS Gadget Activation Process

- [Back to main page](raspberry-pi-usb-kiosk.md)

## Enable USB RNDIS Ethernet Gadget mode

Amend or create the following files as before with `sudo` to enable this feature as well as speed up boot.

#### /boot/config.txt

Scroll down to the `[All]` section and add the following right at the bottom;

```bash
# Enable USB Gadget Ethernet
dtoverlay=dwc2,dr_mode=peripheral

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

Create this file and add **ONE** change from the below - **DO NOT ADD BOTH**.

###### Static USB IP

```bash
auto usb0
allow-hotplug usb0
iface usb0 inet static
  address 10.255.255.3
  netmask 255.255.255.0
  gateway 10.255.255.1
  dns-nameservers 10.255.255.1
```

Change the IPs to suit your desired network. Bear in mind that it will *not* be the same network as your home LAN.

###### Dynamic USB IP

```bash
auto usb0
allow-hotplug usb0
iface usb0 inet dhcp
```

#### /root/usb.sh

**This file is only required for Pi 4 Model B**

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

## Windows RNDIS Gadget Driver

Although the above should work fine for Linux and OSX, Windows by default will *believe* the newly attached device is a COM (aka serial) Port and not do anything with it.

You may need to download and replace the driver with this: [RNDIS Ethernet Gadget Driver.zip](/assets/RNDIS%20Ethernet%20Gadget%20Driver.zip)

You can also, if desired, choose to share your Ethernet / WiFi connection on your computer with this new interface once installed. 

If you do that, you can either opt for [Dynamic USB IP](#dynamic-usb-ip) above, or if you prefer static, use the following config:

```bash
auto usb0
allow-hotplug usb0
iface usb0 inet static
  address 192.168.137.2
  netmask 255.255.255.0
  gateway 192.168.137.1
  dns-nameservers 192.168.137.1
```