<h1 align="center">
Preboot Execution Environment Server
</h1>  

<p align="center">
  <img src="https://github.com/user-attachments/assets/143367fb-07d3-4ed6-8be6-e4bf6c60d7bf" width="200" alt="PXE Logo" />
</p>


<p align="center">  
  <img src="https://img.shields.io/badge/-PXE%20Boot-green?style=flat" />  
  <img src="https://img.shields.io/badge/-TFTP-blue?style=flat" />  
  <img src="https://img.shields.io/badge/-DHCP-orange?style=flat" />  
  <img src="https://img.shields.io/badge/-Kickstart-red?style=flat" /> 
  <img src="https://img.shields.io/badge/-NFS-purple?style=flat" />
</p> 

<hr/>

# PXE Server
A PXE server (Preboot eXecution Environment server) is a system that allows computers (usually without operating systems) to boot over the network, instead of from a local hard drive or CD/USB.
PXE lets you install an OS (like RHEL, Ubuntu, etc.) on a computer remotely and automatically, without needing physical access.

---

## PXE server three main services:
- ```DHCP```: Tells the client where to boot from
- ```TFTP```:  Sends the bootloader & kernel/initrd to the client
- ```NFS```:  Provides the installation media & Kickstart

---

## PXE Boot Workflow

When a VM or bare-metal machine powers on, the following sequence occurs:

1. **DHCP Request**  
   The machine sends out a DHCP broadcast:  
   _"Where do I boot from?"_

2. **PXE Server Response**  
   The PXE server responds with:
   - An **IP address** (via DHCP)
   - The **bootloader file path** (via DHCP Options 66 & 67)

3. **TFTP Download**  
   The client uses **TFTP** to download:
   - The bootloader (e.g., `pxelinux.0`)
   - The kernel image (`vmlinuz`)
   - The initial RAM disk (`initrd.img`)

4. **Bootloader Execution**  
   The bootloader executes and starts the OS installer.

5. **OS Installation**  
   The installer fetches the full OS image from:
   - An **NFS** in our case.

6. **Automated Setup**  
   If a **Kickstart** file is provided, the installation runs fully automated based on predefined instructions.

---

## Configuration Files
- **DHCP config**: /etc/dhcp/dhcpd.conf
- **TFTP root**: /var/lib/tftpboot/
- **OS installation media**: /rhel7-install/os/
- **Kickstart file**: /ks/anaconda-ks.cfg

---

# PXE Server Lab Setup (CentOS 7 VM)
This section describes how to prepare a CentOS 7 VM for PXE boot server configuration.

---

### 1. Create a CentOS 7 Virtual Machine

- Allocate sufficient resources (e.g., 2 GB RAM, 2 vCPUs, 20 GB disk).
- Install **CentOS 7 Minimal** or GUI version.

---

### 2. Configure Two Network Interfaces

- **NIC 1 – NAT**  
  - Provides internet access to install packages and update the system.
- **NIC 2 – Host-only**  
  - Used to provide DHCP, TFTP, and PXE boot services to other VMs.
  - Allows SSH from your host machine.

Ensure both interfaces are enabled in `nmtui` or `ifcfg-*` config files.

---

### 3. Copy the CentOS ISO to the VM

Transfer the ISO image using one of the following methods:


From the powershell:
```bash
scp "C:\path\to\your.iso" username@VM-IP:/rhel7-install/
```

---

# Installation Steps

Mount iso in the /rhel7-install/os dir in the VM
```bash
sudo mount -o loop /rhel7-install/name-of-the-iso /rhel7-install/os
```
--- 

Configure repos in Centos:
```bash
sudo vi /etc/yum.repos.d/CentOS-Base.repo
```

Edit the baseurl and hash the mirror:
```bash
baseurl=http://vault.centos.org/7.9.2009/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=7&arch=$basearch&repo=os&infra=stock
gpgcheck=0
```

After editing use this commands:
```bash
sudo yum clean all
sudo yum makecache
sudo yum update
```
--- 

## Install and configure NFS
```bash
yum install –y nfs-utils
mkdir /ks
```
Edit /etc/exports with the following
```bash
vi /etc/exports
```
```bash
/rhel7-install/os *(ro,sync,no_root_squash,no_subtree_check)
/ks *(ro,sync,no_root_squash,no_subtree_check)
```
Start the NFS service
```bash
systemctl enable --now nfs.service
```

--- 

## Install and Configure DHCP
```bash
yum install dhcp -y
```

Edit the /etc/dhcp/dhcpd.conf file
```bash
vi /etc/dhcp/dhcpd.conf
```
```bash
option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;
 
# Subnet for ens33 (192.168.88.0/24)
subnet 192.168.88.0 netmask 255.255.255.0 {
    option routers 192.168.88.2;
    range 192.168.88.100 192.168.88.200;
    option domain-name-servers 8.8.8.8;
    next-server 192.168.88.39;  # IP of your PXE server on this subnet
 
    class "pxeclients" {
        match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
 
        if option architecture-type = 00:07 {
            filename "grubx64.efi";  # UEFI boot
        } else {
            filename "pxelinux.0";  # BIOS boot
        }
    }
}
```

> **Note:**  
> Edit the IPs according to your network setup


Start the DHCP service
```bash
systemctl enable --now dhcpd
```
---

## Install and configure TFTP Server
```bash
yum install tftp-server -y
yum install xinetd -y
```
> **Note:**  
> Any File under /var/lib/tftpboot will be visable for hosts

Edit the conf file /etc/xinetd.d/tftp
```bash
vi /etc/xinetd.d/tftp
```
```bash
# default: off
# description: The tftp server serves files using the trivial file transfer \
#       protocol.  The tftp protocol is often used to boot diskless \
#       workstations, download configuration files to network-aware printers, \
#       and to start the installation process for some operating systems.
service tftp
{
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = root
        server                  = /usr/sbin/in.tftpd
        server_args             = -v -v -s /var/lib/tftpboot -B 1468
        disable                 = no
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
```
---

> **Note:**  
> To automate the booting process, we need some important files. We'll organize these files in a clear way to make the setup easy and work smoothly.


### Prepare files for booting process
Install the package syslinux that include PXE boot files like pxelinux.0
```bash
sudo yum install syslinux -y
```
After installation, you’ll find it here:
```bash
/usr/share/syslinux/pxelinux.0
```

Copy pxelinux.0 to the TFTP directory

```bash
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
cp /usr/share/syslinux/menu.c32 /var/lib/tftpboot/
```

Create the directory pxelinux.cfg/ in the tftpboot/ directory:
```bash
mkdir /var/lib/tftpboot/pxelinux.cfg
```

Add a configuration file named default to the pxelinux.cfg/ directory.
```bash
DEFAULT menu.c32
PROMPT 0
TIMEOUT 100
ONTIMEOUT linux
 
MENU TITLE PXE Boot Menu
 
LABEL linux
  MENU LABEL Install RHEL 7
  KERNEL images/RHEL-7.1/vmlinuz
  APPEND initrd=images/RHEL-7.1/initrd.img inst.repo=nfs:192.168.88.39:/rhel7-install/os/ inst.ks=nfs:192.168.88.39:/ks/anaconda-ks.cfg ip=dhcp
```

Create a subdirectory to store the boot image files within the /var/lib/tftpboot/ directory, and copy the boot image files to it.
```bash
cp /rhel7-install/os/images/pxeboot/{initrd.img,vmlinuz} /var/lib/tftpboot/images/RHEL-7.1/
```

Create kickstart file
```bash
vi /ks/anaconda-ks.cfg
```
```bash
#version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use graphical install
# Use NFS installation media
nfs --server=192.168.88.33 --dir=/rhel7-install/os/
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8
 
# Network information
network  --bootproto=dhcp --device=ens33 --ipv6=auto --activate
network  --hostname=localhost.localdomain
 
# Root password
rootpw --iscrypted $6$l0ZZ/SWkUFDyeyYJ$IIxRLqKVreJVBtxd5M9egDyIG26SrdIGFQmhHCOjrADUjRLrQo.TIn/nWXj6DPWf2oG.BZ5w9YtF8JmK.hZu50
# System services
services --enabled="chronyd"
# System timezone
timezone America/New_York --isUtc
# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
autopart --type=lvm
# Partition clearing information
clearpart --all --initlabel
 reboot

%packages
@^minimal
@core
chrony
kexec-tools
 
%end
 
%addon com_redhat_kdump --enable --reserve-mb='auto'
 
%end
 
%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
```

Start the Xinetd service
```bash
systemctl enable --now xinetd
```

Configure the Firewall
```bash
# Allow DHCP server (UDP port 67)
sudo firewall-cmd --add-service=dhcp --permanent

# Allow TFTP (UDP port 69)
sudo firewall-cmd --add-service=tftp --permanent

# Allow NFS (TCP/UDP ports: 2049 and others)
sudo firewall-cmd --add-service=nfs --permanent
sudo firewall-cmd --add-service=mountd --permanent
sudo firewall-cmd --add-service=rpc-bind --permanent

# Reload the firewall to apply changes
sudo firewall-cmd --reload
```
```bash
systemctl daemon-reload
```
--- 


<h3 align="center" style="color:red;">🚨 And now you have it! 🚨</h3>

<p align="center">
  <img src="https://github.com/user-attachments/assets/33c61423-ac1f-42d0-ab64-a7919b9fefc2" alt="ghostedvpn-hacker-cat"/>
</p>
































