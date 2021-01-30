# Setup PXE boot environment

## Install

We will use docker containers instead of polluting the host.

We will leverage https://netboot.xyz/ for the tftp / bootloader / ipxe stuff

https://hub.docker.com/r/linuxserver/netbootxyz

Since we already have a DHCP server in the network (ie: my ISP provided home router), and it is unable to provide the extra TFTP options, we will setup dnsmasq in "proxy mode" so it will provide just the required TFTP options to clients without messing with the options provided by the home router DHCP

https://hub.docker.com/r/ferrarimarco/pxe/

Combine both in a docker-composer.yaml

```
version: '3.8'
services:
  dnsmasq:
    image: strm/dnsmasq
    network_mode: host
    restart: unless-stopped
    container_name: dnsmasq
    volumes:
      - ./dnsmasq/dnsmasq.conf:/etc/dnsmasq.conf
    ports:
      - 67:67/udp
    #cap_add:
    #  - NET_ADMIN

  netbootxyz:
    image: ghcr.io/linuxserver/netbootxyz
    network_mode: host
    restart: unless-stopped
    container_name: netbootxyz
    environment:
      - PUID=1000
      - PGID=1000
      - MENU_VERSION=1.9.9 #optional
      - PORT_RANGE=30000:30010 #optional
    volumes:
      - ./netbootxyz/config:/config
      - ./netbootxyz/assets:/assets #optional
    ports:
      - 3000:3000
      - 69:69/udp
      - 8080:80 #optional
    #depends_on:
    #   - dnsmasq
```

For this to work the containers should be started using --net=host

Configure the dnsmasq.conf with

```
# Disable DNS
port=0

# Verbose DHCP logging
log-dhcp

# Disable re-use of the DHCP servername and filename fields as extra
# option space. That's to avoid confusing some old or broken DHCP clients.
dhcp-no-override

# Answer DHCP discovery requests coming in over the ip range of the host network
dhcp-range=YOURSERVERNETWORK,proxy

# Identify the type of PXE client, and set the boot filename accordingly
dhcp-match=set:bios,60,PXEClient:Arch:00000
dhcp-boot=tag:bios,netboot.xyz.kpxe,,YOURSERVERIP
dhcp-match=set:efi32,60,PXEClient:Arch:00002
dhcp-boot=tag:efi32,netboot.xyz.efi,,YOURSERVERIP
dhcp-match=set:efi32-1,60,PXEClient:Arch:00006
dhcp-boot=tag:efi32-1,netboot.xyz.efi,,YOURSERVERIP
dhcp-match=set:efi64,60,PXEClient:Arch:00007
dhcp-boot=tag:efi64,netboot.xyz.efi,,YOURSERVERIP
dhcp-match=set:efi64-1,60,PXEClient:Arch:00008
dhcp-boot=tag:efi64-1,netboot.xyz.efi,,YOURSERVERIP
dhcp-match=set:efi64-2,60,PXEClient:Arch:00009
dhcp-boot=tag:efi64-2,netboot.xyz.efi,,YOURSERVERIP

# Boot the relevant PXE image if above 
pxe-service=x86PC,"Run netboot.xyz, BIOS mode",netboot.xyz-undionly.kpxe
pxe-service=X86-64_EFI, "Run netboot.xyz, UEFI mode", netboot.xyz.efi
pxe-service=BC_EFI, "Run netboot.xyz, UEFI mode", netboot.xyz.efi

```
Where YOURSERVERNETWORK might be something like 192.168.1.0
Where YOURSERVERIP might be something like 192.168.1.105


Ensure firewall is allowing the connections:
```
sudo ufw allow proto udp from any to any port 67
sudo ufw allow proto udp from any to any port 69
sudo ufw allow proto udp from any to any port 4011
sudo ufw allow proto tcp from any to any port 80
```




Use a preseed.cfg file from URL, for example hosted at Github:

https://www.wcooke.org/2020/08/debian-preseed-pxe-boot-install/preseed-buster-desktop.txt


https://gist.github.com/CalvinHartwell/f2d7f5dedbfee2d7d47c583539a10859#file-ubuntu-18-04-lts-preseed-cfg-L200

The one above deals with SSH keys


https://github.com/toshywoshy/ansible-vm-install/blob/master/playbooks/templates/debian/preseed.cfg

The one above is cool that has late command to pull and run an SH script from a URL. This SH can install Ansible and continue with the provisioning.

Here's another example to invoke a remote rc script

http://preseed.panticz.de/preseed/debian-minimal.seed

More preseeds here: https://git.ipr.univ-rennes1.fr/cellinfo/tftpboot/src/branch/master/preseed/debian/buster

References:
https://github.com/linuxserver/docker-netbootxyz
https://github.com/samdbmg/dhcp-netboot.xyz
https://wiki.jarylchng.com/books/linux/page/setting-up-linuxservernetbootxyz-docker-image-and-dnsmasq-dhcp-in-proxy-mode-when-your-main-router-has-locked-dhcp-settings
https://git.ipr.univ-rennes1.fr/cellinfo/tftpboot
https://www.debian.org/releases/buster/amd64/apb.en.html

