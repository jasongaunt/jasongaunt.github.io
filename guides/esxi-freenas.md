One problem I've had to solve recently is none of my ESXi servers will mount a FreeNAS iSCSI datastore on boot. When I attempt to mount it manually in the vSphere client it insists on formatting the datastore. This is clearly unacceptable.

To make matters worse, one of my ESXi servers has no permanent storage / datastore, it boots from USB and has no storage of its own, so generally any customisations to the OS are lost on reboot.

I've done some research and combined other users knowledge to solve this problem, which I will share with you now. This guide assumes…

- You have [enabled shell access](https://kb.vmware.com/s/article/2004746)
- Know how to SSH into your ESXi host
- Know how to edit files with [vi](https://www.howtogeek.com/102468/a-beginners-guide-to-editing-text-files-with-vi/)
- That you are using ESXi 5.5+
- That all your virtual machines / guests are stopped

## **Step 1 - Add shell script to mount all available datastores**

This shell script was written by myself and works perfectly, it triggers a rescan of all datastores, mounts them and then sets the load balancing even round-robin just in case you have multiple paths. This is also safe for single paths.

If at any point you feel uncomfortable or have messed up, you can quit _vi_ without writing a file by pressing ESC three times and typing in :q! and hitting enter. The q is case-sensitive.

Firstly, run the following command to open _vi_ with a new file;

```bash
vi /bootbank/esxi-mount-all-iscsi.sh
```

Once in _vi_ disable automatic indentation of pasted text by typing in `:set noai` and hitting enter.

Next, press lower-case `i` to switch to _insert mode_ and then copy and paste in the following;

```bash
#!/bin/sh

# Script to detect and mount all available iSCSI targets
# Written by JBG 20180816

# Rescan adapters
for adapter in `esxcli storage core adapter list | grep iscsi | awk '{print $1}'`; do
    esxcli storage core adapter rescan --adapter $adapter
done

# Mount volumes
for vol in `esxcfg-volume -l | grep UUID | egrep -oE '[0-9a-f]{8}-[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{12}'`; do
    esxcfg-volume -M $vol
done

# Enable load balancing
for device in `esxcfg-scsidevs -c | awk '{print $1}' | grep naa. `; do
    esxcli storage nmp device set --device $device --psp VMW_PSP_RR
    esxcli storage nmp psp roundrobin deviceconfig set --type=iops --iops=1 --device=$device
done
```

Then press `ESC` to exit _insert mode_ and return to _command mode_, when you have done that type in `:wq` and hit enter to Write the file and Quit.

## **Step 2 - Make the shell script executable**

This is an easy one, just run the following command;

```bash
chmod +x /bootbank/esxi-mount-all-iscsi.sh
```

## **Step 3 - Call the script on boot**

Please be careful with this next step, if you damage this file you may render your ESXi unbootable. As stated before, if you mess up, press `ESC` three times and type in `:q!` and hit enter to quit without saving the file.

Edit the _local.sh_ file with _vi_..

```bash
vi /etc/rc.local.d/local.sh
```

Disable automatic indentation again by typing in `:set noai` and hitting `enter`, then using your arrow keys, move the cursor to the line _above_ the text `exit 0` and press `i` to go into _insert mode_ once again.

Next, copy / paste in the following code…

```bash
# Spawn background thread to mount iSCSI
nohup /bootbank/esxi-mount-all-iscsi.sh &
```

… then add a newline at the end for neatness and then hit `ESC` to return to _command mode_. Assuming your _local.sh_ file has no other customisations, it should look exactly like this;

```bash
#!/bin/sh

# local configuration options
# Note: modify at your own risk! If you do/use anything in this
# script that is not part of a stable API (relying on files to be in
# specific places, specific tools, specific output, etc) there is a
# possibility you will end up with a broken system after patching or
# upgrading. Changes are not supported unless under direction of
# VMware support.
# Spawn background thread to mount iSCSI
nohup /bootbank/esxi-mount-all-iscsi.sh &
exit 0
```

When you are happy it looks correct, press `ESC` again for good measure and then type in `:wq` followed by `enter` to save and exit the file.

## **Step 4 - Reboot**

Reboot your server with the reboot command (or do it via the vSphere client) if it is safe to do so. You should now find your datastore is now ready for use when the server has booted up :)
