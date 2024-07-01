---
title: "Automated Proxmox VE Installation with netboot.xyz"
date: 2024-07-01
tags: [proxmox]
categories: [Homelab]
description: Proxmox VE has great support for automated (i.e unattended) installation. If the hardware supports PXE, it's very easy to set up completely hands-off (no USB stick needed!) installs using netboot.xyz. There are a couple of gotchas that aren't well documented, but they are quite easy to fix.
---
<!--more-->
## Introduction
Proxmox VE has great support for automated (i.e unattended) installation. If the hardware supports PXE, it's very easy to set up completely hands-off (no USB stick needed!) installs using netboot.xyz. There are a couple of gotchas that aren't well documented, but they are quite easy to fix.

## Initial Setup
For simplicity, I used [LinuxServer.io's netboot.xyz image](https://github.com/linuxserver/docker-netbootxyz/). It includes all the netboot.xyz boot files and menus, a TFTP server, and even the [webapp](https://github.com/netbootxyz/webapp) which allows you to easily modify the menus and download and serve assets locally. The rest of this post assumes you have a tftp and http server with the necessary files up and running with the necessary router / DHCP settings configured.

**N.B.:** if your TFTP server is behind a firewall, you'll likely need to fix the data transfer ports using the `PORT_RANGE` environment variable since they are different from the data request port, 69 (see the container documentation for more information).

## Automated Proxmox VE ISO
Proxmox has [pretty good documentation](https://pve.proxmox.com/wiki/Automated_Installation) covering the installation answer file format and how to prepare the ISO. If you're not setting this up on a host with Proxmox VE, you'll need to add [their repository](https://pve.proxmox.com/wiki/Package_Repositories) in order to download the `proxmox-auto-install-assistant` package. While it is possible to embed the answer file into the ISO directly, it's much more flexible to serve it over HTTP because you can change it without having to re-build the ISO. Per [the documentation](https://pve.proxmox.com/wiki/Automated_Installation#Answer_Fetched_via_HTTP), once you have the stock ISO downloaded and answer file prepared you'll want to validate the answer file with:
```
proxmox-auto-install-assistant validate-answer answer.toml
```
then create the ISO with:
```
proxmox-auto-install-assistant prepare-iso /path/to/source.iso --fetch-from http --url "http://netboot_assets.mydomain.tld/answer.toml"
```

You can also use https and even pin the certificate, though I'd recommend against the latter as they tend to expire quickly.

## Serving the ISO and answer file over HTTP
Both the prepared ISO and answer file must be accessible from your HTTP server. For the LSIO image, this means placing it wherever you've mapped `/assets` in the container to. Keep in mind that by default, netboot.xyz downloads ISO's, kernels, etc from some deeply nested path because they're hosted on Github. For convenience, I put the prepared ISO in the same path that the default ISO would be (e.g. `/assets/asset-mirror/releases/download/8.2-1-613c19ff/proxmox-automated.iso`), but I put the answer file at the root (`/assets/answer.toml`). You can see what the default ISO path is either by using the webapp to download the default ISO, or by checking the `/config/menus/proxmox.ipxe` menu file.

### Gotcha #1
The first tricky bit is serving the answer file. With the ISO prepared to get the answer over HTTP, the Proxmox installer actually sends an HTTP POST request to the specified URL with a JSON body payload containing a bunch of information about the host where installation is running. The idea is to enable dynamically generated answers based on the host and its hardware using something like [natankeddem/autopve](https://github.com/natankeddem/autopve). I can see how this might be useful at enterprise scale, but for me, a static file that I can edit every now and then is good enough.

The problem is that the nginx server in the LSIO container will not serve static files in response to a POST request: it returns an HTTP 405 error. This can easily be solved by adding `error_page 405 =200 $uri;` to the bottom of the `location /` block in the nginx config file (`/config/nginx/site-confs/default`). Generally, reverse proxies will not interfere with HTTP responses based on method, but some may restrict particular methods by default. If you use a reverse proxy, you can check that everything is working by running `curl -vX POST http://netboot_assetsmydomain.tld/answer.toml`, and making sure it returns your answer file correctly.

## Setting up netboot.xyz
All of the netboot.xyz menus are set up as [iPXE](https://ipxe.org/) script files, so adding an automated Proxmox VE install is as easy as adding a new menu option. The webapp makes this very easy, or you can edit the file directly (`/config/menus/proxmox.ipxe`).

The first thing to do is create a variable for the name of the proxmox iso:
```
set os Proxmox
set iso proxmox.iso # <---
menu ${os}
```

Then add an entry for our automated install:
```
item --gap ${os} VE
item pve-normal ${space} ${os} VE 8.2-1
item pve-automated ${space} ${os} VE 8.2-1 (Automated) # <---
item pve-text ${space} ${os} VE 8.2-1 (Text)
item pve-debug ${space} ${os} VE 8.2-1 (Debug)
```

### Gotcha #2
It took me a while to figure out why the installer wouldn't take the answer file. It turns out the Proxmox VE installer actually triggers the automation based on a kernel parameter, rather than the files that get added to the ISO. (I found this by digging through the ISO's grub config files). Add the following above the `:pve-debug` section:

```
:pve-automated
set params splash=silent proxmox-start-auto-installer
set iso proxmox-automated.iso
goto boot-pve
```

Finally, the last step is to update the `boot-pve` section:

```
:boot-pve
set kernel_url ${local_endpoint}/asset-mirror/releases/download/8.2-1-613c19ff/
set proxmox_version 8.2-1
imgfree
kernel ${kernel_url}vmlinuz vga=791 video=vesafb:ywrap,mtrr ramdisk_size=16777216 rw quiet ${params} initrd=initrd.magic ${cmdline}
initrd ${kernel_url}initrd
initrd ${kernel_url}${iso} /proxmox.iso
boot
```

You'll notice I added a `local_endpoint` variable. I put the definition in my `boot.cfg` file, pointing to the URL that gets reverse proxied to the container's asset HTTP server. This way I can use it for other locally served assets as well. You could also just put the full URL in this menu file if you wanted to. Note that the kernel (vmlinuz) and initial RAM disk (initrd) also point to the same host as the ISO, so those must be available locally as well. I just used the webapp to download them, and it automatically places the files in that nested directory. This is why I suggested doing the same for the prepared ISO in the beginning.

If you want the full menu file for reference, you can download it [here](files/proxmox.ipxe).

## That's It!
With the two small catches resolved, you should now be able to netboot into a fully automatic Proxmox VE installation. If your hardware has some kind of remote KVM solution like iDRAC, vPro, or a PiKVM, you can even do a "hands-free" bare metal install completely remotely! I absolutely love netboot.xyz. It's one of the cornerstones of my homelab setup, and the first thing I get set up when I'm updating hardware.