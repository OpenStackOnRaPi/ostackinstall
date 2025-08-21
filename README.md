# OpenStack on Raspberry Pi

**TODO: init-runonce files to be updated !!!**

## Executive summary

In this repo, we describe how to install OpenStack on Raspberry Pi cluster using Kolla-Ansible. The main field of application of such clusters is instructional tools for teaching. While you can find other guides on the Internet on this topic, they are generally based on an old version of OpenStack and do not cover a number of important details that would make an OpenStack Rapberry Pi cluster run reliably.

A number of OpenStack cluster configurations have been tested thoroughly. Each configuration was determined by Raspberry Pi board type (4B or 5), Kolla-Ansible OpenStack release (2023.1 or 2025.1), virtualization type (Qemu emulation or KVM). In all configurations, the RPis run under Raspberry Pi OS (Debian 12 Bookworm derivative). We confirm that KVM virtualization works excellent on RPi 4 and RPi 5 for both tested releases of Kolla-Ansible/OpenStack (2023.1 and 2025.1). Qemu only works well on RPi 4 , and the VMs are more than twice as slow compared to when host-passthrough KVM is used (and Qemu requires additional settings to be added to the Kolla-Ansible globals.yml and nova-compute.conf configuration files). Qemu has failed to work on RPi 5 because there is no support for the Cortex-A76 processor model in the libvirt package installed by Kolla-Ansible (we base this conclusion on our analysis of libvirtd logs). Considering the above, KVM is the recommended virtualization option for OpenStack deployments on the Raspberry Pi platform. Moreover, in all cases tested we had to increase the swap memory size on the OpenStack control node from 512 MB (the default value in the Raspberry Pi OS) to a much safer capacity of 4 GB.

In summary, both the Raspberry Pi 4 and 5 are suitable for setting up small and cheap, bare metal OpenStack clusters for education purposes. This guide covers Kolla-Ansible releases 2023.1 and 2025.1, with release 2025.1 being considered the reference version (differences for version 2023.1 are discussed directly in the text). Further updates will be added as new results or recommendations emerge.

## Table of contents

1. [Introduction](#1-introduction)
2. [Platform components](#2-platform-components)
3. [Raspberry Pi preparation](#3-raspberry-pi-preparation)
   1. [RPi system configuration](#3i-rpi-system-configuration)
   2. [RPi network configuration - pure flat provider network](#3ii-rpi-network-configuration---pure-flat-provider-network)
   3. [VLAN provider networks - part 1 (RPi network configuration for flat network)](#3iii-vlan-provider-networks---part-1-rpi-network-configuration-for-flat-network)
4. [Management host preparation](#4-management-host-preparation)
   1. [General notes](#4i-general-notes)
   2. [VM creation and basic configs](#4ii-vm-creation-and-basic-configs)
   3. [Docker installation](#4iii-docker-installation)
5. [Kolla-ansible and OpenStack installation](#5-kolla-ansible-and-openstack-installation)
   1. [Koll-Ansible installation](#5i-kolla-ansible-installation)
   2. [Prepare configuration files for Kolla-Ansible](#5ii-prepare-configuration-files-for-kolla-ansible)
   4. [Configure Kolla-Ansible files for specific OpenStack depolyment](#5iii-configure-kolla-ansible-files-for-specific-openstack-depolyment)
   5. [Deploy OpenStack](#5iv-deploy-openstack)
   6. [Postdeployment and the first instance](#5v-postdeployment-and-the-first-instance)
6. [Managing your cluster](#managing-your-cluster)
   1. [Shut down the cluster and start it again](#6i-shut-down-the-cluster-and-start-it-again)
   2. [Destroy your cluster](#6ii-destroy-your-cluster)
7. [VLAN provider networks - part 2 (enabling and using VLAN provider networks)](#7-vlan-provider-networks---part-2-enabling-and-using-vlan-provider-networks)
   1. [Setting VLANs for provider networks](#7i-setting-vlans-for-provider-networks)
      1. [Setting VLANS in the physical network (the TP-Link switch)](#7ia-setting-vlans-in-the-physical-network-the-tp-link-switch)
      2. [Setting VLANS on the RPi hosts](#7ib-setting-vlans-on-the-rpi-hosts)
      3. [Separating the flat network using a VLAN](#7ic-separating-the-flat-network-using-a-VLAN)
   3. [Creating and using VLAN provider networks](#7ii-creating-and-using-vlan-provider-networks)
      1. [Provider network dedicated to a tenant](#7iia-provider-network-dedicated-to-a-tenant)
      2. [External network using VLAN provider network](#7iib-external-network-using-vlan-provider-network)
8. [ADDENDUM - accessing the cluster using VPN](#8-addendum---accessing-the-cluster-using-vpn)
   
## 1. Introduction

The scope of application of our clusters is teaching. A cluster of this type allows us to present/explain various features/concepts of OpenStack, some of them being hard or impossible to show using AIO or virtualized OpenStack setups. Many of such features are related to the administration of OpenStack data centre - a domain of activity that is hidden from regular users in "true" DCs. For example, the management of provider networks where the admin needs to configure VLANs in the physical network of the data centre and declare them in OpenStack and Kolla-Ansible config files. Considering that the time needed to practice and learn even basic things is non negigible, a decent amount of resources is needed to serve a dozen or more student teams in limited time. Our approach allows us to achieve these goals without having to reserve dozens of servers, each worth thousands of $, for extended periods of time.

This guide covers several steps leading to the instantiation of your first VM in RPi OpenStack cluster. This includes: **system configuration on the Raspberries** (OS installation, setting needed permissions, configuring network settings that are required by Kolla-Ansible, etc.), **configuration of the management host** (a separate host is used where we execute Kolla-Ansible installation commands and later access OpenStack using command line tool), **installation and customization of Kolla-Ansible** and customization of its configuration files on the management host to meet the needs of our deployment, **actual deploymnet of OpenStack** using Kolla-Ansible, then **creation of basic elements of virtualized infrastructure** (VM image of CirrOS, external and tenant networks, public and private routers, security groups, etc.) and finally **creation of the first VM instance** in our cloud. These steps are detailed in the remainder of this document.

> [!NOTE]
> * There are other minicomputer platforms available on the market that may seem attractive for implementing OpenStack educational deployments, to mention Turing Pi 2 / 2.5 and DeskPi Super6C. However, they support Raspberry Pi CM4 modules and not CM5 (Turing 2.5 supports also other, more powerfull modules, but not Raspberry Pi CM5), and result in higher budget requirements for the same educational result. Moreover, they are not so easily available in various countries so one may prefer to adhere to more conservative but "safer" derivatives of [Raspberry Pi Dramble](https://www.pidramble.com/) :grin:.
> 
> * It's also worth noting that Raspberry Pi clusters make an excellent base for Kubernetes experiments. In our lab, we use a bare-metal K3s configuration, which works perfectly for the Raspberry Pi.

## 2. Platform components

All procedures described in this guide assume compliance with the setup options for HW and SW in the cluster as specified below. Other configurations may also work perfectly (e.g., those with more hosts will definitely work), but we haven't tested them.

1. Raspberry Pi 4 and Raspberry 5
   * tested options: [2x4GB RAM + 2x8GB RAM] or [3/4x8GB RAM] per cluster (uniform 8GB RAM clusters are better)
     * mimimal possible: [1x4GB RAM + 1x8GB RAM] (a minimal cluster, enough to create a single CirrOS virtual machine and nothing more)
     * OpenStack control/network nodes run on the same 8GB RPi host
   * all Pi are equipped with 32GB SD disk
   * Note: a single 8GB RAM RPi host in all-in-one setup of Kolla-Ansible OpenStack is barely able to host OpenStack in absolutely minimal configuration. In such a cluster, one can create a single CirrOS instance with 512MB memory, but the cloud becomes unstable. We have seen many times that even such a simple configuration ends in failure soon after instantiating the VM. And OpenStack all-in-one system immediately crashes due to lack of memory when a second similar CirrOS instance is created, unless you increase the swap memory size. This second option, however, carries the risk that everything will become very slow. That is the reason why we consider the dual-host configuration to be the minimum possible.
   * An illustrative diagram of the cluster with Linksys WRT54-GL as the router is shown below:

<p align="center">
<img src=images/cluster.jpg width='65%' />
</p>
   
2. SW
   * OS: Raspberry Pi OS Lite 64bit (a port of Debian 12 Bookworm with no desktopp environment).
   * Kolla-Ansible 2023.1 or 2025.1. 
3. Network:
   * the RPis are equipped with 802.3af/at PoE HAT from Waveshare (PoE is optional but simplifies cluster wiring) 
   * they are powered form TP-Link TL-SG105PE switch (it supports 802.1Q which can be used to set multiple VLAN provider networks in OpenStack)
   * TP-Link switch is connected to a local router with DHCP enabled to isolate the network segment of OpenStack DC from the rest of local network infrastructure
   * **reserve a pool of IP addresses for the use by OpenStack** on your local router (Linksys in the diagram shown above); 20 addresses will be sufficient for our purposes. They **must not** be managed by the DHCP server on the local router. Some of them will be assigned by you to the RPis using netplan (see [Configuring the network (flat)](#configuring-the-network-flat) in subsection 3.ii), and one will be allocated as the so-called ```kolla_internal_vip_address``` (see [section 5.iii](#5iii-configure-kolla-ansible-files-for-specific-openstack-depolyment)). Remaining addresses will serve as OpenStack ```floating IP addresses``` for accessing created VMs from the outside of your cloud.
4. Virtualization
   * we have tested qemu and KVM positively on Raspberry Pi 4, and KVM on Raspberry Pi 5 (qemu does not work correctly on Raspberry Pi 5 with standard Kolla-Ansible installation).
5. Notes
   * there are various types of PoE HAT for Raspberry Pi 4/5 and various PoE switches that also should work; one should only take care of the required power budget of the switch (please note that the required power budget will be different for the Raspberry Pi 4 and 5 platforms).
   * for general education purposes, we use setups with at least 3 RPis and a managed switch (802.1Q) in the cluster to demonstrate how VLAN-based provider networks can be used in OpenStack; this is impossible to show using AIO (all-in-one) OpenStack setups (and may be very tricky when using virtual machines as the OpenStack hosts). But if one does not need VLAN provider networks, unmanaged switch can be used as well.
   * other details that may be relevant are explained in the description that follows
  
## 3. Raspberry Pi preparation

The following has to be done for each Rasppbery Pi host in your cluster and the instructions will be described one by one. However, you are free to make some automation using bash scripts or other tools if you want (Note: sometimes a reboot is needed so you will have to prepare a couple of scripts for semi-automated installation or, e.g., Ansible playbook to automate the installation completely, but how to do it is out of the scope of this guide). The process is split into two phases: system configuration (installs, upgrades, etc.) and host network configuration (enabling networkd, installing netplan).

You will access the devices in the cluster (RPis, TP-Link) using ssh. The IP addresses to use can be checked in Linksys panel `Status->Device List`:

<p align="center">
 <img src=images/device-list-linksys.png width='62%' />
</p>

Before you begin, make sure your TP-Link switch is set to the manufacturer's settings. One can press a button in a small hole on the back side of the switch for a couple of seconds. It is important that there are no VLANS set in the `VLAN->802.1Q VLAN Configuration` tab, i.e., after logging to TP-Link the following is correct (otherwise all existing VLANs should be deleted):

<p align="center">
   <img src=images/tplink-factory.png width='60%' />
</p>

> [!Note]
> In our TP-Link routers, after restoring factory settings, we log in as user=admin, passwd=admin and the password change is forced upon first login.

Now you can proceed to the next subsection.

### 3.i RPi system configuration

Basically, we follow the guidlines from [Kolla-Ansible support matrix](#https://docs.openstack.org/kolla-ansible/2023.1/user/support-matrix.html) in choosing the installaion environment.

1. Flash the Raspberry Pi OS Lite 64bit (a port of Debian 12 Bookworm with no desktop environment) onto microSD card. We use Raspberry Pi Imager for that, but other tools can be used as well.
   * make sure password authentication for ssh access is enabled (the instructions given below fit this authentication method); this allows one to avoid the use of authentication keys in Ansible (not recommended for production, but simpler)
   * it is recommended to set host name, user name and password as you will use afterwards in Kolla-Ansible playbooks. In the examples below, we set "ubuntu" for both the user name and password, and use the convention ost01, ost02, ... to set the host name of our RPis.
  
2. Reserve a set of IP addresses in the subnetwork where your OpenStack instance will be run. They MUST NOT be managed by your local DHCP server. They will serve as fixed IP addresses of your RPi hosts, addresses of selected OpenStack services and so-called `floating IP addresses` used to expose instances (virtual machines) to the outside of the OpenStack cloud. A continuous set of fifteen addresses will be sufficient for our purposes.
  
3. After switching on the RPis, SSH to each of them using the credentials from step 1 above. Their IP addresses can be found in the management panel of your local router (in our lab setup, check the ```Device list``` panel in the Linksys router GUI). These addresses are one-time use and will later be overwritten with persistent (fixed) addresses during network stack configuration on each RPi host (you will draw those fixed IP addresses from the pool reserved in step 2 above).

**Execute steps 3-9 for each RPi**

3. Add your user to the `sudo` group (adapt user name according to your setup):

   ```$ sudo usermod -aG sudo ubuntu```

4. System upgrade (there is no package unattended-upgrades installed on debian, so remove can be skipped in this case)

   ```$ sudo apt-get remove unattended-upgrades -y && sudo apt-get update -y && sudo apt-get upgrade -y```

5. Install for the use by Ansible

  ```
$ sudo apt-get install sshpass -y && sudo apt-get install ufw -y  <=== "install fw" needed only on Debian, default on Ubuntu
$ sudo visudo    ==> change user group "sudo" permissions to:
# Allow members of group sudo to execute any command
%sudo ALL=(ALL:ALL) NOPASSWD: ALL
  ```

6. Install usefull tools

   Note: ```lm-sensors``` does not serve OpenStack purposes directly, but can be used to monitor CPU temperature (one has to ssh onto the RPi) 

  ```
$ sudo apt-get install net-tools -y && sudo apt-get install lm-sensors -y

# run to check CPU temperature
$ sensors
  ```

   * check temperature every 20 sec. (not necessary, just informative)
```
while :; do sensors; sleep 30; done
```

7. Enable packet forwarding on the RPi

  ```
$ sudo nano /etc/sysctl.conf

# uncomment the line: net.ipv4.ip_forward=1
# save the file, quit and check if forwarding has been activated:
$ sudo sysctl -p
  ```

8. Install qemu-system-arm (qemu-kvm) - critical for enabling virtualization

   Note: you can check first:
   
   * ```$ sudo apt install --simulate qemu-kvm```
   
   * ```$ sudo apt show qemu-system-arm```
   
  ```
$ sudo apt-get update && sudo apt-get install -y qemu-system-arm
  ```    

9. Increase swap memory size on the `control` node

> [!IMPORTANT]
> This configuration is only for the host that will combine OpenStack roles of "control node" and "network node" in your OpenStack cluster. In this guide, we assume these roles will be assigned to the host with name `ost04`. In [section 5.iii](#5iii-configure-kolla-ansible-files-for-specific-openstack-depolyment) you check how we assign roles to hosts in Kolla-Ansible inventory file `multinode`. Remaining hosts can have default swap memory size settings (512MB in the Raspberry Pi OS).
>
> Kolla-Ansible allows one to separate `control` and `network` functions by assigning them to different hosts in Ansible inventory. While this slightly reduces the maximal memory occupancy of the hosts, it does not relieve you of neither the need to increase the swap memory size nor the responsibility to monitor memory usage on the `control` and `network' hosts. Of course, if you are curious, you can give it a try.

  * follow the instructions below; they are drawn from [this guide](https://itsfoss.com/pi-swap-increase/). The highest usage of swap memory that we have observed so far is 2.5GB. Below we set the size to **`4096`** which should provide a safe margin, but you can double it if you want greater guarantees.
  ```
sudo dphys-swapfile swapoff
sudo nano /etc/dphys-swapfile
CONF_SWAPSIZE=4096   # <=== desired size, save
CONF_MAXSWAP=4096    # <=== should be no less than CONF_SWAPSIZE (limit)
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
swapon --show        # ensure it is now active
sudo reboot
  ```

### 3.ii RPi network configuration - pure flat provider network

In this section, we describe how to configure networking in our OpenStack providing support only for flat provider network. This is the simplest option regarding network configuration in OpenStack, still sufficient to demonstrate many OpenStack features. Introducing VLAN provider networks requires additional configurations in L2 of the data center. In our case, this concerns TP-Link switch and the internal network devices in our RPis: ```eth0```, ```brmux``` and ```veth1br``` (VLANs must be configured in all those devices).

If you want to use flat provider networks only, follow the remainder of this subsection and then continue with sections [5. Kolla-ansible and OpenStack installation](#5-kolla-ansible-and-openstack-installation) and [6. Managing your cluster](#managing-your-cluster).

If you are determined to use VLAN provider networks in your cluster, follow this section up to step _2. Install and enable netplan_ (but not _3. Host network configuration (for flat provider network)_), and then proceed to [3.iii VLAN provider networks - part 1 (RPi network configuration for flat network)](#3iii-vlan-provider-networks---part-1-rpi-network-configuration-for-flat-network). Note that in this case we use a two-step approach. In subsection 3.iii only a flat provider network will be set in the cluster, while VLAN provider networks will be deployed as late as in section [7. VLAN provider networks - part 2 (enabling and using VLAN provider networks)](#7-vlan-provider-networks---part-2-enabling-and-using-vlan-provider-networks).

#### Network configuration description

Network devices on our RPi have to meet Kolla-Ansible requirements for network interfaces. In particular, Kolla-Ansible requires that there are at least two network interfaces available on each OpenStack host (Kolla-Ansible user will then assign OpenStack roles to those interfaces). As Raspbbery Pi comes with only one network card we have to use virtual interfaces to fulfill the above requirement. We will create veth pairs and a linux bridge, and we will put them together to emulate desired configuration.

This is depicted in the figure below where also the role of respective interfaces is shown. In our setup, interfaces ```veth0``` and ```veth1``` correspond to physical interfaces in production OpenStack host. They will be configured by Kolla-Ansible according to Kolla-Ansible/OpenStack networking principles and we assume that ```veth0``` and ```veth1``` will serve as Kolla-Ansible ```network_interface``` and ```neutron_external_interface```, respectively. For more information on Kolla-Ansible networking for OpenStack, please refer to respective [documentation](https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html).

```
SCHEMATIC VIEW OF RPi INTERNAL NETWORK EMULATING REAL DC NETWORK ENVIRONMENT

network_interface, veth0          neutron_external_interface, veth1
(OStack services, tenant nets)    (provider networks, tenant routers/floating IPs)
static IP 192.168.1.6x/24         no IP address assigned (Kolla-Ansible requires that)
 
                                                    HOST NETWORK domain ("host-internal" in production),
                                                    - under Nova/Neutron governance
    +---------+                     +---------+
=== |  veth0  | =================== |   veth1 | ==  interfaces to be specified in globals.yml, used by Kolla-Ansible and OpenStack
    +----┬----+                     +----┬----+     they correspond to physical network cards (interfaces) in a production server
         |                               |          tagged VLANS will be configured for veth1 in case of using VLAN provider networks
         |                               |
         | <-------- veth pairs -------> |          DATA CENTER NETWORK domain ("physical" in production),
         |                               |          - under DC admin governance
    +----┴----+                     +----┴----+ 
    | veth0br |                     | veth1br |     tagged VLANs have to be configured by the admin from eth0 across veth1br to
    +----┬----+                     +----┬----+     veth1 in caseof using VLAN provider networks
    +----┴-------------------------------┴----+
    |                   brmux                 |     L2 device, IP address not needed here, tagged VLANs have to be configured here
    +---------------------┬-------------------+     (they extend towards veth1) in case of using provider VLAN networks
                          |                         - corresponds to a physical switch in data center L2 network
   Raspberry Pi      +----┴----+
   xxxxxxxxxxxxxxxxx |  eth0   | xxxxxxxxxxxxxx     physical interface of RPi (taken by brmux), no IP address is needed, but 
                     +---------+                    tagged VLAN have to be configured here in case of using VLAN provider networks
```

To make sure the above structure is persistent (survives system reboots), we use ```networkd``` and ```netplan``` configuration files to define our network setup. Basically, networkd files allow to define veth pair devices and tagged VLANs on ```eth0```, ```brmux```, ```veth0br``` and ```veth1br```, while neplan code contains the rest of needed information. In fact, the use of both levels (networkd and netplan files) was necessary a time ago when it was not possible to define all details solely in netplan. It cannot be ruled out that with newer releases of netplan all needed configurations (veth pairs and tagged VLANs on eth0, brmux, veth0br and veth1br) will be possible entirely in netplan (interested user can check it on her/his own). For more details on how to configure network devices in networkd and netplan, please refer to respective documentation.

#### Configuring the network (flat)

> [!Note]
> Steps 1 and 2 below are needed only in case of Debian and can be skipped for Ubuntu.

**_1. Stop NetworkManager, and and start systemd-networkd_**

We use networkd to have persistent configuration of network devices in our RPis. One can use NetworkManager for this, but it will be necessary to convert the code from networkd notation to NetworkManager notation (both notations differ one from the other).

  ```
$ sudo systemctl stop NetworkManager && sudo systemctl disable NetworkManager
$ sudo systemctl enable systemd-networkd && sudo systemctl start systemd-networkd
$ sudo systemctl status systemd-networkd                  <= should be Active: active (running) ... 
  ```

**_2. Install and enable netplan_** (ref. https://installati.one/install-netplan.io-debian-12/?expand_article=1)

  ```
$ sudo apt-get update && sudo apt-get -y install netplan.io
  ```

**_3. Host network configuration (for flat provider network)_**

> [!NOTE]
> This step is prepared for flat provider network only in your OpenStack DC. Introducing VLAN provider networks requires additional configurations for ```eth0```, ```brmux``` and ```veth1br``` to serve VLANs in those devices. Respective VLAN configurations have also to be introduced in the TP-Link switch. If you are interested in setting VLAN provider networks in your cluster, skip this step and go to [3.iii VLAN provider networks - part 1 (RPi network configuration for flat network)](#3iii-vlan-provider-networks---part-1-rpi-network-configuration-for-flat-network).

  * for `networkd` backend, for `veth0-veth0br` pair

```
sudo tee /etc/systemd/network/ost-net-itf-veth0.netdev << EOT
#network_interface w globals kolla-ansible
[NetDev]
Name=veth0
Kind=veth
[Peer]
Name=veth0br
EOT
```

  * for `networkd` backend, for `veth1-veth1br` pair
```
sudo tee /etc/systemd/network/ost-neu-ext-veth1.netdev << EOT
#neutron_external_interface w globals kolla-ansible
[NetDev]
Name=veth1
Kind=veth
[Peer]
Name=veth1br
EOT
```

  * for `netplan` utility, remaining settings of the network

    **NOTE 1: adjust the IP address of veth0 in each RPi according to you network setup.**

    **NOTE 2:** in case of problems during `netplan generate` (or `netplan apply`) check the format of your file `/etc/netplan/50-cloud-init.yaml` - it's YAML and spaces matter.

> [!Warning]
> This part sometimes does not copy-paste correctly. If command `sudo netplan generate` reports formatting errors then edit file /etc/netplan/50-cloud-init.yaml and manually format it to the form visible below by inserting missing spaces. 

```
sudo tee /etc/netplan/50-cloud-init.yaml << EOT
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
# WiFi configurations (optional)          #
# (uncomment if needed)                   #
#-----------------------------------------#
# WiFi can be hepful during configurations when we will teporarily loose connectivity to
# RPis from the fixed (cable) network used by OpenStack). RPis (but not OpenStack) will
# then be accessible via WiFi.
# WiFi should be configured as isolated subnetwork - distinct form the fixed (cable)
# network used to access OpenStack (provider external network/management network).
# E.g., on Linksys, you should dedicate a separate bridge (e.g., br1) to WiFi with
# distinct IP address range for this subnetwork. You will also have to enable routing
# between WiFi subnetwork and the fixed subnetwork used to access OpenStack.

## wlan0 interface will receive IP address from the Linksys DHCP.
#  adjust access point SSID and password according to your environment
#  wifis:
#    wlan0:
#      access-points:
#        someaccesspoint:
#          password: somepassword
#      dhcp4: true
#      optional: true

#-----------------------------------------#
# Kolla-Ansible networking configurations #
#-----------------------------------------#

# Interfaces

  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false

    # veth0-veth0br pair
    # note thet the veth pair device has to be defined in networkd file, not here
    veth0:                  # network_interface for kolla-ansible
      addresses:
        - 192.168.1.6x/24   # ADJUST THIS ADDRESS FOR EACH YOUR RPI !!!!!!!!
      nameservers:
        addresses:
          - 192.168.1.1     # ADJUST to Linksys dhcp server
          - 8.8.8.8
          - 8.8.2.2
      routes:
        - to: 0.0.0.0/0
          via: 192.168.1.1  # ADJUST to Linksys router
    veth0br:
      dhcp4: false
      dhcp6: false

    # veth1-veth1br pair
    # note thet the veth pair device has to be defined in networkd file, not here
    veth1:                  # neutron_external_interface for kolla-ansible
      dhcp4: false
      dhcp6: false
    veth1br:
      dhcp4: false
      dhcp6: false

# Bridge brmux 
# logically, this is a switch belonging to the local network of the
# data centre provider network (normally, L2 segment of a physical network)

  bridges:
    brmux:
      interfaces:
        - eth0
        - veth0br
        - veth1br
EOT
```

  > [!IMPORTANT]
  > Now edit file `/etc/netplan/50-cloud-init.yaml` and adjust static IP address, DHCP server and DNS server addresses for your RPi (see the Netplan code above). In the future, each RPi will be accessible on the static IP address just configured.

```
sudo nano /etc/netplan/50-cloud-init.yaml
```
  * deploy the changes permanently - create and configure devices
    
    Note: you will loose connectivity to your RPi because of the change of IP address. To continue, ssh again using the new address.

```
$ sudo netplan generate      <=== neglect a WARNING about too open permissions
$ sudo netplan apply         <=== neglect five WARNINGS about too open permissions

ssh disconnects so reconnect, but using fixed IP addresses you set in file `50-cloud-init.yaml`
check the connectivity
$ ping wp.pl
$ sudo reboot
```

### 3.iii VLAN provider networks - part 1 (RPi network configuration for flat network)

#### General

There's no single generic configuration of provider networks in OpenStack. In this guide, we describe how to deploy a combination of single flat and several VLAN (tagged) provider networks. A flat (untagged) provider network will support OpenStack management, external and tenant overlay networks (so similarly to the basic flat provider network setup described in section 3.ii). Additionally, we will create a set of VLANs allowing the admin to create additional, tagged provider networks. Such tagged provider networks can then be configured as external or internal (without external access) networks, depending on the actual needs in a given data center.

We adopt a two-step incremental approach. This section describes the first step. We will create a flat provider network setup already known from section 3.ii. This time most of the network configurations will be defined in networkd files (that is, in section 3.ii we used networkd as little as possible). This will result in an OpenStack network configuration that is identical to the one in section 3.ii (allowing you to perform the same experiments with OpenStack as with the configuration in section 3.ii). However, the current set of files will be better suited for conversion to enable VLAN provider networks.

In the second, crucial step, we will change some networkd configuration files to actually create tagged VLANs, enable VLAN provider networks and use them in our cluster. That is covered in section [7. VLAN provider networks - part 2](#7-vlan-provider-networks---part-2-enabling-and-using-vlan-provider-networks).

#### Configuring the network (flat)

  * First, if not already done, execute steps 1 and 2 from the previous section 3.ii (stop NetworkManager and install netplan).
  * Then upload the files stored in this repo in directory `flat` to respective directories on each RPi. Make sure to preserve the names of the paths contained inside directory `flat`. That is, files from the `flat/etc/netplan` directory to the `/etc/netplan` directory on the RPi, and from the `flat/etc/systemd/network` directory to the `/etc/systemd/network` directory. These files configure the RbPi for a flat network, meaning no VLANs. There is no Ethernet layer isolation between tenants; for now, this will be implemented in OpenStack using VXLANs, which encapsulate Ethernet frames into IP/UDP packets.
  * On each RPi, edit file `/etc/netplan/50-cloud-init.yaml` and update IP address of interface `veth0`, DHCP server address (the Linksys or other router in your network), and the default route (typically the same as the DHCP server address in your router).
```
sudo nano /etc/netplan/50-cloud-init.yaml
```
  * Then, if you are sure all files hav been prepared correctly simply reboot the RPi and it should be available on the statically assigned IP address provided in the /etc/netplan/50-cloud-init.yaml file. If you want to first check for correctness, do the following:

```
$ sudo netplan generate      <=== neglect a WARNING about too open permissions
$ sudo netplan apply         <=== neglect five WARNINGS about too open permissions

ssh disconnects so reconnect, but using fixed IP addresses you set in file `50-cloud-init.yaml`
check the connectivity
$ ping wp.pl
$ sudo reboot
```

Your network configuration is now the same as that from section 3.ii. You can continue with the installation procedure (section 4 and 5) to enjoy using OpenStack with a flat provider network.

## 4. Management host preparation

### 4.i General notes

1. Maintaining 100% consistency between the version of Kolla-Ansible used and the OpenStack release deployed is key for successfull installation of the OpenStack cloud. 
2. This guide refers to OpenStack release ```2023.1``` and respective Kolla-Ansible guide is available under the link ```https://docs.openstack.org/kolla-ansible/2023.1/user/quickstart.html)```. Please, note the ```2023.1``` discriminator of OpenStack release in the Kolla-Ansible URI.
3. The main goal of this guide is to instruct how to _**install**_ OpenStack cloud with Kolla-Ansible on Raspberry Pi cluster. For information on how to _**manage**_ OpenStack cloud using Kolla-Ansible, please refer to the original documentation of the Kolla-Ansible project.
4. **Assumption: for Kolla-Ansible 2023.1, the management host is implemented as Ubuntu 22.04 desktop in VirtualBox.** Other OS may work too after appropriate adaptations. However, there are also dependencies between releases. For example, Kolla-Ansible/OpenStack 2025.1 requires Ubuntu 24.04 or Debian 12 on the management host.

### 4.ii VM creation and basic configs

Here, we assume using a virtual machine, but things do not change if the management host is a physical machine.

  * Create `Ubuntu 22.04 LTS` VM as a desktop machine. We are using `Ubuntu 22.04 LTS` for Kolla-Ansible 2023.1 as the highest distribution supported by Kolla-Ansible 2023.1 (_Note: 'Ubuntu 24.04` is not supported by Kolla-Ansible 2023.1; but it becomes mandatory for Kolla-Ansible 25.1_). We have not tested other Linux distributions. Resource requirements of the VM are moderate (4GB RAM, 20GB disk, 1vCPU should be sufficient).
  * We use VirtualBox and configure the network card of the VM to work in ```Bridged``` mode. Nevertheless, NAT mode should work as well in a basic scenario, but using briged mode is more convenient in non-standard situations (e.g., when one needs to copy some files from the RPis to the management host). It is generally simpler when all components (cluster, management host, our physical host) work in the same subnetwork.
  * It's best when your physical host is connected in the same L2 network segment as the OpenStack cluster (in our lab setup, attach it to the Linksys router).
  * After launching the VM in VirtualBox, the copy-paste feature will probably not work. You will have have to install GuestAdditions. This can be done in a while. First follow the steps that follow.
  * Disable the automatic upgrade option in the VM; in the desktop, search for this setting in `Options`.
  * If the terminal suspends/does not open, the screen is flickering or the cursor takes the form of a black rectangle, disable Wayland display server protocol, see e.g., [this](https://linuxconfig.org/how-to-enable-disable-wayland-on-ubuntu-22-04-desktop)
  * Assign sudo privileges to your user (we assume username `ubuntu`)
  ```
$ sudo usermod -aG sudo $USER
  ```

   * Update/upgrade
  ```
$ sudo apt update && sudo apt upgrade
  ```

  * When using VirtualBox, install GuestAdditions - refer, e.g., to [this](https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/how-to-install-virtualbox-guest-additions-on-ubuntu-22-04.html?utm_content=cmp-true)
    
  * Enable passwordless sudo logging on the management host (required by Ansible) and reboot (reboot is necessary for guaranteeing Ansible permissions if you make all the installation in one attempt)
  ```
$ sudo apt install sshpass
$ sudo reboot
  ```

### 4.iii Docker installation

> [!Note]
> Docker can be installed according to [this original guide](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository). You can safely refer to that document. Below, we replicate it only for sake of completeness of this guide.

  * Add Docker's official GPG key:
  ```
$ sudo apt update
$ sudo apt install ca-certificates curl gnupg
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
  ```

  * Add the repository to Apt sources:
  ```
$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

$sudo apt update
$sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

$ sudo groupadd docker   <=== for any case
$ sudo usermod -aG docker $USER
$ newgrp docker          <=== adds the user to group docker in current shell without rebooting
  ```
  * Verify if docker works (check if you can run docker without sudo - should be possible)
  ```
$ docker run hello-world
  ```

## 5. Kolla-Ansible and OpenStack installation

> [!Note]
> In the following, we assume `ubuntu@labs:~/labs/ostack$` to be the working (current) directory. When copy-pasting, adjust the commands according to your environment.

### 5.i Kolla-Ansible installation

The installation procedure is in principle the same as in the original [Kolla-Ansible guide for the 2025.1 release](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart.html). Installation of [Kolla-Ansible 2023.1](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart.html) is very similar, but not identical, to 2025.1. One difference is that in 2023.1 one has to install Ansible explicitly by running a separate command while in 2025.1 this step is already included in the Kolla-Ansible script and you do nto bother with Ansible. Other differences are a direct consequence of changing the status of 2023.1 release to "unmaintained" without appropriate updates in the publicly available Kolla-Ansible guide and in one Kolla-Ansible configuration file (the term ```stable``` is used instead of ```unmaintained```). Corrective changes to respective instructions for 2023.1 are outlined in the text below.

One can use the original source document for 2025.1 and install on his own or take advantage of 100% ready-to-use commands documented below.

**THE TEXT BELOW IS NOW BEING REVISED FOR 2025.1 as the reference release. Temporarily mistakes are possible.**

  * Install Python packages, create and activate virtual environment
```
$ sudo apt-get update
$ sudo apt-get install -y git python3-dev libffi-dev gcc libssl-dev python3-venv
$ python3 -m venv kolla-2025.1   # you can use whatever name you want for the virtual environment
$ source kolla-2025.1/bin/activate

```

  - Hint: deactivate active venv / completely delete given venv:
```
$ deactivate
$ rm -r <venv-root-folder-name->
```

> [!Note]
> From now on, we work in virtual environment kolla-2025.1. The versions of all components installed are compatible with kolla-2025.1 and should be changed appropriately if you use other release of Kolla-Ansible (follow respective Kolla-Ansible _Quick start section_ for each release: [quickstart](https://docs.openstack.org/kolla-ansible/YOUR-RELEASE/user/quickstart.html)).

  * Install Kolla-Ansible in the active venv
```
$ pip install -U pip
      ==> for 2023.1 additionally run: pip install 'ansible>=6,<8'
          installing Ansible in 2025.1 is done by the Kolla-Ansible install script below so we ommit installing Ansible in case of 2025.1
$ pip install git+https://opendev.org/openstack/kolla-ansible@stable/2025.1
      ==> for 2023.1: pip install git+https://opendev.org/openstack/kolla-ansible@unmaintained/2023.1
          release 2023.1 has got the "unmaintained" status and one has to use a non-standard branch name to install it (see also the Warning below)
```

> [!WARNING]
> If you see error message similar to _```error: pathspec 'stable/2023.1' did not match any file(s) known to git ERROR! Failed to switch a cloned Git repo `https://opendev.org/openstack/ansible-collection-kolla` to the requested revision `stable/2023.1`.```_, do not panic. Your release has probably got the "unmaintained" status. You should go to [this repo](https://opendev.org/openstack/kolla-ansible) and check the name of the branch where your specific release is currently stored and where you will find its current branch name. Then change the name of your (supposed) branch (```@stable``` in the example) to the right one. To this end, edit local file: ```nano kolla-2023.1/share/kolla-ansible/requirements.yml```. Most probably you will have to change the name from ```stable/2023.1``` to ```unmaintained/2023.1``` (every release will eventually get the "unmaintained" status so treat the above as an example and adapt it appropriately to your particular case if needed).

### 5.ii Prepare configuration files for Kolla-Ansible

Now, generated by Kolla-Ansible, there are several configuraton files with default settings. We have to customize them (and even add new files) to make the configuration appropriate for our cluster.

```
# directory with customization files expected by Kolla-Ansible
$ sudo mkdir -p /etc/kolla
$ sudo chown $USER:$USER /etc/kolla
```

  * Copy globals.yml and passwords.yml to /etc/kolla directory, and kolla-ansible inventory file to our working directory. We will customize these file for our deployment soon.

    We remind ```~/labs/ostack``` is the working directory.
    
```
$ sudo cp -r kolla-2025.1/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
$ sudo cp kolla-2025.1/share/kolla-ansible/ansible/inventory/* .
```

  * Install Ansible Galaxy dependencies
```
$ kolla-ansible install-deps
```

  * Update Ansible configuration file ```ansible.cfg``` (it can be kept in the working directory or in directory ```/etc/ansible/ansible.cfg```)
```bash
$ tee ansible.cfg << EOT
[defaults]
host_key_checking=False
pipelining=True
forks=100
EOT
```

  * Once we have prepared networking on our RPis we should update the file ```/etc/hosts``` (on the management host) by adding our OpenStack hosts (RPis); this information will be used by Ansible. For example:
```
$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	labs

192.168.1.61    ost01
192.168.1.62    ost02
192.168.1.63    ost03
192.168.1.64    ost04
...
```

### 5.iii Configure Kolla-Ansible files for specific OpenStack depolyment

1. Prepare passwords.yml file

  * Generate passwors in file ```/etc/kolla/passwords.yml```

> [!Warning]
> In case of getting `PermissionError: [Errno 13] Permission denied: '/etc/kolla/passwords.yml'` change file permissions: `chown a+wr /etc/kolla/passwords.yml`. You can restore original permissions of `passwords.yml` after that.

```
$ kolla-genpwd
```
  * edit file ```/etc/kolla/passwords.yml``` and change the keystone admin password to human readable form (search ```keystone_admin_password```; you can set it simple, e.g., ```admin``` as in our case
```
$ sudo nano /etc/kolla/passwords.yml
```

2. Prepare inventory file ```multinode``` and feature configuration file ```globals.yml```

> [!IMPORTANT]
> Kolla-Ansible expects the user to assign OpenStack roles (```control```, ```network```, ```compute```, etc.) to hosts and specify those role allocations in inventory file. Our experiments have shown that the roles ```compute``` and ```network``` MUST be separated (i.e., allocated to different hosts) in clusters with 8GB RAM Raspberries. Otherwise the cluster becomes prone to out of memory issues which lead to severe instabilities. In all-in-one installation (one RPi in the cluster) the node runs close to out of memory very soon after creating the first CirrOS VM and OpenStack tends to crush. With more than one VM instance such cluster crushes immediately. Then, even if one can login to the hosts after rebooting and some Kolla-Ansible containers run in a healthy state, OpenStack as a whole is not functional anymore. Moreover, even in multinode cluster, when control and network functions run on the same host and compute function runs elswhere, the compute/network host saturates close to out of memory state after creating second CirrOS VM. So, even in Raspberry Pi multinode configuration the merging of ```compute``` and ```network``` should be avoided. These conclusions are reflected below in a sample fragment of the inventory file where ```copmute``` and ```network``` roles are assigned to different hosts (```ost4``` and ```ost3```, respectively).

  * file ```multinode``` is in the working directory

```
$ sudo nano multinode

# for every appearance of each node provide ansible directives ansible_user/password/become, e.g.:
[control]
ost04 ansible_user=ubuntu ansible_password=ubuntu ansible_become=true

[network]
ost03

[compute]
ost[01:03] ansible_user=ubuntu ansible_password=ubuntu ansible_become=true

[monitoring]
#monitoring01

[storage]
#storage01
```

> [!Important]
> Below, you will check cluster connctivity for Ansible. If Ansible can not reach your RPis _via_ ssh with messages `UNREACHABLE! => {
    "changed": false, "msg": "Failed to connect to the host via ssh...` then check file `/etc/hosts` for the presence of the resolution data. If previous installation of OpenStack failed and you have reinstalled the OS on the RPis, deleting file `~/.ssh/known_hosts` should solve the problem. 

```
# check if ansible can reach target hosts (our RPis):
$ ansible -i multinode all -m ping
```

  * file globals.yml is in `/etc/kolla/globals.yml`

> [!IMPORTANT]
> Take care of adjusting the attributes listed below. Activating an attribute requires uncommenting respective line (deleting the '#' sign opening the line). **do not touch the field ```#openstack_release: "some-identifier"```**.
>
> The value of **```kolla_internal_vip_address```** should be adjusted according to your environment. It must be an unused address in your network, and among others it will serve for accessing Horizon (OpenStack dashboard) to manage the OpenStack by the admin and to manage user stacks by the users (tenants).
>
> Enabling **```enable_neutron_provider_networks```** is not required in our case as we will use only flat provider network (we do not configure VLANs in our example setup). But you can have this feature enabled should you decide to introduce VLAN provider networks in the future.
>
> **```kolla_internal_vip_address```** MUST be anassigned (unused - free) IP address in your network (so, in particular, it will be different from the address assigned to veth0).


   **Attributes to update**

   You can copy-paste the following at the beginning of `globals.yml` (remembering to adjust `kolla_internal_vip_address`).
```
$ sudo nano /etc/kolla/globals.yml
---
# Valid options are ['centos', 'debian', 'rocky', 'ubuntu']
kolla_base_distro: "debian"
openstack_tag_suffix: "-aarch64"
kolla_internal_vip_address: "192.168.1.60"
network_interface: "veth0"
neutron_external_interface: "veth1"
enable_neutron_provider_networks: "yes"
# The following line must be commented out for 2023.1 where proxysql is not used at all. However,
# it is needed for 2025.1, otherwise there will be page size incompatibility between
# Raspberry Pi OS and the proxysql application.
enable_proxysql: "no"
```
  Note: more details about OpenStack networking with Kolla-Ansible can be found [here](https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html).

  * prepare file `/etc/kolla/conf/nova/nova-compute.conf`
    Here, we actually only request that the virtual machines that had been created before stopping the OpenStack are automatically brought to their previous state (e.g., ACTIVE) when OpenStack is restarted. This is convenient when we stop OpenStack (e.g., to shut down the cluster for some time) and start it again afterwards. The latter is described in section [6.i Shut down the cluster and start it again](#6i-shut-down-the-cluster-and-start-it-again).

> [!NOTE]
> File /etc/kolla/config/nova/nova-compute.conf can overwrite the settings given in file `/etc/kolla/globals.yml`. In particular, we can set non-default virtualization type and cpu mode and type in the ```[libvirt]``` section of file ```nova-compute.conf``` shown for Raspberry Pi 4 below, but commented out. It does works for RPi4, but as of this writing (July 2025) similar setting for Raspberry Pi 5 do not (it seems that libvirt does not currently support cpu model `cortex-a76` implemented by RPi 5). So basically KVM is preferred in our case. Luckily, KVM is the default choice in Kolla-Ansible when CPU architecture of hosts (declared by ```openstack_tag_suffix``` in ```globals.yml```) is ```aarch64```. In such a case (```aarch64```) libvirt driver in ```nova-compute``` imposes ```host-passthrough``` CPU mode (see [here](https://review.opendev.org/c/openstack/nova/+/530965) for more on that). All in all, we use KVM for both Raspberry Pi 4 and 5, and in this case it is enough to accept the default setting for ```nova_compute_virt_type``` in ```globals.yml``` (KVM is default) and not to specify ```[libvirt]``` section in ```nova-compute.conf``` at all.

<pre>
$ sudo mkdir -p /etc/kolla/config/nova
$ sudo tee /etc/kolla/config/nova/ << EOT
[DEFAULT]
resume_guests_state_on_host_boot = true

# The following settings are not required for KVM configuration (our choice). They are provided here for informational purposes only.
#for RPi 5 enable cortex-a76 <== maybe for the future, but currently theres a lack of support for cortex-a76 in 2023.1 and 2025.1 libvirtd
#for RPi 4 enable cortex-a72
#[libvirt]
# qemu works only for RPi 4; if you really want it enable all three settings that follow
#virt_type = qemu
#cpu_mode = custom
#cpu_models = cortex-a72
EOT                                                   
</pre>

 * prepare file `/etc/kolla/config/neutron/ml2_conf.ini`
<pre>
$ sudo mkdir -p /etc/kolla/config/neutron
$ sudo tee /etc/kolla/config/neutron/ml2_conf.ini << EOT
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan

[ml2_type_vlan]
network_vlan_ranges = physnet1:101:200

[ml2_type_flat]
flat_networks = physnet1
EOT
</pre>

> [!Note]
> This note is for students interested in what's going on above in the `ml2_conf.ini` file. Remember that in our case, we use OVS as a layer-2 agent in OpenStack (it's THE default SETTING in globals.yml).
>
> 1) Attribute `network_vlan_ranges = physnet1:101:200` will be included in the controller (network node) configuration file `/etc/neutron/plugins/ml2/ml2_conf.ini`. This is the allowed range of VLANS to be used by VLAN provider networks in our OpenStack ([see here](https://docs.openstack.org/neutron/queens/configuration/ml2-conf.html))
> 2) Based on the above, knowledge of the established Neutron operating principles and the configuration we provided in the `globals.yml` file for the physical port `neutron_external_interface: "veth1"`, Kolla-Ansible will configure the following bindings:
> * in the `ovs-vsctl` database, binding of `veth1` to `br-ex` with a command like: `ovs-vsctl add-port br-ex veth1` (br-ex is the name of a bridge preassumed by OpenStack)
> * in the `/etc/neutron/plugins/ml2/openvswitch_agent.ini` file, the binding of the above-mentioned external bridges (br-ex, br-ex1, ...) to the physical networks: `bridge_mappings = physnet1:br-ex,physnet2:br-ex2` (in our case, there's no `physnet2:br-ex2` - we'd need another "physical" port, e.g., `veth2`, for that.
> (Neutron assumes interface names in the form and order `br-ex`, `br-ex2`, `br-ex3`, ..., and Kolla-Ansible associates them with the names `physnet1`, `physnet2`, ... based on the order of their appearance in the list define by attribute `network_vlan_ranges = ...`. For example, if we additionally used `physnet2` in our cluster, `network_vlan_ranges` could look like this: `network_vlan_ranges = physnet1:101:200,physnet2:201:300`, [see here](https://docs.openstack.org/ocata/config-reference/networking/samples/openvswitch_agent.ini.html))
> 3) We will use the name physnet1 (and optionally physnet2, etc.) from the admin level when creating a provider network. Respective command specifies the physical network (thus indirectly references a specific bridge, `br_ex`, and thus a physical interface, `vlan1`) on which the declared provider network (flat or VLAN) is to be created (you will have to use VLAN IDs from the pool associated with `physnet1`, etc.).
> 4) Note: Binding physical interfaces (they are `veth0, veth1,...` in our case, but could be `eth0, eth1,...` in a physical infrastructure) to internal OpenStack bridges is the responsibility of the OpenStack installer, in our case Kolla-Ansible, but it could be human. That is why we have to specify the roles of the individual interfaces in our installation in the `globals.yml` file so that Kolla-Ansible knows exactly how to bind them. Important to nbote is that all these configurations are performed by the OpenStack administrator and are hidden from regular users. (tenants).
>
> More about bridge mapping in OpenStack:
> * https://docs.redhat.com/en/documentation/red_hat_openstack_platform/10/html/networking_guide/bridge-mappings#maintaining_bridge_mappings
> * https://docs.redhat.com/en/documentation/red_hat_openstack_platform/10/html/networking_guide/bridge-mappings#bridge-mappings
> * https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html#openvswitch-ml2-ovs

### 5.iv Deploy OpenStack

In case of kolla-ansible below has problems with ssh to reach your RPis ("WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!"), check file /etc/hosts for the presence of resoultion data. If you have reinstalled the OS on the RPis, you have to **do delete** `rm ~/.ssh/known_hosts`.

```
$ kolla-ansible bootstrap-servers -i multinode 
$ kolla-ansible prechecks -i multinode
# kolla-ansible deploy -i multinode
```

> [!IMPORTANT]
> If some node fails to complete a task related to restarting a container this may result from conectivity problems to `quay.io` container repository. To recover from this just run `kolla-ansible deploy` again. Below is an example of such recoverable failue. If the problem persists check `quay.io` health in [this portal](https://statusgator.com/services/quay) and if there is a clear evidence of deteriorations try to re-run `deploy` after some time. Another (more advanced) option is to build images of kolla containers on your own and keep them in a local repository according to [this guide](https://docs.openstack.org/kolla/latest/admin/image-building.html) (to this and, you will need to install and run your own repo, reserve some GB of storage for images; you will also need to adapt the commands from the linked guide to aarch64).
> ```
> RUNNING HANDLER [neutron : Restart neutron-openvswitch-agent container] **************************************************************
> changed: [ost01]
> fatal: [ost02]: FAILED! => {"changed": false, "msg": "Unknown error message: dial tcp: lookup quay.io on 192.168.10.1:53: read udp 192.168.10.22:47363->192.168.10.1:53: i/o timeout"}
> changed: [ost03]
> ```
>
> We believe it is good practice to monitor memory usage on hosts (remote terminal and `htop` command does a good job of this), especially on the control node. Memory usage crossing 7GB suggests that OpenStack may crush soon. You can try to remedy this by stopping unnecessary workload (VMs). What may work if there is no such spare workload is stopping OpenStack using command `kolla-ansible stop-containers` and resuming the cloud with command `kolla-ansible deploy-containers` (refer to [section 6.i](#6i-shut-down-the-cluster-and-start-it-again) for more details). This latter method (stopping/starting) takes quite a bit of time (say, 30 minutes in total), but it's worth it because it gives you a chance to avoid installing the cluster from scrach (and sometimes even reinstalling the operating system on the failed node(s)).

### 5.v Postdeployment and the first instance

#### 5.v.a Postdeployment

Postdeployment includes installing OpenStack CLI tool, running additional `post-delopy` script to generate configuration file `clouds.yaml` containing credentials to use OpenStack CLI tool, and copying `clouds.yaml` to appropriate locations.

  * Install `python-openstackclient` to access OpenStack commands in the console
    ```bash
    $ pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2025.1
           ==> for the unmaintained release 2023.1: 
               pip install python-openstackclient -c https://opendev.org/openstack/requirements/raw/branch/unmaintained/2023.1/upper-constraints.txt
    ```

  * Run post-deploy script, create appropriate directories and copy cloud.config file (it will be needed by install-runonce stript in a while to create your first instance): 

    ```


    # for 2025.1 (a form slightly different from 2023.1 because inventory folder /etc/kolla/ansible/inventory has been
    # introduced in 2025.1 and script post-deploy requires this inventory as command parapeter, but this change has not been
    # reflected in the original Kolla-Ansible guide); we can point to inventory file located elsewhere, e.g., in our
    # working directory as assumed below:
    $ kolla-ansible post-deploy -i multinode-2025.1
    
    # for 2023.1 (existence of inventory file in working directory is checked so do do not have to provide inventory file name here) 
    $ kolla-ansible post-deploy
    
    $ sudo mkdir ~/.config 
    $ mkdir ~/.config/openstack
    #if you see error notification about unreachability of a file, do: $ chmod -R u+r <unreachable-directory-name>
    $ sudo cp /etc/kolla/clouds.yaml /etc/openstack/clouds.yaml
    $ cp /etc/kolla/clouds.yaml ~/.config/openstack/clouds.yaml
    ```
    
#### 5.v.b First checks - create the first instance

   Your first instance will be created from openstak client command line. First, you will use a prepared script to create tenant network and other artifacts (e.h., VM image to create form) needed to create instances. Then, you will enable python-openstack client tool and create the instance from command line using openstack command line client (similar to creating pods in Kubernetes using kubctl).

  * use init-runonce script to create external network, subnetwork, router, download the image of cirros VM, and create a couple of flavors of VM in the admin project
    - init-runonce script files for various kolla-ansible/OpenStack releases and tuned for our Raspberry Pi environment are available in this repo. Select appropriate <kolla-ansible-release> and <vm-os-name>, possibly edit the file to change some settings according to your environment (e.g., external network address range). The file name format of those scripts is as follows:
   
      `init-runonce.<kolla-ansible-release>.<flavor-os-name>`
      
      **Remark 1**: the original version of init-runonce script is generated by the command `kolla-ansible post-deploy`. It is stored as /path/to/ouur/venv/share/kolla-ansible/init-runonce. We have to use slightly modified version of the script to match the requirements/limitations of our Raspberry Pi platform (e.g., we need aarch64 image, we can afford only quite small VM flavors, etc.).
      
      **Remark 2**: currently, the tested and recommended settings are `<kolla-ansible-release>=2025.1` and `<flavor-os-name>=cirros`, so to run the recommended version use file `init-runonce.2023.1.cirros`:

      - run the script
      ```bash
      $ ./init-runonce.2025.1.cirros
      
      # or (for 2023.1)
      $ ./init-runonce.2023.1.cirros
       ```
      
> [!Note]
> Script `init-runonce` illustrates the use of several `openstack client` commands in bash. It is recommended to analyse it, possibly even echo-ing openstack client commands to see how they look like in plain. It may help you write your own commands if such a need arises in the future. Remember that `init-runonce` is designed to be run "as is" only **once** and subsequent runs will generate errors because of all constructs having already been created. In such a case free experimenting with it should depend on echo-ing openstack commands (to display them as ready-to-run strings) at the same time suppressing in the script the execution of the commands in OpenStack.

After running `./init-runonce.20xy.z`, the following OpenStack objects are created and ready for use: the external and tenant networks, VM image, several VM flavors (we have adjusted their properties for Raspberry Pi) and the default security group settings. The first server instance can now be created in the following two steps:

  * source `admin-openrc.sh` script to enable python-openstackclient
    ``` bash
    $ source /etc/kolla/admin-openrc.sh
    ```
  * run the VM from openstack command line client
    ```
    openstack --os-cloud=kolla-admin server create --image cirros --flavor m1.tiny --key-name mykey --network demo-net cirros1
    ```
    - the following is only for our record, do not run it unless you have respective images (hou have to prepare them by yourself)
      ```bash
      $ ./init-runonce.2025.1.alpine
      openstack --os-cloud=kolla-admin server create --image alpine --flavor m1.large --key-name mykey --network demo-net alpine1
      ```
      ```bash
      $ ./init-runonce.2025.1.ubuntu
      openstack --os-cloud=kolla-admin server create --image ubuntu --flavor m1.medium --key-name mykey --network demo-net ubuntu1
      ```
      
  * check the status of the instance
    ```
    $ openstack server list
    ```

  * now you can ssh to the instance; both password and key authentication work for ther cirros (the key was installed in the `openstack create` command you run above); the user is `cirros` and the passoword is `gocubsgo`

## 6. Managing your cluster 

### 6.i Shut down the cluster and start it again

#### 6.i.a Shut down the cluster

This will stop the entire cluster (to eventually power off the RPis) without damaging anything. After that it will be possible to power on the RPis and resume complete OpenStack recovering also the state of all your instances.

  * Stop containers
    ```
    $ kolla-ansible stop -i multinode --yes-i-really-really-mean-it
    ```
> [!Warning]
> If Ansible issues a notification as below informing about certain hosts being ureachable, run **`kolla-ansible stop`** command again.
> ```
> TASK [Gather facts] **************************************************************************************************
> [WARNING]: Unhandled error in Python interpreter discovery for host ost03: Failed to connect to the host via ssh: ssh:
> connect to host > ost03 port 22: Connection refused
> fatal: [ost03]: UNREACHABLE! => {"changed": false, "msg": "Data could not be sent to remote host \"ost03\". Make sure
> this host can be > reached over ssh: ssh: connect to host ost03 port 22: Connection refused\r\n", "unreachable": true}
> ```
    
  * Power off RPis
    - write and run a bash command containing the following script (we assume $CLUSTER_INVENTORY_FILE is a parameter passed to the command; you will probably want to set it to `multinode`)
    ```bash
    #!/bin/bash
    # check the following links for the ansible command options used:
    # https://www.jeffgeerling.com/blog/2018/reboot-and-wait-reboot-complete-ansible-playbook
    # https://www.middlewareinventory.com/blog/ansible_wait_for_reboot_to_complete/
    CLUSTER_FILE="multinode"
    ansible all -i $CLUSTER_INVENTORY_FILE --limit 'baremetal' -b -B 1 -P 0 -m shell -a "sleep 5 && shutdown now" -b   
    ```
    
 #### 6.i.b Start the cluster

This will restart the entire cluster after it has been stopped and resume OpenStack recovering also the state of all instances.
 
  * Power on RPis

  * On the management host: activate the virtual environment of your Kolla-Ansible installation and resume OpenStack operation by executing:
    ```
    $ kolla-ansible deploy-containers -i multinode 
    ```

### 6.ii Destroy your cluster

To reinstall your cluster in case of failure, first destroy current installation (this cleans RPis from all Kolla-Ansible/OpenStack artifacts). To do this, it is best to first stop or remove all running virtual machine instances in the cluster and only then run the command:

```
$ kolla-ansible destroy --yes-i-really-really-mean-it -i multinode
```

## 7. VLAN provider networks - part 2 (enabling and using VLAN provider networks)

In this section, we describe how to enable VLAN provider networks by modifying the setup from section [3.iii VLAN provider networks - part 1](#using-vlan-provider-networks---part-1-rpi-network-configuration-for-flat-network). Then we show how such networks can be used.

Note that we'll be reconfiguring a running OpenStack instance. However, all of the configurations below can be performed just as easily in section [3.iii VLAN provider networks - part 1](#3iii-vlan-provider-networks---part-1-rpi-network-configuration-for-flat-network) so before running the Kolla-Ansible commands. Please note also that all VLAN IDs used here are examples only.

### 7.i Setting VLANs for provider networks

#### 7.i.a Setting VLANs in the physical network (the TP-Link switch)

In the `VLAN -> 802.1Q VLAN Configuration` tab, enable VLAN support using the `Enable` switch at the top, and then add specific VLANs individually on the switch ports. In our example below, we create a VLAN with VLAN ID 101, enabling it on the switch ports that connect our RPis (note that port 5 in TP-Links switch is the "WAN" port and it connects to your router (Linksys or TOTO-Link in our case). This way, VLAN 101 will be present on the control, network and compute hosts in the cluster, and once it is created, the corresponding provider network will have a similar coverage. Repeat this operation for the remaining VLANs we want to have in the cluster. The set of VLANs configured on the TP-Link switch should be consistent with VLAN declarations in the replacement files in the next section (actually, we define three VLANS there with VLAN IDs 101, 102, 103).

<p align="center">
 <img src=images/tplink-vlan101.png width='60%' />
</p>

#### 7.i.b Setting VLANs on the RPi hosts

On each RPi, replace some files in the `/etc/systemd/network` directory with versions that define the VLAN configuration. New versions are available in this repository in the `vlanned/etc/systemd/network` directory - upload them to your RPis. You are encouraged to review these files to learn how persistent configuration of VLANs can be achieved in Linux. The files contain explanation of each construct used. After uploading them, you can restart the RPi with the `reboot` command, or if you attach to your RPis using WiFi you can restart only the networking with the following commands:

```
$ ip link set down brmux
$ ip link del dev brmux
$ systemctl restart systemd-networkd
```

After completing the above steps, VLANs 101, 102 and 103 will be activated in the OpenStack infrastructure, ready for building VLAN provider networks on top of them.

#### 7.i.c Separating the flat network using a VLAN

As an additional exercise, you can isolate the traffic of the flat provider network in the physical part of the L2 segment (i.e., between the TP-Link switch and the RPis) using a VLAN. That is, we want the untagged traffic corresponding to the flat provider network in our cluster to be carried in a dedicated VLAN between the TP-Link switch and the `brmux` virtual devices in the RPis. Note that such a VLAN will be distinct from the VLANs created for VLAN provider networks. Its traffic will be unatgged when leaving TP-Link via Port 5, or when leaving `brmux` via devices `veth0br` or `veth1br` (of course, untagged traffic in the opposite direction appearing on TP-Link and `brmux` devices will be tagged with the ID of our additional VLAN). Effectively, OpenStack and the Linksys router will perceive the flat provider network unchanged (its traffic untagged), but at the same time this traffic will be isolated using a VLAN on the remaining part of the physical network.

The practical benefit of using this extra encapsulation will be increased security in the physical part of our network infrastructure (not a fundamental change, but worth mentioning). From an educational point of view, the procedure described here allows for a better understanding of the essence of the VLAN provider network (described in the previous two subsections).

Follow the steps described below.

* Create the additional VLAN in the TP-Link switch (we set VLAN ID = 2 but anthing different from the VLAN IDs reserved for provider networks will be fine). As before, you do this in the `VLAN -> 802.1Q VLAN Configuration` tab.

<p align="center">
 <img src=images/tplink-vlan2.png width='60%' />
</p>

* Additionally, in the `VLAN -> 802.1Q PVID Setting` tab set the VLAN ID to tag traffic coming from the router (Linksys) to the switch, i.e., on Port 5 of the switch.

<p align="center">
 <img src=images/tplink-pvid2.png width='60%' />
</p>

As before, you should now be able to access OpenStack via the dashboard or the OpenStack CLI.

### 7.ii Creating and using VLAN provider networks

TODO

#### 7.ii.a Provider network dedicated to a tenant

TODO

#### 7.ii.b External network using VLAN provider network

TODO

## 8. ADDENDUM - accessing the cluster using VPN

You can set a VPN to access the cluster remotely.

You can use a VPN platform of your choice. In our lab, we are using [Zero Tier](https://my.zerotier.com/) VPN solution. You can install it according to the original instructions. Polish version of the guide (customized for our environment) is available [here](https://github.com/dbursztynowski/k3s-taskforce/blob/master/zt-manual.md#poradnik-po%C5%82%C4%85czenia-klastra-z-sieci%C4%85-zerotier-vpn).

If you have a dedicated computer to host the VPN connection (jump host), your local router has been configured, and your Raspberry Pi is up and running with the newly installed SD cards, you will be able to complete all the remaining steps in this guide. Note that such a jump host has to be connected to the L2 network segment of your OpenStack.

If you do not have a spare machine to set VPN jump host than you can assign selected OpenStack node (selected Raspberry Pi board) for this role. However, make sure you have assigned a board that will be pure `compute node` in your OpenStack (do not overload OpenStack control nodes). Mind also that in such a case if you shut down the cluster completely, your jump host RPi will become inaccessible, too. As a consequence, the only option to start the cluster again will be to perform a local restart. In contrast, using a separate machine to host the VPN connection allows one to start the cluster remotely (in either case, starting the cluster can be done by accessing the TP-Link switch dashboard and restarting the switch using the `System->Reboot` option). So if you plan to take the cluster down from time to time, it's better (and safer) to use a separate machine as the VPN jump host.

Enjoy working with your OpenStack remotely!


