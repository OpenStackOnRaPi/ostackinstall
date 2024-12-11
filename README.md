# OpenStack installation on Raspberry Pi
This repo describes how to install OpenStack on a Raspberry Pi cluster using Kolla-Ansible.

## Table of contents

1. [Introduction](#introduction)
2. [Assumptions](#assumptions)
3. [Raspberry Pi preparation](#raspberry-pi-preparation)
   1. [RaPi system configuration](#rapi-system-configuration)
   2. [RaPi network configuration](#rapi-network-configuration)
5. [Management host preparation](#management-host-preparation)
   1. [Management host system configuration](#management-host-system-configuration)
   2. [Management host environment configuration](#management-host-environment-configuration)
7. [Kolla-ansible and OpenStack installation](#kolla-ansible-and-openstack-installation) 

## Introduction

The scope of application of our clusters is education. A cluster of this type allows us to present/explain various features/concepts of OpenStack that are hard or impossible to show using AIO or virtualized setups. Many of them are related to the administration of OpenStack data centre - a domain of activity that is hidden from regular users in "normal" DCs. For example, the management of provider networks where the admin needs to configure VLANs in the physical network of the data centre and declare them in OpenStack config file. Considering that the time needed to practice and learn even basic things is non negigible, a decent amount of resources is needed to serve a dozen or more student teams. One does not need to allocate dozens of servers worth thousands of $ each for that.

Currently, Raspberry Pi 4 is assumed as the HW implementation base. It is expected that extensions (if any) needed for Rraspberry Pi 5 will be added to this guide in the future once we test RaPi 5 setup sufficiently well.

Worth of noting is also that our clusters are a perfect base for experimenting with Kubernetes. In our lab, we use bare-metal setup of K3s - an excellent match for Raspbbery Pi.

## Assumptions

All procedures described herein refer to HW and SW setup of the cluster as specified below:

1. Raspberry Pi 4
   * [1x4GB RAM + 1x8GB RAM] or [2x4GB RAM+2*8GB RAM] per cluster
   * all Pi are equipped with 32GB SD disk
   * Note: we believe that a single RaPi 8GB RAM under the all-in-one setup of Kolla-Ansible OpenStack should work well for basic evaluation, but we have not tested this option.
2. SW
   * OS: Raspberry Pi OS Lite (64bit), a port of Debian 12 (Bookworm) with no desktopp environment
   * Kolla-Ansible 2023.1; respective envirnment components according to 
   * Note: newer releases of Kolla-Ansible will be tried in the future (2023.1 has got status "unmaintained" recently) and we'll update these instructions accordingly after completing the tests 
4. Network:
   * the Pis are equipped with 802.3af/at PoE HAT from Waveshare
   * they are powered form TP-Link TL-SG105PE switch
   * TP-Link switch is connected to a local router with DHCP enabled to separate the network of OpenStack DC frome the rest of the network environment
5. Notes
   * other PoE HATs for Raspberry Pi 4 and other PoE switches should work, too
   * for education purposes, we use setups with at least 3 RaPis and a managed switch (802.1Q) to demonstrate how VLAN-based provider networks can be used in OpenStack; this is impossible to show using AIO (all-in-one) OpenStack setups
   * other details that may be relevant are explained in the description that follows
   * trials with Raspberry Pi 5 are planned for the near future
  
## Raspberry Pi preparation

The following has to be done for each Rasppbery Pi in your cluster. The instructions will be given one by one, but you are free to gather them in bash scripts if you wish (sometimes a reboot is needed so you will have to prepare a couple of such scripts or you could prepare Ansible playbook to automate the installation completely, but how to do it is out of the scope of this guide). The configurations include two phases: system configuration (inslalls, upgrades, etc.) and configuration of the network configuration.

### RaPi system configuration

1. Flash the OS (Raspberry Pi OS Lite (64bit), a port of Debian 12 (Bookworm) with no desktopp environment) onto microSD card. We recommend using Raspberry Pi imager.
   * make sure password authentication for ssh access is enabled (the instructions given below fit this authentication method)
   * it is recommended to set the value of host name, user name and password as you will use afterwards in Kolla-Ansible playbooks. In the examples below, we set "ubuntu" for both the user name and password, and use the convention ost01, ost02, ... to set Raspbbery Pi host name.
  
2. After switching on the RaPi, find its IP address. In our setup, check the ```Device list``` panel in the Linksys router GUI (access on, e.g., 192.168.1.1, login as root, pwd admin). SSH to the RaPi using the credentials from step 1 above.

2. Assuming your user name on the RaPi is ubuntu (otherwise, adapt the following) run

   ```$ sudo usermod -aG sudo ubuntu```

4. Stop NetworkManager, and and start systemd-networkd

   _Note: for some historical reasons, we use networkd for defining persistent configuration of network devices on our RaPis; one can use NetworkManager for this, but it will be necessary to convert respective network constructs from networkd to NetworkManager notation (different form the one used by networkd)._

```
$ sudo systemctl stop NetworkManager
$ sudo systemctl disable NetworkManager
$ sudo systemctl enable systemd-networkd && sudo systemctl start systemd-networkd
$ sudo systemctl status systemd-networkd                  <= should be Active: active (running) ... 
```

5. Configure interface eth0 for networkd (it is case sensitive!)

```
$ sudo tee /etc/systemd/network/20-wired.network << EOT
[Match]
Name=eth0

[Network]
DHCP=yes
EOT
```

6. Install and enable netplan (ref. https://installati.one/install-netplan.io-debian-12/?expand_article=1)

```
$ sudo apt-get update && sudo apt-get -y install netplan.io
$ sudo netplan generate
$ sudo netplan apply

# check the connectivity
$ ping wp.pl

```

6. System upgrade

```
$ sudo apt-get remove unattended-upgrades -y && sudo apt-get update -y && sudo apt-get dist-upgrade -y
```

7. Installs for the use by Ansible

```
$ sudo apt-get install sshpass -y 
$ sudo apt-get install ufw -y     <=== needed on debian, not necessary on Ubuntu
$ sudo visudo    ==> change user group "sudo" permissions to:
# Allow members of group sudo to execute any command
%sudo ALL=(ALL:ALL) NOPASSWD: ALL
```

8. Install usefull tools

   Note: ```lm-sensors``` does not serve OpenStack purposes directly, but can be used to monitor CPU temperature (one has to ssh onto the RaPi) 

```
$ sudo apt-get install net-tools -y && sudo apt-get install lm-sensors -y

# run to check CPU temperature
$ sensors
```

8. Enable packet forwarding on the RaPi

```
$ sudo nano /etc/sysctl.conf

# uncomment the line: net.ipv4.ip_forward=1
# save the file, quit and check if forwarding has been activated:

$ sudo sysctl -p
```

9. Install qemu-system-arm (qemu-kvm) - critical for enabling virtualization

   Note: you can check first:
   
   * ```$ sudo apt install --simulate qemu-kvm```
   
   * ```$ sudo apt show qemu-system-arm```
   
```
$ sudo apt-get update && sudo apt-get install -y qemu-system-arm
```    

10. Upgrade for any case, reboot

```
$ sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get autoremove -y && sudo reboot    
```

### RaPi network configuration

#### Configuration description

We must configure network devices on our RaPi to meet Kolla-Ansible requirements for network interfaces. In particular, Kolla-Ansible requires that there are at least two network interfaces available on each OpenStack host (Kolla-Ansible user will then assign various OpenStack roles to those interfaces). As Raspbbery Pi has only one network card we have to create virtual interfaces to fulfill the above qualitative requirement. To this end, we create veth pairs and a linux bridge, and put them together them in appropriate configuration. This is shown in the figure below where also the role of respective interfaces is depicted. In our setup, interfaces ```veth0``` and ```veth1``` correspond to OpenStack host physical interfaces. They will be configured by Kolla-Ansible according to OpenStack networking principles and we assume that ```veth0``` and ```veth1``` serve as Kolla-Ansible ```network_interface``` and ```neutron_external_interface```, respectively. For more information on Kolla-Ansible networking for OpenStack, please refer to Kolla-Ansible documentation.

```
network_interface                 neutron_external_interface 
(OStack svcs, tenant nets)        (provider networks, tetnant routers/floating IPs)
 IP 192.168.1.6x/24               no IP addr assigned (Kolla-Ansible requires that)
    +---------+                     +---------+
    |  veth0  |                     |  veth1  |  <=== intfcs to be declared in globals.yml, used by Kolla-Ansible and OpenStack
    +---------+                     +---------+
         |                               |            HOST NETWORK domain ("host-internal" - under Nova/Neutron governance)
    - - -|- - - - - - - - - - - - - - - -|- - - - - - - - - - - - - - - - - - - - - - - - -                      
         | <-------- veth pairs -------> |            DATA CENTRE NETWORK domain ("physical" - under DC admin governance)
    +---------+                     +---------+ 
    | veth0br |                     | veth1br |       VLANs have to be configured in case of using provider VLAN networks
    +---------+                     +---------+
    +----┴-------------------------------┴----+
    |                   brmux                 |       L2 device, IP address not needed here, VLANs have to be configured here
    +---------------------┬-------------------+       in case of using provider VLAN networks
                     +---------+
                     |  eth0   |      physical interface of RaPi (taken by brmux), no IP address is needed, but VLANs 
                     +---------+      have to be configured in case of using provider VLAN networks
```

To make sure the above structure is persistent (survives system reboots), we use ```networkd``` and ```netplan``` files to define our network setup. Basically, networkd files allow to define tagged VLANs on ```eth0```, ```brmux```, ```veth0br``` and ```veth1br```, while neplan complements the definitions with the rest of needed information. In fact, the use of both levels (networkd and netplan files) was necessary a time ago when it was not possible to configure tagged VLANs solely in netplan. This may have changed since then and it may happen that with newer releases of netplan all needed configurations (including tagged VLANs on eth0, brmux, veth0br and veth1br) are possible using netplan (interested user can check it on her/his own). For more details on how to configure network devices in networkd and netplan, please refer to respective documentation.

#### Configuration implementation

In the following, terminal commands to be run on each RaPi are shown (ssh to the RaPi first).

**NOTE: this setup is prepared for a flat provider network only in the OpenStack DC. To allow for VLAN provider networks, additional configurations are needed for ```eth0```, ```brmux``` and ```veth1br``` to set the VLANs that should be served by those devices (respective configurations of VLANS should also be introduced in the TP-Link switch). If you are interested in setting VLAN provider networks in your cluster, contact the instructor for more info.**

  * networkd, for veth0-veth0br pair
```
$ sudo tee /etc/systemd/network/veth-openstack-net-itf-veth0.netdev << EOT
#network_interface w globals kolla-ansible
[NetDev]
Name=veth0
Kind=veth
[Peer]
Name=veth0br
EOT
```

  * networkd, for veth1-veth1br pair
```
$ sudo tee /etc/systemd/network/veth-openstack-neu-ext-veth1.netdev << EOT
#neutron_external_interface w globals kolla-ansible
[NetDev]
Name=veth1
Kind=veth
[Peer]
Name=veth1br
EOT
```

  * netplan, remaining settings for the network
    * **NOTE: adjust the IP address of veth0 according to you network setup**
```
$ sudo tee /etc/netplan/50-cloud-init.yaml << EOT
# This file is generated from information provided by the datasource. Changes
# to it will not persist across an instance reboot.  In ubuntu, to disable
# cloud-init's network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
# in debian, this is not needed.
#=========================================#
# Network configs for the OpenStack lab   #
#=========================================#
network:
  version: 2
  renderer: networkd

#-----------------------------------------#
# WiFi configurations (as the last-resort)#
#   (uncomment the operations if needed)  #
#-----------------------------------------#

## wlan0 itf will receive IP address from the Linksys DHCP.
#  adjust access point SSID and password according to your environment
#  wifis:
#    wlan0:
#      access-points:
#        FreshTomato06:
#          password: klasterek
#      dhcp4: true
#      optional: true

#-----------------------------------------#
# Kolla-Ansible networking configurations #
#-----------------------------------------#

# Interfejsy

  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false

    # veth0-veth0br pair
    veth0:                  # network_interface for kolla-ansible
      addresses:
        - 192.168.1.6x/24   # ADJUST THIS ADDRESS FOR EACH YOUR RAPI !!!!!!!!
      nameservers:
        addresses:
          - 192.168.1.1     # Lnksys dhcp server
          - 8.8.8.8
          - 8.8.2.2
      routes:
        - to: 0.0.0.0/0
          via: 192.168.1.1  # Linksys router
    veth0br:
      dhcp4: false
      dhcp6: false

    # veth1-veth1br pair
    veth1:                  # neutron_external_interface for kolla-ansible
      dhcp4: false
      dhcp6: false
    veth1br:
      dhcp4: false
      dhcp6: false

# Bridge brmux - intermediary 
# logically, this is a switch belonging to in the infrastructure of the
# data centre provider network (physical)

  bridges:
    brmux:
      interfaces:
        - eth0
        - veth0br
        - veth1br
EOT
```


## Management host preparation

### General notes

1. **Maintaining 100% consistency between the version of Kolla-Ansible used and the OpenStack release deployed is key for successfull installation of the OpenStack cloud.**
2. **This guide refers to OpenStack release ```2023.1``` and respective Kolla-Ansible guide is available under the link ```https://docs.openstack.org/kolla-ansible/2023.1/user/quickstart.html)```. Please, note the ```2023.1``` discriminator of OpenStack release in the URI.**
3. **Basically, the guide instructs how to install OpenStack cloud with Kolla-Ansible. For information how to manage OpenStack cloud using Kolla-Ansible, please refer to the original documentation of the Kolla-Ansible project.**

### Management host system configuration

#### Assumption: management host is implemented as Ubuntu 22.04 desktop od VirtualBox. Other solutions will work too after appropriate adaptations.

1. Basic configs

  * Create the VM as desktop machine. There are not high requirements for the resources (4GB RAM, 20GB disk, 2vCPU should be sufficient)
  * Configure the network card of the VM in VirtualBox as ```Bridged```. This is well suited for Ansible as it may not work well running behind NAT.
  * Your host should work in the same network as the cluster (in our lab setup, attach it to the Linksys router)
  * After launching the VM, the copy-paste feature will probably not work. You will have have to install GuestAdditions. This can be done in a while. First follow the next steps.
  * Disable the automatic upgrade option; in desktop search for this in Options
  * If the terminal suspends/does not open, the screen is flickering or the cursor takes the form of a black rectangle, disable Wayland display server protocol, see e.g., [this](https://linuxconfig.org/how-to-enable-disable-wayland-on-ubuntu-22-04-desktop)
  * If your user (we assume username ```ubuntu``` in the following) has not sudo privileges
```
$ su
$ usermod -aG sudo $USER
$ reboot
```

  * Run
```
$ sudo apt update && sudo apt upgrade
```
  * Install GuestAdditions
    refer, e.g., to [this](https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/how-to-install-virtualbox-guest-additions-on-ubuntu-22-04.html?utm_content=cmp-true)
  * Enable passwordless sudo logging on the management host (required by Ansible)
```
$ sudo apt install sshpass
```

### Management host environment configuration



## Kolla-Ansible and OpenStack installation




 
