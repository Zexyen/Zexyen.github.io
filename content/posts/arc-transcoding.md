+++
title = "Passing through Intel Arc GPU to LXCs for Transcoding"
date = "{{ .Date }}"
author = "Zexyen"
cover = ""
tags = ["proxmox","plex","transcoding"]
keywords = ["proxmox", "plex","transcoding","Intel"]
description = "How being an early adopter sucks"
showFullContent = false
readingTime = true
hideComments = false
color = "orange" #color from the theme settings
+++

# Foreword

I have a Proxmox server where all of my homelab applications lives in, It's super complicated with a bunch of random things just strung together in a semi-coherent fashion, but it works. I'm currently in the middle of doing a TV-esque broadcasting solution using [ersatzTV](https://ersatztv.org/), which is super cool if you want a continous TV Channel running in Plex. The issue with this however is that it's a continously running FFMPEG stream, which if you scale it up and add more channels to it, will absolutely hammer your CPU. So I decided to buy a GPU & pass it through to the docker LXC containing it.

## Part 1: Mistakes were made

![Mistakes were made](/images/huge_mistake.jpg)
The proxmox server is running a Ryzen 5600G which was originally purchased with the idea of using the intergrated iGPU for Plex transcodes. Obviously that idea didn't pan out because plex's AMD GPU Support is severely lacking and it never properly worked, and since I never transcoded anyways because 90% of my streams were Direct Play I decided to just not bother dealing with it.

In my infinite wisdom I thought that running an Intel Arc GPU Would be infinitely easier to deal with & cheaper than a NVIDIA GPU. _I Was Wrong._

### Part 2: Living with your Mistakes

I quickly figured out that the Arc GPU Support isn't in Linux native till the 6.1 Kernel, which Proxmox thankfully has as a opt-in option for their kernel. You can opt-in to 6.2 by going to your proxmox host's Shell, and run the following commands:

{{< code language="Terminal" title="PVE Kernel Update" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
apt update
apt install pve-kernel-6.2
reboot
{{< /code >}}

and now were in 6.2!

Next we are going to want to figure out the `/dev/dri` numbers for the GPU, which can be found by running `ls -l /dev/dri` on your Shell. Below you can see that I got `226, 0` & `226, 128` respectively on mine.

{{< code language="terminal" title="/dev/dri output" id="2" expand="Hide" collapse="Hide" isCollapsed="false">}}
root@pve:~# ls -l /dev/dri
total 0
drwxr-xr-x 2 root root         80 Jun  3 11:41 by-path
crw-rw---- 1 root video  226,   0 Jun  3 11:41 card0
crw-rw---- 1 root render 226, 128 Jun  3 11:41 renderD128
{{< /code >}}

Now that we have those numbers we can go ahead and pass it through to the LXC Containers, which you can do by editing `/etc/pve/lxc/<id>.conf` and appending the following:

```

lxc.cgroup2.devices.allow: c <numbers found earlier> rwm
lxc.cgroup2.devices.allow: c <numbers found earlier> rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

It should come out like this example config:
{{< code language="crlf" title="Plex LXC Config" id="3" expand="Hide" collapse="Hide" isCollapsed="false">}}
arch: amd64
cores: 12
features: mount=nfs;cifs,nesting=1
hostname: plex
memory: 3050
mp0: /mnt/pve/plex,mp=/media/plex
net0: name=eth0,bridge=vmbr0,gw=10.10.10.1,type=veth
onboot: 1
ostype: ubuntu
rootfs: local-lvm:vm-110-disk-0,size=151G
swap: 1536
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
{{< /code >}}

And once you have that edited you can go ahead and restart the container.

After Restarting you're going to want to make sure that the User Plex runs under has access to your GPU by running `gpasswd -a plex render` & `gpasswd -a plex video` __In the container__. You'll know the GPU was properly passthrough (passthroughed?) when you run `hwinfo --display` (which can be installed with `apt-get install hwinfo` ) and see this as the output:

{{< code language="terminal" title="hwinfo readout" id="4" expand="Hide" collapse="Hide" isCollapsed="false">}}
root@plex:~# hwinfo --display
06: PCI 300.0: 0300 VGA compatible controller (VGA)
  [Created at pci.386]
  Unique ID: svHJ.28Q6RjwKYw7
  Parent ID: GA8e.mr2N3fBJq5F
  SysFS ID: /devices/pci0000:00/0000:00:01.1/0000:01:00.0/0000:02:01.0/0000:03:00.0
  SysFS BusID: 0000:03:00.0
  Hardware Class: graphics card
  Model: "Intel VGA compatible controller"
  Vendor: pci 0x8086 "Intel Corporation"
  Device: pci 0x56a5
  SubVendor: pci 0x1849 "ASRock Incorporation"
  SubDevice: pci 0x6004
  Revision: 0x05
  Driver: "i915"
  Driver Modules: "i915"
  Memory Range: 0xfa000000-0xfaffffff (rw,non-prefetchable)
  Memory Range: 0x7c00000000-0x7dffffffff (ro,non-prefetchable)
  Memory Range: 0xfb000000-0xfb1fffff (ro,non-prefetchable,disabled)
  IRQ: 119 (259700 events)
  Module Alias: "pci:v00008086d000056A5sv00001849sd00006004bc03sc00i00"
  Config Status: cfg=new, avail=yes, need=no, active=unknown
  Attached to: #44 (PCI bridge)

Primary display adapter: #6
{{< /code >}}

#### Part 3: How I learned to appreciate what I have

Now in Plex you can go ahead and run any video through the WebUI, which should force it to transcode and you can confirm in the dashboard if it's using your GPU or CPU.

![Plex Dashboard HW Transcode](https://images.zexyen.com/Images/2023/06-03/ZAzVA6.png)

If it's listing the transcode as `Transcode (hw)` then congrats! It's working.
