# Raspberry Pi USB Kiosk - Locking it down

- [Back to main page](raspberry-pi-usb-kiosk.md)

## Finalising it

When you are happy everything is working as expected, it's time to enable the overlay (this makes the filesystem read-only).

To do this, run `sudo raspi-config` and choose one of these options (it moves depending on OS release):

* `Performance Options`->`Overlay File System`
* `Advanced Options`->`Overlay FS`

When prompted, enable the overlay file system but *do not* write-protect the boot partition. That way you can easily edit `/boot/kioskhost.txt` to change the URL and zoom of the web page it loads.

Please note, it may take some time for `update-initramfs` to complete (couple of minutes, especially on older devices) but when that's done, `sudo reboot` and enjoy!

If you need to undo this and make the OS writable again, just re-run `sudo raspi-config`, follow the above menu options but choose *not* to enable it and then reboot.