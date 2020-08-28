#### Preamble

I recently noticed that the CPU for my Ubuntu guest OS in my home ESXi server was using a heavy amount of CPU for both [Shinobi Video](https://shinobi.video/) (CCTV recording software) and [Plex Media Server](https://www.plex.tv/).

This was starting to frustrate me as I could see how much power my server was pulling via a UPS. After doing this my power usage dropped about 20%.

After some research, I decided to buy a used NVIDIA GTX 1060 to handle the heavy lifting for these services. This card seems to be a sweet-spot for price _vs_ hardware video encoding / decoding performance.

It took a couple of evenings to get it to work but I finally nailed the setup instructions. This guide would also benefit anyone looking to run any CUDA / OpenCV applications too.

*Note*: Plex requires a paid-for [Plex Pass](https://www.plex.tv/en-gb/plex-pass/) to enable hardware video encoding / decoding support.

#### Expectations

This guide was written with ESXi in mind. If you are **not** using ESXi you can _skip_ the first section on setting up the hardware and go straight to the OS install.

This guide also assumes the following;

1. That _if_ you are running [VMware ESXi](https://www.vmware.com/go/get-free-esxi)  you are running version 5.5 or later (I am using 6.7u3)
2. That you have a [recent NVIDIA card that supports NVENC and NVDEC](https://developer.nvidia.com/video-encode-decode-gpu-support-matrix) (those made mid-2012 onwards should work, mid-2016 onwards are better)
3. That you are planning to run a **new installation** of [Ubuntu 20.04.1 LTS (Focal Fossa)](https://releases.ubuntu.com/20.04/) (this guide _heavily_ assumes it will be a new installation)
4. That all commands are run as **root** - _yes_ I know this isn't best-practice however everything here is safe
5. That you are careful and copy / paste commands **one at a time**

#### Setting up the hardware (ESXi only)

To make this work with ESXi there are a couple of hoops you need to jump through first. This is namely because NVIDIA don't like you running consumer-grade GPU's in what they class as "server" environments.

##### Hardware requirements

* You will need both a CPU and Motherboard that supports a technology called [I/O MMU](https://en.wikipedia.org/wiki/X86_virtualization#I/O_MMU_virtualization_(AMD-Vi_and_Intel_VT-d)) (commonly referred to as Intel VT-d, AMD-Vi or simply "PCI passthrough").
* You will need an absolute minimum of 8 GB RAM and 2 CPU cores, I recommend at least 32 GB RAM and 8 CPU cores. 
* In your BIOS you will need to have enabled the Intel VT-d / AMD-Vi / PCI Passthrough option and any other "virtualization" options

##### ESXi host setup

* You must have enabled your GPU for passthrough ([here is a similar guide to this page that explains how to do it](https://elatov.github.io/2018/04/esxi-65-passthrough-video-card-to-plex-vm/))

##### Guest OS setup

* You must tick the `Reserve all guest memory (All locked)` option when creating the guest machine
* Double check there's a network adaptor added that is connected to your main guest virtual network switch
* You must add the GPU to the OS by going to the `Add other device -> PCI device` option and selecting the GPU
* If your GPU has a separate sound card device you must also add this as above
* You must edit your guest OS advanced settings under `VM Options -> Advanced -> Edit Configuration` and add the following keys / values:

| Key | Value |
| - | - |
| hypervisor.cpuid.v0 | FALSE |
| pciHole.start | 1200 |
| pciHole.end | 2200 |

#### Setting up the OS

##### Installation

This part assumes that you are starting with a barebones Ubuntu 20.04 server install image ([ubuntu-20.04.1-live-server-amd64.iso](https://releases.ubuntu.com/20.04/ubuntu-20.04.1-live-server-amd64.iso)) however the desktop image may work too.

1. Write the ISO to whatever you need to install it (burn it to DVD, write it to a USB thumbdrive with [Rufus](https://rufus.ie/), mount it in ESXi etc)
2. Start the installation process and go through it
3. When prompted I highly recommend enabling the SSH server
4. Reboot and log in

##### Set a static IP address (optional)

Ubuntu recently changed the way you manage system IP addresses, this is for anyone who struggles with that.

1. Run `ip a` and note the interface name that (hopefully) you can now see a DHCP-issued IP address (my interface was called `ens224` but yours may differ)
2. Run `editor /etc/netplan/00-installer-config.yaml` and change the file contents to something **similar** to the following;

```yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens224:
      addresses: [192.168.0.69/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8, 192.168.0.1]
  version: 2
```

Be **very** careful to get the indentation right, each indentation is exactly 2 spaces, no more, no less.

You will most likely want to change the IPs to suit your own network.

3. Run `netplan apply` to apply the IP address. If you are connected via SSH this will most likely now boot you out and you'll need to reconnect by the new address

#### Disable the built-in graphics driver and apply pending system updates

1. We **must** disable _nouveau_ (the built in graphics card driver) for this to work, you can do this by running this command;

```bash
echo -ne "blacklist nouveau\noptions nouveau modeset=0" > /etc/modprobe.d/blacklist-nouveau.conf
update-initramfs -u
```

2. Next we _should_ apply all pending system updates and reboot so we are up to date;

```bash
apt -y update && apt -y upgrade && reboot
```

#### Install software dependencies

1. We need to add a new repository to `apt` so that we can build `ffmpeg`, run the following commands;

```bash
add-apt-repository ppa:mc3man/focal6
apt update
apt install -y build-essential python3-pip ninja-build meson nasm mlocate frei0r-plugins-dev gnutls-dev libass-dev libmp3lame-dev libopencore-amrnb-dev libopencore-amrwb-dev libopenjp2-7-dev libopus-dev librtmp-dev librubberband-dev libsoxr-dev libspeex-dev libtheora-dev libvidstab-dev libvo-amrwbenc-dev libvorbis-dev libvpx-dev libwebp-dev libx264-dev libx265-dev libxvidcore-dev libzimg-dev
```

2. Next we need to install some NVIDIA-specific development headers;

```bash
mkdir -p /opt/nvidia
cd /opt/nvidia
git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
cd nv-codec-headers
make install
```

3. Now we need to download and install the NVIDIA driver and CUDA libraries (note this is a LARGE download and may take some time);

```bash
cd /opt/nvidia
wget https://developer.download.nvidia.com/compute/cuda/11.0.3/local_installers/cuda_11.0.3_450.51.06_linux.run
chmod +x cuda_11.0.3_450.51.06_linux.run
./cuda_11.0.3_450.51.06_linux.run
```

4. Next we will patch the NVIDIA driver to allow more than 3 concurrent hardware encoding / decoding jobs;

```bash
cd /opt/nvidia
git clone https://github.com/Snawoot/nvidia-patch.git
cd nvidia-patch
bash ./patch.sh
bash ./patch-fbc.sh
```

5. Now we need to add the tools the above installed to our system environment and reboot

```bash
sed -i 's/PATH="/PATH="\/usr\/local\/cuda-11.0\/bin:/g' /etc/environment
reboot
```

6. Once the system has rebooted, run `nvidia-smi` to see if the card is now ready for use, you should see output similar to the following;

```
root@video-server:~# nvidia-smi
Thu Aug 27 21:08:26 2020
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.51.06    Driver Version: 450.51.06    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  GeForce GTX 106...  Off  | 00000000:03:00.0 Off |                  N/A |
| 32%   50C    P5     9W / 120W |      0MiB /  6078MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

If at this point you get an error or don't see the above output, continuing with this guide will be pointless.

#### Build ffmpeg with NVENC support

1. First we need to build one external dependency (`vmaf`), this one is quick;

```bash
mkdir ~/src
cd ~/src
git clone https://github.com/Netflix/vmaf.git
cd vmaf
pip3 install cython numpy
make
make install
ldconfig
```

2. Now we download `ffmpeg` sources, patch them to work with the above driver and compile successfully;

```bash
cd ~/src
git clone https://git.ffmpeg.org/ffmpeg.git
cd ffmpeg
sed -i 's/compute_30,code=sm_30/compute_60,code=sm_60/g' configure
./configure --enable-gpl --enable-version3 --enable-static --disable-debug --disable-ffplay --disable-indev=sndio --disable-outdev=sndio --enable-fontconfig --enable-frei0r --enable-gnutls --enable-gray --enable-libfribidi --enable-libass --enable-libvmaf --enable-libfreetype --enable-libmp3lame --enable-libopencore-amrnb --enable-libopencore-amrwb --enable-libopenjpeg --enable-librubberband --enable-librtmp --enable-libsoxr --enable-libspeex --enable-libvorbis --enable-libopus --enable-libtheora --enable-libvidstab --enable-libvo-amrwbenc --enable-libvpx --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxvid --enable-libzimg --enable-cuda-sdk --enable-cuvid --enable-nvenc --enable-nonfree --enable-libnpp --extra-cflags=-I/usr/local/cuda/include --extra-ldflags=-L/usr/local/cuda/lib64
make -j 10
make install
```

Depending on your CPU speed, this can take a long time to compile (mine takes about 5 minutes on a quad core Xeon)

3. Once that's done and installed you can verify your `ffmpeg` install works by running the following command;

```bash
ffmpeg -encoders | grep nv
```

If this has worked successfully you should see a few lines that look like this at the bottom of the output;

```
...
 V..... h264_nvenc           NVIDIA NVENC H.264 encoder (codec h264)
 V..... nvenc                NVIDIA NVENC H.264 encoder (codec h264)
 V..... nvenc_h264           NVIDIA NVENC H.264 encoder (codec h264)
 V..... nvenc_hevc           NVIDIA NVENC hevc encoder (codec hevc)
 V..... hevc_nvenc           NVIDIA NVENC hevc encoder (codec hevc)
```

#### Install and configure Shinobi Video for hardware decoding (optional)

1. Install [Shinobi Video](https://shinobi.video/docs/start) by doing the following;

```bash
cd ~/src
git clone https://gitlab.com/Shinobi-Systems/Shinobi.git
cd Shinobi
chmod +x INSTALL/ubuntu.sh
INSTALL/ubuntu.sh
```

2. Follow the on-screen guide, when prompted you probably want to create a database server
3. Shinobi's installer should detect `ffmpeg` is installed and use it automatically
4. Once installed, run `pm2 save` to make it start on boot
5. When adding a camera follow the `How to Decode an RTSP Stream with NVIDIA Graphics Cards` part of [this](https://shinobi.video/docs/gpu) guide

#### Install and patch Plex Media Server for sequential hardware video decoding / encoding (optional)

*Reminder*: Plex requires a paid-for [Plex Pass](https://www.plex.tv/en-gb/plex-pass/) to enable hardware video encoding / decoding support.

1. Install Plex Media Server by running the following;

```bash
echo deb https://downloads.plex.tv/repo/deb public main | sudo tee /etc/apt/sources.list.d/plexmediaserver.list
curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo apt-key add -
apt update
apt install plexmediaserver
```

2. Plex will automatically start at this point, stop it with the following command;

```bash
service plexmediaserver stop
```

3. Patch Plex to fix an issue with concurrent hardware transcoding jobs;

```bash
cd ~/src
git clone https://github.com/revr3nd/plex-nvdec.git
cd plex-nvdec
bash plex-nvdec-patch.sh
service plexmediaserver start
```

Et voilà!
