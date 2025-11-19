# Installation of OpenStack on Raspberry Pi

## Executive summary

In this repo, we describe how to install OpenStack on Raspberry Pi cluster using Kolla-Ansible. The main field of application of such clusters is instructional tools for teaching. While you can find other guides on the Internet on this topic, they are generally based on old versions of OpenStack and do not cover a number of important details that would make an OpenStack Raspberry Pi cluster run reliably.

A number of OpenStack cluster configurations have been tested thoroughly. Each configuration was determined by the Raspberry Pi board type (4B or 5), Kolla-Ansible OpenStack release (2023.1 or 2025.1), virtualization type (Qemu emulation or KVM). In all configurations, the RPis run under Raspberry Pi OS (Debian 12 Bookworm derivative). We confirm that KVM virtualization works excellent on RPi 4 and RPi 5 for both tested releases of Kolla-Ansible/OpenStack (2023.1 and 2025.1). Qemu only works well on RPi 4, and the VMs are more than twice as slow compared to when host-passthrough KVM is used. Qemu emulation has failed to work on RPi 5 because the Cortex-A76 processor model is not supported by the libvirtd package installed by Kolla-Ansible (we drew this conclusion based on the analysis of libvirtd logs). Considering the above, KVM is the recommended virtualization option for OpenStack deployments on the Raspberry Pi platform. Moreover, in all tested cases, the OpenStack control node was deployed on an RPi board with 8GB of RAM, and we had to increase the swap memory size on this node from 512MB (the default value in the Raspberry Pi OS) to a much safer capacity of 4 GB.

In summary, both the Raspberry Pi 4 and 5 are suitable for setting up small and cheap, bare metal OpenStack clusters for education purposes. This guide describes the complete installation procedure for release 2023.1 and 2025.1 using Kolla-Ansible, assuming 2025.1 to be the reference version (differences applicable in release 2023.1 are discussed directly in the text). Updates will be added to this guide as new findings, propositions or recommendations emerge.

> [!NOTE]
> At the time of writing this guide, the latest stable version is Kolla-Ansible 2025.1.
> Update: although OpenStack 2025.2 was released in October 2025, Kolla-Ansible release _latest_ (2025.2) is currently (November 2025) known to contain bugs and we do not recommend using it for educational purposes.

## Table of contents

1. [Introduction](#1-introduction)
2. [Platform components](#2-platform-components)
3. [Local network and Raspberry Pi preparation](#3-local-network-and-raspberry-pi-preparation)
   1. [Local network preparation](#3i-local-network-preparation)
   2. [RPi system configuration](#3ii-rpi-system-configuration)
   3. [RPi network configuration - pure flat provider network](#3iii-rpi-network-configuration---pure-flat-provider-network)
   4. [VLAN provider networks - part 1 (RPi network configuration for a flat network)](#3iv-vlan-provider-networks---part-1-rpi-network-configuration-for-a-flat-network)
5. [Management host preparation](#4-management-host-preparation)
   1. [General notes](#4i-general-notes)
   2. [VM creation and basic configs](#4ii-vm-creation-and-basic-configs)
   3. [Docker installation](#4iii-docker-installation)
6. [Kolla-ansible and OpenStack installation](#5-kolla-ansible-and-openstack-installation)
   1. [Kolla-Ansible installation](#5i-kolla-ansible-installation)
   2. [Preparing configuration files for Kolla-Ansible](#5ii-preparing-configuration-files-for-kolla-ansible)
   4. [Configuring Kolla-Ansible files for specific OpenStack depolyment](#5iii-configuring-kolla-ansible-files-for-specific-openstack-depolyment)
   5. [Deploying OpenStack](#5iv-deploying-openstack)
   6. [Postdeployment and the first instance](#5v-postdeployment-and-the-first-instance)
7. [Managing your cluster](#managing-your-cluster)
   1. [Shut down the cluster and start it again](#6i-shut-down-the-cluster-and-start-it-again)
   2. [Destroy your cluster](#6ii-destroy-your-cluster)
8. [VLAN provider networks - part 2 (enabling and using VLAN provider networks)](#7-vlan-provider-networks---part-2-enabling-and-using-vlan-provider-networks)
   1. [Setting VLANs for provider networks](#7i-setting-vlans-for-provider-networks)
      1. [Setting VLANs in the physical network (the TP-Link switch)](#7ia-setting-vlans-in-the-physical-network-the-tp-link-switch)
      2. [Setting VLANs on the RPi hosts](#7ib-setting-vlans-on-the-rpi-hosts)
      3. [Separating the flat network using a VLAN](#7ic-separating-the-flat-network-using-a-VLAN)
   3. [Creating and using VLAN provider networks](#7ii-creating-and-using-vlan-provider-networks)
      1. [Provider network dedicated to a tenant](#7iia-provider-network-dedicated-to-a-tenant)
      2. [External network using VLAN provider network](#7iib-external-network-using-vlan-provider-network)
9. [ADDENDUM - accessing the cluster using VPN](#8-addendum---accessing-the-cluster-using-vpn)
   
## 1. Introduction

The scope of application of our clusters is teaching. A cluster of this type allows us to present/explain various features/concepts of OpenStack, some of them being hard or impossible to show using AIO or virtualized OpenStack setups. Many of such features are related to the administration of OpenStack data centre - a domain of activity that is hidden from regular users in "true" DCs. For example, the management of provider networks where the admin needs to configure VLANs in the physical network of the data centre and declare them in OpenStack and Kolla-Ansible config files. Considering that the time needed to practice and learn even basic things is non negigible, a decent amount of resources is needed to serve a dozen or more student teams in limited time. Our approach allows us to achieve these goals without having to allocate dozens of physical servers, each worth thousands of $, for extended periods of time.

This guide covers the steps to create your first virtual machine in a newly deployed RPi OpenStack cluster. This includes: **system configuration on the Raspberries** (OS installation, setting needed permissions, configuring network settings that are required by Kolla-Ansible, etc.), **configuration of the management host** (this is a separate host where we install Kolla-Ansible environment, run Kolla-Ansible installation commands, and from where we manage OpenStack using a command line tool), **installation and customization of Kolla-Ansible** and customization of its configuration files on the management host to meet the needs of our deployment, **actual deploymnet of OpenStack** using Kolla-Ansible, then **creation of basic elements of virtualized infrastructure** (VM image of CirrOS, external and tenant networks, public and private routers, security groups, etc.) and finally **creation of the first VM instance** in our cloud. We also show how to provision **VLAN-based provider networks** in our OpenStack and how to set up VPN access to the cluster using the _ZeroTier_ platform. The above steps are detailed in the remainder of this document.

This guide assumes a specific combination of network hardware and network topology in the cluster. However, it should also be applicable to other configurations provided the required networking constructs (e.g., VLANs) are supported.

> [!NOTE]
> * There are other minicomputer platforms available on the market that may seem attractive for implementing OpenStack educational deployments, to mention Turing Pi 2 / 2.5, DeskPi Super6C or Sipeed's NanoCluster. However, Turing Pi and DeskPi Super6C support Raspberry Pi CM4 modules and not CM5 (Turing Pi 2.5 supports also other, more powerfull modules, still not Raspberry Pi CM5). They also result in higher budget requirements for the same educational result. The Sipeed NanoCluster may be attractive because it comes with a managed switch and a router on board, which can simplify network configuration; we have not evaluated overall cost of a setup based on it, though. Moreover, all three alternatives may be not so easily available in various countries and it may be safer to adhere to more conservative derivatives of [Raspberry Pi Dramble](https://www.pidramble.com/) :grin:.
> 
> * It's also worth noting that Raspberry Pi clusters make an excellent base for Kubernetes experiments. In our lab, we set [bare-metal K3s clusters](https://github.com/dbursztynowski/k3s-taskforce), which works perfectly for the Raspberry Pi.

## 2. Platform components

All procedures described in this guide assume compliance with the setup options for HW and SW in the cluster as specified below. Other configurations may also work perfectly (e.g., those with more hosts will definitely work), but we haven't tested them.

1. Raspberry Pi 4 and Raspberry 5
   * tested options: [2x4GB RAM + 2x8GB RAM] or [3/4x8GB RAM] per cluster (uniform 8GB RAM clusters are better)
     * mimimal possible: [1x4GB RAM + 1x8GB RAM] (a minimal cluster, enough to create a single CirrOS virtual machine and nothing more)
     * OpenStack control/network nodes run on the same 8GB RPi host
     * ideally, you would have one RPi 5 with 16GB of RAM in the cluster to run the OpenStack control and network node (this eliminates swap memory size issues, see step 9 in section [RPi system configuration](#3i-rpi-system-configuration))
   * all Pi are equipped with 32GB SD disk
   * Note: a single 8GB RAM RPi host in all-in-one setup of Kolla-Ansible OpenStack is barely able to host OpenStack in absolutely minimal configuration. In such a cluster, one can create a single CirrOS instance with 512MB memory, but the cloud becomes unstable. We have seen many times that even such a simple configuration ends in failure soon after instantiating the VM. And OpenStack all-in-one system immediately crashes due to lack of memory when a second similar CirrOS instance is created, unless you increase the swap memory size. This last option (increased swap memory), however, carries the risk that everything will become  v e r y  slow. That is the reason why we consider the dual-host configuration to be the minimum possible.
   * An illustrative diagram of the physical setup of the cluster with Linksys WRT54-GL as the local router is shown below. Notice that the Linksys router connects to the TP-Link switch on the WAN port of the switch (Port 5 in the TP-Link management dashboard).

<p align="center">
<img src=images/cluster.jpg width='65%' />
</p>
   
2. SW
   * OS: Raspberry Pi OS Lite 64bit (a port of Debian 12 Bookworm with no desktop environment).
     - Ubuntu 24.04 LTS for Raspberry Pi may work as well. It comes with netplan and systemd-networkd as default network configuration tools, which should simplify certain installation steps from this guide.However, we haven't tried using Ubuntu on our RPi OpenStack clusters yet. The main reason is that the Kolla-Ansible container images for the latest OpenStack releases and AARCH64 (Raspberry Pi processors) available in the default repository (quay) are built for Debian, not Ubuntu.
   * Linux network configuration tools: netplan and systemd-networkd (they are not default on Debian but we use them to comply with the policies adopted in our labs).
   * Kolla-Ansible 2023.1 or 2025.1. 
3. Physical network:
   * the RPis are equipped with 802.3af/at PoE HAT from Waveshare (PoE is optional but simplifies cluster wiring) 
   * they are powered form TP-Link TL-SG105PE switch (it supports 802.1Q which can be used to set multiple VLAN provider networks in OpenStack)
   * TP-Link switch is connected to a local router with DHCP enabled to isolate the network segment of OpenStack DC from the rest of the local network infrastructure
   * **reserve a pool of IP addresses for the use by OpenStack** on your local router as described in subsection 3.i.
4. Virtualization
   * we have successfully tested qemu and KVM on Raspberry Pi 4 and KVM on Raspberry Pi 5 (qemu does not work correctly on Raspberry Pi 5 with standard Kolla-Ansible installation).

> [!NOTE]
> * there are various types of PoE HAT for Raspberry Pi 4/5 and various PoE switches available on the market that also will work for you; one should only take care of the required power budget of the switch (please note that the required power budget will be different for the Raspberry Pi 4 and 5 platforms).
>
> * for education purposes, we use configurations with at least 3 RPis and a managed switch (802.1Q) in the cluster to demonstrate how VLAN-based provider networks can be used in OpenStack; this is impossible to show using AIO (all-in-one) OpenStack setups (and may be very tricky when using virtual machines as OpenStack hosts). But if one does not plan to deploy VLAN provider networks, unmanaged switch can be used as well.
>
> * in this guide, we will assume the cluster contains a TP-Link switch and a Linksys router, and we will use these names in the text; map these names to your configuration if you are using different hardware in your OpenStack data center and your local network, respectively
>
> * other details that may be relevant are explained in the description that follows
  
## 3. Local network and Raspberry Pi preparation

In section 3.i we describe the basic configurations necessary for the physical devices of the cluster network.

In sections 3.ii, 3.iii, and 3.iv, configurations needed for your RPis are described. These configurations have to be applied independently to each RPi host in your cluster. They are given in a form corresponding to a manual procedure. However, you can automate many steps using bash scripts or other tools if you wish (_Note: sometimes a restart of the Raspberry Pi is required, so you'll likely need to create some semi-automatic installation scripts. Alternatively, you can use Ansible to completely automate the installation. However, discussing the details of such solutions is beyond the scope of this guide._) The process is split into two phases: system configuration (installs, upgrades, etc.; subsection 3.ii) and host network configuration (enabling networkd, installing netplan, configuring persistent virtual devices; subsection 3.iii). In addition, subsection 3.iv describes the initial configurations of the RPis that are recommended if we plan to use VLAN provider networks in the cluster.

### 3.i Local network preparation

It is best to start by configuring the appropriate IP subnet parameters (subnetwork mask, DHCP address range) on your local router (Linksys router in the figure from section 2). Throughout this guide, we assume the mask is 192.168.10.0/24.

Then reserve a contiguous pool of IP addresses for use by OpenStack on your local router (Linksys); 16 addresses will be sufficient for our purposes. The term **reserved** means here these addresses **must not** be managed by the DHCP server on your local router, but they should be routable in your local network (typically, it suffices to define a DHCP address pool such that it does not contain that reserved address pool). Some of these reserved addresses will later be assigned by you to the RPis using netplan configuration file (see [Configuring the network (flat)](#configuring-the-network-flat) in subsection 3.iv). One address will be allocated as the so-called `kolla_internal_vip_address` (see [section 5.iii](#5iii-configuring-kolla-ansible-files-for-specific-openstack-depolyment)). The remaining addresses will be used to create OpenStack `floating IP addresses`. In OpenStack, floating IPs allow access to virtual machines from outside the cloud. (We assume you will not create more than 11 virtual machines, so the suggested pool of 16 reserved addresses for the cluster will be more than enough.)

You will access the devices in the cluster (RPis, TP-Link) using ssh. The IP addresses to use can be checked in the Linksys panel `Status->Device List`:

<p align="center">
 <img src=images/device-list-linksys.png width='62%' />
</p>

Make sure your TP-Link switch is set to the factory settings. This can be performed by holding the reset button in a small hole on the rear panel of the switch for a couple of seconds. It is important that there are no VLANs defined in the switch. You can check it the `VLAN->802.1Q VLAN Configuration` tab. The following configuration is correct (otherwise all existing VLANs should be deleted):

<p align="center">
   <img src=images/tplink-factory.png width='60%' />
</p>

> [!Note]
> In our TP-Link switches, after restoring factory settings, we log in with username=`admin`, password=`admin` and the password change is forced upon the first login. The management portal of the switch is intuitive. To access it, you can use a Web browser and (on Windows) _Easy Smart Configuration Utility_ application. You can download the guides and the utility from [here](https://www.tp-link.com/pl/support/download/tl-sg105pe/).

Now you can proceed to subsection 3.ii.

### 3.ii RPi system configuration

Generally, when choosing the installation environment, we follow the guidelines contained in the [Kolla-Ansible support matrix](#https://docs.openstack.org/kolla-ansible/2025.1/user/support-matrix.html) .

1. Flash the Raspberry Pi OS Lite 64bit (a port of Debian 12 Bookworm with no desktop environment) onto microSD card. We use Raspberry Pi Imager for that, but other tools can be used as well.
   * make sure password authentication for ssh access is enabled (the instructions given in the following fit this authentication method); this allows one to avoid the use of authentication keys in Ansible (not recommended for production, but simpler)
  
<p align="center">
 <img src=images/enablessh.png width='30%' />
</p>

   * it is recommended to set the host name, user name and password to what you will use later in Kolla-Ansible playbooks. In the examples below, we set "ubuntu" for both the user name and the password, and use the convention ost01, ost02, ... to set the host name of our RPis.
  
3. After switching on the RPis, SSH to each of them using the credentials from step 1 above. Their IP addresses can be found in the management panel of your local router (e.g., in configurations with a Linksys router, check the ```Device list``` panel in the Linksys router GUI). These addresses are single-use and will later be replaced with permanent addresses during network stack configuration on each RPi host (you will draw those fixed IP addresses from address pool reserved as described in subsection 3.i).

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

This section only applies when you are going to use 8GB RPi as the OpenStack control/network node. If you are going to use 16GB RAM RPi 5 for the control/network node in your OpenStack then you can stay with the default swap settings and skip the rest of this section. However, since we have not tested the 16GB RAM option so far, we still recommend monitoring memory/swap usage at least during the initial cluster startup period. This will ensure that resource usage is under control and will not cause any problems.

> [!IMPORTANT]
> This configuration applies only to the host that will combine OpenStack roles of "control node" and "network node" in your OpenStack cluster. In this guide, we assume these roles will be assigned to the host with name `ost04`. In [section 5.iii](#5iii-configuring-kolla-ansible-files-for-specific-openstack-depolyment) you can check how we assign roles to hosts in Kolla-Ansible inventory file `multinode`. Remaining hosts can be left with default swap memory size settings (512MB in the Raspberry Pi OS), but allocating more space will not be a mistake.
>
> Kolla-Ansible allows one to separate the `control` and `network` functions by assigning them to different hosts in Ansible inventory. If you are curious, you can give it a try. However, remember that while such separation reduces the maximal memory occupancy of the hosts, it does not relieve you of neither the need to increase the swap memory size nor the responsibility to monitor memory usage on the `control` and `network` hosts.

  * follow the instructions below; they are drawn from [this guide](https://itsfoss.com/pi-swap-increase/). The highest usage of swap memory that we have observed so far is 2.5GB. Below we set the size to **`4096`** which should provide a safe margin.
  ```
sudo dphys-swapfile swapoff
sudo nano /etc/dphys-swapfile
CONF_SWAPSIZE=4096   # <=== desired size, safe
CONF_MAXSWAP=4096    # <=== should be no less than CONF_SWAPSIZE (limit)
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
swapon --show        # ensure it is now active
sudo reboot          # just for any case
  ```

### 3.iii RPi network configuration - pure flat provider network

In this section, we describe how to configure networking in our OpenStack providing support only for flat provider network. This is the simplest option regarding network configuration in OpenStack, still sufficient to demonstrate many OpenStack features. Introducing VLAN provider networks requires additional configurations in L2 of the data center. In our case, this concerns TP-Link switch and the internal network devices in our RPis: ```eth0```, ```brmux``` and ```veth1br``` (VLANs must be configured in all those devices).

If you want to use flat provider networks only, follow the remainder of this subsection and then continue with sections [5. Kolla-ansible and OpenStack installation](#5-kolla-ansible-and-openstack-installation) and [6. Managing your cluster](#managing-your-cluster).

If you are determined to use VLAN provider networks in your cluster, follow this section only up to step _2. Install and enable netplan_, and then proceed to subsection [3.iv VLAN provider networks - part 1 (RPi network configuration for a flat network)](#3iv-vlan-provider-networks---part-1-rpi-network-configuration-for-a-flat-network). Note that in the latter case we use a two-step approach: in subsection 3.iv we only set a flat provider network in the cluster (step 1), while VLAN provider networks will be deployed as late as in section [7. VLAN provider networks - part 2 (enabling and using VLAN provider networks)](#7-vlan-provider-networks---part-2-enabling-and-using-vlan-provider-networks) (step 2).

#### Network configuration description

Network devices on our RPi have to meet Kolla-Ansible requirements for network interfaces. In particular, Kolla-Ansible requires that there are at least two network interfaces available on each OpenStack host (Kolla-Ansible user will then assign OpenStack roles to those interfaces). As Raspbbery Pi comes with only one network card we have to use virtual interfaces to fulfill the above requirement. We will create veth pairs and a linux bridge, and we will put them together to emulate desired configuration.

This is depicted in the figure below where also the role of respective interfaces is shown. In our setup, interfaces ```veth0``` and ```veth1``` correspond to physical interfaces in production OpenStack host. They will be configured by Kolla-Ansible according to Kolla-Ansible/OpenStack networking principles and we assume that ```veth0``` and ```veth1``` will serve as Kolla-Ansible ```network_interface``` and ```neutron_external_interface```, respectively. For more information on Kolla-Ansible networking for OpenStack, please refer to respective [documentation](https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html).

```
SCHEMATIC VIEW OF RPi INTERNAL NETWORK EMULATING REAL DC NETWORK ENVIRONMENT

network_interface, veth0          neutron_external_interface, veth1
(OStack services, tenant nets)    (provider networks, tenant routers/floating IPs)
static IP 192.168.10.2x/24         no IP address assigned (Kolla-Ansible requires that)
 
                                                    HOST NETWORK domain ("host-internal" in production),
                                                    - under Nova/Neutron governance
    +---------+                     +---------+
=== |  veth0  | =================== |   veth1 | ==  - interfaces to be specified in globals.yml, used by Kolla-Ansible and OpenStack;
    +----┬----+                     +----┬----+     - they correspond to physical network cards (interfaces) in a production server
         |                               |          - tagged VLANS will be configured for veth1 in case of using VLAN provider networks
         |                               |
         | <-------- veth pairs -------> |          DATA CENTER NETWORK domain ("physical" in production),
         |                               |          - under DC admin governance
    +----┴----+                     +----┴----+ 
    | veth0br |                     | veth1br |     - tagged VLANs have to be configured by the admin from eth0 across veth1br to
    +----┬----+                     +----┬----+     veth1 in case of using VLAN provider networks
    +----┴-------------------------------┴----+
    |                   brmux                 |     - L2 device, IP address not needed here, tagged VLANs have to be configured here
    +---------------------┬-------------------+     (they extend towards veth1) in case of using provider VLAN networks
                          |                         - corresponds to a physical switch in data center L2 network
   Raspberry Pi      +----┴----+
   xxxxxxxxxxxxxxxxx |  eth0   | xxxxxxxxxxxxxx     - physical interface of RPi (taken by brmux), needs no IP address, but 
                     +---------+                    tagged VLAN have to be configured here in case of using VLAN provider networks
```

To make sure the above structure is persistent (survives system reboots), we use ```networkd``` and ```netplan``` configuration files to define our network setup. Basically, networkd files allow to define veth pair devices and tagged VLANs on ```eth0```, ```brmux```, ```veth0br``` and ```veth1br```, while neplan code contains the rest of needed information. In fact, using networkd files turned out to be necessary because it was not possible to declare all the necessary details in netplan. It cannot be ruled out that all necessary configurations (veth pairs and tagged VLANs on eth0, brmux, veth0br and veth1br) will be fully achievable in future netplan versions (interested user can check it on her/his own). For more details on how to configure network devices in networkd and netplan, please refer to respective documentation.

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
> This and the following steps are prepared for the use of flat provider network only in your OpenStack DC. **That means, you are not planning to use VLAN provider networks.** Introducing VLAN provider networks requires additional configurations for ```eth0```, ```brmux``` and ```veth1br``` to serve VLANs in those devices. Respective VLAN configurations have also to be introduced in the TP-Link switch. If you are interested in setting VLAN provider networks in your cluster, skip the remainder of this subsection and go to subsection [3.iv VLAN provider networks - part 1 (RPi network configuration for a flat network)](#3iv-vlan-provider-networks---part-1-rpi-network-configuration-for-a-flat-network).

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
# distinct IP address range for this subnetwork. It will then be sufficient to attach
# from your namagement host to Linsys WiFi access point. Alternatively, if you prefer to
# continue using the cable connection to Linksys, you will also have to enable routing on
# Linksys between this subnetwork and the fixed subnetwork used to access OpenStack.

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
        - 192.168.10.2x/24  # ADJUST THIS ADDRESS FOR EACH YOUR RPI !!!!!!!!
      nameservers:
        addresses:
          - 192.168.10.1    # ADJUST to match your Linksys dhcp server address
          - 8.8.8.8         # this is optional
          - 8.8.2.2         # this is optional
      routes:
        - to: 0.0.0.0/0
          via: 192.168.10.1 # ADJUST to match your Linksys router address
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

# ssh now disconnects so reconnect, but using fixed IP addresses you set in file `50-cloud-init.yaml`
# check the connectivity again and reboot for any case

$ ping wp.pl
$ sudo reboot
```

### 3.iv VLAN provider networks - part 1 (RPi network configuration for a flat network)

#### General

There's no one generic configuration of provider networks in OpenStack. In this guide, we describe how to deploy a combination of a single flat and several VLAN (tagged) provider networks. In our case, flat (untagged) provider network will support OpenStack management, external and tenant overlay networks (so similarly to the basic flat provider network setup described in section 3.iii). Additionally, we will create a set of VLANs allowing the admin to create additional, VLAN provider networks. Such VLAN provider networks can be configured as external or internal (without external access) networks, depending on tenant needs in a given data center.

We adopt a two-step incremental approach for VLAN provider networks. This section describes the first step and we will create a flat provider network setup already known from section 3.iii. However, this time most of the network configurations will be defined in networkd files (contrary to this, in section 3.iii we used networkd as little as possible). This will result in an OpenStack network configuration that is identical to the one from section 3.ii (allowing one to perform the same experiments with OpenStack as with the configuration in section 3.iii). However, the current set of networkd and netplan files will be better suited for further conversion to enable VLAN provider networks.

In the second, crucial step, we will change some networkd configuration files to actually create tagged VLANs, enable VLAN provider networks and use them in our cluster. That is covered in section [7. VLAN provider networks - part 2](#7-vlan-provider-networks---part-2-enabling-and-using-vlan-provider-networks).

#### Configuring the network (flat)

  * First, if not already done, execute steps 1 and 2 from the previous section 3.iii (i.e., stop NetworkManager and install netplan).
  * Then upload the files stored in this repo in directory `flat` to respective directories on each RPi. Make sure to preserve the names of the paths contained inside directory `flat`. That is, files from the `flat/etc/netplan` directory should be uploaded to the `/etc/netplan` directory on each RPi, and those from the `flat/etc/systemd/network` directory to the `/etc/systemd/network` directory on each RPi. These files configure the RbPi for a flat network, meaning no VLANs. There is no Ethernet traffic isolation between tenants using such a flat provider network. Basic tenant L2 traffic isolation will be implemented using VXLANs, which encapsulate Ethernet frames into IP/UDP packets.
  * On each RPi, edit file `/etc/netplan/50-cloud-init.yaml` and update the IP address of interface `veth0`, DHCP server address (the Linksys or other router in your network), and the default route (typically the same as the DHCP server address in your router) according to your environment.
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

Your network configuration is now the same as that from section 3.iii. You can continue with the installation procedure (section 4 and 5) to enjoy using OpenStack with a flat provider network.

## 4. Management host preparation

### 4.i General notes

1. Maintaining 100% consistency between the version of Kolla-Ansible used and the OpenStack release deployed is key for successfull installation of the OpenStack cloud. 
2. This guide refers to OpenStack release ```2025.1``` and respective Kolla-Ansible guide is available under [this link](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart.html). Please, note the ```2025.1``` discriminator of OpenStack release in the Kolla-Ansible URI.
3. The main goal of this guide is to instruct how to _**install**_ OpenStack cloud with Kolla-Ansible on Raspberry Pi cluster. For information on how to _**manage**_ OpenStack cloud using Kolla-Ansible, please refer to the original documentation of the Kolla-Ansible project.
4. Be careful to install the right OS version. Kolla-Ansible/OpenStack 2025.1 requires Ubuntu 24.04 or Debian 12 on the management host. Kolla-Ansible 2023.1 requires Ubuntu 22.04 on the management host. Other OS-es may also work, possibly after appropriate adaptations, but we have not check other options.

### 4.ii VM creation and basic configs

Here, we assume using a virtual machine, but things do not change if the management host is a physical machine.

  * Create `Ubuntu 24.04 LTS` VM, preferably as a desktop machine. We are using `Ubuntu 24.04 LTS` for Kolla-Ansible 2025.1 (_Note: 'Ubuntu 24.04` is not supported by Kolla-Ansible 2023.1, and you should use Ubuntu 22.04 as the highest distribution supported by Kolla-Ansible 2023.1_). We have not tested other Linux distributions. Resource requirements for the VM are moderate (4GB RAM, 20GB disk, 1vCPU should be sufficient).
  * We use VirtualBox and configure the network card of the VM to work in ```Bridged``` mode. Nevertheless, NAT mode should work as well in a basic scenario, but using briged mode is more convenient in non-standard situations (e.g., when one needs to copy some files [from](https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/how-to-install-virtualbox-guest-additions-on-ubuntu-22-04.html?utm_content=cmp-true) between the RPis and the management host). It is generally simpler when all components (cluster, management host, our physical host) operate in the same L2 segment.
  * After launching the VM in VirtualBox, the copy-paste feature will probably not work. You will have have to install GuestAdditions. This can be done in a while. First follow the steps that follow.
  * Disable the automatic upgrade option in the VM; in the desktop, search for this setting in `Options`.
  * If the terminal suspends/does not open, the screen is flickering or the cursor takes the form of a black rectangle, disable Wayland display server protocol, see e.g., [this for 24.04](https://askubuntu.com/questions/1536250/disabling-wayland-permanently-under-ubuntu-24-04-1-lts-doesnt-work-in-virtualbo) and [this for 22.04](https://linuxconfig.org/how-to-enable-disable-wayland-on-ubuntu-22-04-desktop)
  * Assign sudo privileges to your user (in this guide, we assume username `ubuntu`)
  ```
$ sudo usermod -aG sudo $USER
  ```

   * Update/upgrade
  ```
$ sudo apt update && sudo apt upgrade
  ```

  * When using VirtualBox, install GuestAdditions - refer, e.g., to [this](https://linuxconfig.org/installing-virtualbox-guest-additions-on-ubuntu-24-04) or [this](https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/how-to-install-virtualbox-guest-additions-on-ubuntu-22-04.html?utm_content=cmp-true)
    
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
  * Verify if docker works (check if you can run docker without sudo - it should be possible)
  ```
$ docker run hello-world
  ```

## 5. Kolla-Ansible and OpenStack installation

> [!Note]
> In the following, we assume `ubuntu@labs:~/labs/ostack$` to be the working (current) directory. When copy-pasting, adjust the commands according to your environment.

### 5.i Kolla-Ansible installation

The installation procedure is in principle the same as in the original [Kolla-Ansible guide for the 2025.1 release](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart.html). Installation of [Kolla-Ansible 2023.1](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart.html) is very similar to, but not identical as that of 2025.1. One difference is that in 2023.1 one has to install Ansible explicitly by running a separate command while in 2025.1 this step is already included in the Kolla-Ansible script and you do not bother with Ansible installation. Other differences are a direct consequence of the 2023.1 release status being changed to "unmaintained", but without corresponding updates in the publicly available Kolla-Ansible manual and in a single Kolla-Ansible configuration file (the term ```stable``` is used instead of ```unmaintained```). Corrective changes to respective instructions applicable for 2023.1 are outlined in the following.

One can use the original source document for 2025.1 and install on his own or take advantage of 100% ready-to-use commands documented below.

  * Install Python packages, create and activate virtual environment
```
$ sudo apt-get update
$ sudo apt-get install -y git python3-dev libffi-dev gcc libssl-dev python3-venv
$ python3 -m venv kolla-2025.1   # you can use whatever name you want for the virtual environment
$ source kolla-2025.1/bin/activate

```

  - Hint: deactivate an active venv and completely remove a venv in question:
```
$ deactivate
$ rm -r <venv-root-folder-name->
```

> [!Note]
> From now on, we work in virtual environment kolla-2025.1. In the text below, the versions of all components installed are assumed to be compatible with kolla-2025.1 and should be changed appropriately if you use other release of Kolla-Ansible (follow respective Kolla-Ansible _Quick start section_ for each release: [quickstart](https://docs.openstack.org/kolla-ansible/YOUR-RELEASE/user/quickstart.html)).

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
> If you see error message similar to _```error: pathspec 'stable/2023.1' did not match any file(s) known to git ERROR! Failed to switch a cloned Git repo `https://opendev.org/openstack/ansible-collection-kolla` to the requested revision `stable/2023.1`.```_, do not panic. Your release has probably got the "unmaintained" status. You should go to [this repo](https://opendev.org/openstack/kolla-ansible) and check the name of the branch where your specific release is currently stored and where you will find its current branch name. Then change the name of your (supposed) branch (```@stable``` in the example) to the right one. To this end, edit local file: ```nano kolla-2023.1/share/kolla-ansible/requirements.yml```. Most probably you will have to change the name from ```stable/2023.1``` to ```unmaintained/2023.1``` (every release will eventually get the "unmaintained" status so treat the above as an example and adapt it appropriately to your particular case if needed). Refer to [here](https://docs.openstack.org/project-team-guide/stable-branches.html#maintenance-phases) for more on maintenance phases of OpenStack releases.

### 5.ii Preparing configuration files for Kolla-Ansible

Now, generated by Kolla-Ansible, there are several configuraton files with default settings. We have to customize them (and even add new files) to make the configuration appropriate for our cluster.

```
# directory with customization files expected by Kolla-Ansible
$ sudo mkdir -p /etc/kolla
$ sudo chown $USER:$USER /etc/kolla
```

  * Copy globals.yml and passwords.yml to /etc/kolla directory, and kolla-ansible inventory file to our working directory. We will customize these file for our deployment soon.
    
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

192.168.10.21    ost01
192.168.10.22    ost02
192.168.10.23    ost03
192.168.10.24    ost04
...
```

### 5.iii Configuring Kolla-Ansible files for specific OpenStack depolyment

1. Prepare passwords.yml file

  * Generate passwors in file ```/etc/kolla/passwords.yml```

```
$ kolla-genpwd
```

> [!Warning]
> In case of receiving `PermissionError: [Errno 13] Permission denied: '/etc/kolla/passwords.yml'` change file permissions: `chown a+wr /etc/kolla/passwords.yml`. You can restore original permissions of `passwords.yml` after that.

  * edit file `/etc/kolla/passwords.yml` and change the keystone admin password to a human-readable form (search for `keystone_admin_password`; you can set the password simple, e.g., `admin` as in our case)
```
$ sudo nano /etc/kolla/passwords.yml
```

2. Prepare inventory file `multinode` and feature configuration file `globals.yml`

> [!IMPORTANT]
> Kolla-Ansible expects the user to assign OpenStack roles (```control```, ```network```, ```compute```, etc.) to hosts by specifying role assignments in the inventory file. Our experiments revealed that the roles ```compute``` and ```network``` SHOULD be separated (i.e., allocated to different hosts) in clusters with 8GB RAM Raspberries. Otherwise the cluster becomes prone to out of memory (OOM) issues which lead to severe instabilities of the cloud. In an all-in-one installation (one RPi in a cluster), the node goes out of memory right after the first CirrOS VM is created and OpenStack tends to hang. If there is more than one virtual machine instance, the cluster fails immediately. In such situations, even if one can login to the hosts after rebooting and even if some Kolla-Ansible containers remain in a healthy state, OpenStack as a whole is not functional anymore. In a multinode cluster, when the control and network functions run on the same host and the compute function runs elswhere, the control/network host reaches a stete close to the OOM regime after creating the second CirrOS VM. So, in the case of Raspberry Pi clusters, even with a multinode configuration the merging of ```compute``` and ```network``` functions in a single host should be avoided. These conclusions are reflected below in a sample fragment of the inventory file where ```copmute``` and ```network``` roles are assigned to different hosts (```ost4``` and ```ost3```, respectively).

  * file `multinode` is stored in the working directory

Edit the inventory file `multinode` and assign the `control`, `network` and `compute` functions to individual RPis. The following are the recommended settings for a four-host cluster. Adjust them for other cluster sizes, keeping the `control` function on a separate host. Note: for every appearance of each node provide ansible directives `ansible_user`/`password`/`become` (it is sufficient to provide the directives once for a given host regardless of the number of groups it belongs to, e.g. ost03 in our example). Remaining groups in file `multinode` should be left unchanged.

```
$ sudo nano multinode

...

[control]
ost04 ansible_user=ubuntu ansible_password=ubuntu ansible_become=true

[network]
ost03

[compute]
ost[01:03] ansible_user=ubuntu ansible_password=ubuntu ansible_become=true

(Note: remaining groups go here - leave them unchanged)

[monitoring]
#monitoring01

[storage]
#storage01

...
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
kolla_internal_vip_address: "192.168.10.20"
network_interface: "veth0"
neutron_external_interface: "veth1"
enable_neutron_provider_networks: "yes"
# The following line must be commented or deleted for 2023.1 where proxysql is not used at all. However,
# it must be uncommented for 2025.1 on Raspberry Pi OS to disable using proxysql, otherwise
# there will be page size incompatibility between Raspberry Pi OS and the proxysql application.
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

# The following settings are applicable only for qemu (not needed by KVM at all). Currently they serve informational purposes only.
#for RPi 5 enable cortex-a76 <== maybe for the future, but currently theres a lack of support for cortex-a76 in 2023.1 and 2025.1 libvirtd
#for RPi 4 enable cortex-a72
#[libvirt]
# qemu works only for RPi 4; if you really want, it enable all three settings that follow
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
> This note is for students interested in what's going on under the hood and in the `ml2_conf.ini` file as above. Remember that in our case, we use OVS as a layer-2 agent in OpenStack (it's THE default SETTING in globals.yml).
>
> 1) Attribute `network_vlan_ranges = physnet1:101:200` will be included in the controller (network node) configuration file `/etc/neutron/plugins/ml2/ml2_conf.ini`. This is the allowed range of VLANS to be used by VLAN provider networks in our OpenStack ([see here](https://docs.openstack.org/neutron/queens/configuration/ml2-conf.html))
> 2) Based on the above, knowledge of the established Neutron operating principles and the configuration we provided in the `globals.yml` file for the physical port `neutron_external_interface: "veth1"`, Kolla-Ansible will configure the following bindings:
> * in the `ovs-vsctl` database, binding of `veth1` to `br-ex` with a command like: `ovs-vsctl add-port br-ex veth1` (br-ex is the name of a bridge preassumed by OpenStack)
> * in the `/etc/neutron/plugins/ml2/openvswitch_agent.ini` file, the binding of the above-mentioned external bridges (br-ex, br-ex2, ...) to the physical networks: `bridge_mappings = physnet1:br-ex,physnet2:br-ex2` (in our case, there's no `physnet2:br-ex2` - we'd need another "physical" port, e.g., `veth2`, for that).
> (Neutron assumes interface names in the form and order `br-ex`, `br-ex2`, `br-ex3`, ..., and Kolla-Ansible associates them with the names `physnet1`, `physnet2`, ... based on the order of their appearance in the list defined by attribute `network_vlan_ranges = ...`. For example, if we additionally used `physnet2` in our cluster with VLANs allowed, `network_vlan_ranges` could look like this: `network_vlan_ranges = physnet1:101:200,physnet2:201:300`, [see here](https://docs.openstack.org/ocata/config-reference/networking/samples/openvswitch_agent.ini.html))
> 3) We will explicitly use the name `physnet1` (and optionally `physnet2`, etc.) from the admin level when creating a provider network. So, the admin must be aware of the presence of such `physnetX` networks in the configuration files. Respective command specifies the physical network (thus indirectly references a specific bridge, `br_ex`, and thus to a physical interface, `veth1`) on which the declared provider network (flat or VLAN) is to be created (you will have to use VLAN IDs from the pool associated with `physnet1`, etc.).
> 4) Note: Binding physical interfaces (they are `veth0, veth1,...` in our case, but could be `eth0, eth1,...` in a physical infrastructure) to internal OpenStack bridges is the responsibility of the OpenStack installer, in our case Kolla-Ansible, but it could be human. That is why we have to specify the roles of the individual interfaces in our installation in the `globals.yml` file so that Kolla-Ansible knows exactly how to bind them. Important to note is that all these configurations are performed by the OpenStack administrator and are hidden from regular users (tenants).
>
> More about bridge mapping in OpenStack:
> * https://docs.redhat.com/en/documentation/red_hat_openstack_platform/10/html/networking_guide/bridge-mappings#maintaining_bridge_mappings
> * https://docs.redhat.com/en/documentation/red_hat_openstack_platform/10/html/networking_guide/bridge-mappings#bridge-mappings
> * https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html#openvswitch-ml2-ovs

### 5.iv Deploying OpenStack

In case of kolla-ansible below has problems with ssh to reach your RPis ("WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!"), check file /etc/hosts for the presence of resoultion data. If you have reinstalled the OS on the RPis, you have to **do delete** `rm ~/.ssh/known_hosts`.

```
$ kolla-ansible bootstrap-servers -i multinode 
$ kolla-ansible prechecks -i multinode
$ kolla-ansible deploy -i multinode
```

> [!IMPORTANT]
> If any host is unable to complete the container restart task, it may be due to connectivity issues with the `quay.io` container repository. To recover from this just run `kolla-ansible deploy` again. Below is an example of such recoverable failue. If the problem persists check `quay.io` health in [this portal](https://statusgator.com/services/quay) and if there is a clear evidence of deteriorations try to re-run `deploy` after some time. Another (more advanced) option is to build images of kolla containers on your own and keep them in a local repository according to [this guide](https://docs.openstack.org/kolla/latest/admin/image-building.html) (to this and, you will need to install and run your own repo, reserve some GB of storage for images; you will also need to adapt the commands from the linked guide to aarch64).. To recover from this just run `kolla-ansible deploy` again. Below is an example of such recoverable failue. If the problem persists check `quay.io` health in [this portal](https://statusgator.com/services/quay) and if there is a clear evidence of deteriorations try to re-run `deploy` after some time. Another (more advanced) option is to build images of kolla containers on your own and keep them in a local repository according to [this guide](https://docs.openstack.org/kolla/latest/admin/image-building.html) (to this and, you will need to install and run your own repo, reserve some GB of storage for images; you will also need to adapt the commands from the linked guide to aarch64).
> ```
> RUNNING HANDLER [neutron : Restart neutron-openvswitch-agent container] **************************************************************
> changed: [ost01]
> fatal: [ost02]: FAILED! => {"changed": false, "msg": "Unknown error message: dial tcp: lookup quay.io on 192.168.10.1:53: read udp 192.168.10.22:47363->192.168.10.1:53: i/o timeout"}
> changed: [ost03]
> ```
>
> We believe that monitoring memory usage on hosts is a good practice, especially on the control node (remote terminal and `htop` command does a good job of this). Memory usage crossing 7GB suggests that OpenStack may crush soon. You can try to remedy this by stopping unnecessary workload (VMs). What may work in the absence of such a workload to delete is stopping OpenStack using the command `kolla-ansible stop-containers` and resuming the cloud with the command `kolla-ansible deploy-containers` (refer to [section 6.i](#6i-shut-down-the-cluster-and-start-it-again) for more details). This latter method (stopping/starting) takes quite a bit of time (say, 30 minutes in total), but it's worth it because it gives you a chance to avoid installing the cluster from scrach (and sometimes even reinstalling the operating system on the failed node(s)).

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
    - init-runonce script files for various kolla-ansible/OpenStack releases and tuned for our Raspberry Pi environment are available in this repo. Select appropriate <kolla-ansible-release> and <image-os-name>, possibly edit the file to change some settings according to your environment (e.g., external network address range). The file name format of those scripts is as follows:
   
      `init-runonce.<kolla-ansible-release>.<image-os-name>`
      
      **Remark 1**: the original version of init-runonce script is generated by the command `kolla-ansible post-deploy`. It is stored as /path/to/our/venv/share/kolla-ansible/init-runonce. We have to use slightly modified version of the script to match the requirements/constraints of our Raspberry Pi platform (e.g., we need aarch64 image, we can afford only quite small VM flavors, OES-es with a rather small footprint, etc.).
      
      **Remark 2**: currently recommended settings are `<kolla-ansible-release>=2025.1` and `<image-os-name>=cirros`. So, to run the recommended version use file `init-runonce.2025.1.cirros`. However, tested is also `<kolla-ansible-release>=2025.1`.

      - run the script
      ```bash
      # for Kolla-Ansible 2025.1
      $ ./init-runonce.2025.1.cirros
      
      # or (for Kolla-Ansible/OpenStack 2023.1)
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

In this section, we describe how to enable VLAN provider networks by modifying the setup from section [3.iv VLAN provider networks - part 1](#3iv-vlan-provider-networks---part-1-rpi-network-configuration-for-a-flat-network). Then we show how such networks can be used.

Note that we'll be reconfiguring a running OpenStack instance. However, all of the configurations below can be performed just as easily in section [3.iv VLAN provider networks - part 1](#3iv-vlan-provider-networks---part-1-rpi-network-configuration-for-a-flat-network) so before running the Kolla-Ansible commands (before installing OpenStack).

### 7.i Setting VLANs for provider networks

#### 7.i.a Setting VLANs in the physical network (the TP-Link switch)

In the `VLAN -> 802.1Q VLAN Configuration` tab, enable VLAN support using the `Enable` switch at the top, and then add specific VLANs individually on the switch ports. In our example below, we create a VLAN with VLAN ID 101, enabling it on the switch ports that connect our RPis (note that port 5 in TP-Links switch is the "WAN" port and it connects to your router (Linksys or TOTO-Link in our case). This way, VLAN 101 will be present on the control, network and compute hosts in the cluster, and once it is created, the corresponding provider network will have a similar coverage. Repeat this operation for the remaining VLANs we want to have in the cluster. The set of VLANs configured on the TP-Link switch should be consistent with VLAN declarations in the replacement files in the next section (actually, we define three VLANS there with VLAN IDs 101, 102, 103).

<p align="center">
 <img src=images/tplink-vlan101.png width='60%' />
</p>

#### 7.i.b Setting VLANs on the RPi hosts

On each RPi, replace a couple of files in the `/etc/systemd/network` directory with new versions that define the VLAN configuration. Those new versions are available in this repository in the `vlanned/etc/systemd/network` directory - upload them to your RPis.

> [!Warning]
> Remember that it is necessary to only replace the files provided here in the `vlanned/etc/systemd/network` directory **keeping untouched remaining files** already present on the RPis in directory `/etc/systemd/network`. Otherwise you will cut off remote access to your RPis unless you enable WiFi access to the boards before. As a last resort, to avoid a complete reinstallation, you will need to connect your devices to a monitor, keyboard, and mouse.

You are encouraged to review these files to learn how persistent configuration of VLANs can be achieved in Linux. The files contain explanation of each construct used. After uploading them, you can restart the RPi with the `reboot` command. Alternatively, you can reach your RPis using Linksys WiFi access point and restart only the networking with the following commands:

```
$ ip link set down brmux
$ ip link del dev brmux
$ systemctl restart systemd-networkd
```

> [!Note]
> To access RPi board using WiFi you have to activate WiFi access point on the Linksys router, enable WiFi on the board in your netplan file `50-cloud-init.yaml`, and run `netplan generate` - refer to point 3 in [this part](README.md#configuring-the-network-flat) of the guide for details.

After completing the above steps, VLANs 101, 102 and 103 will also be activated in the virtual devices on our RPi hosts. This will make these networks finally ready for building OpenStack VLAN provider networks on top of them.

#### 7.i.c Separating the flat network using a VLAN

As an additional exercise, you can use VLAN to isolate the traffic of the flat provider network in the physical part of your L2 segment (i.e., between the TP-Link switch and the RPis). That is, we want the untagged traffic corresponding to the flat provider network in our cluster to be carried in a dedicated VLAN between the TP-Link switch and the `brmux` virtual devices in the RPis. Please note that such a VLAN will be separate from the VLANs created for the tagged VLAN provider networks and will penetrate the cluster more shallowly than the provider VLANs. Its traffic will be unatgged when leaving TP-Link via Port 5 (this is similar to VLAN provider networks rendered as external, but also when leaving `brmux` via devices `veth0br` or `veth1br` (of course, untagged traffic in the opposite direction appearing on TP-Link and `brmux` devices will be tagged with the ID of our additional VLAN). Effectively, OpenStack and the Linksys router will perceive the flat provider network unchanged (its traffic will untagged for OpenStack), but at the same time this traffic will be isolated using a VLAN on the remaining part of the physical network.

The practical benefit of using this extra encapsulation will be increased security in the physical part of our network infrastructure (not a fundamental change, but worth mentioning). From an educational point of view, the procedure described here allows for a better understanding of the essence of the VLAN provider network concept in OpenStack (described in the previous two subsections).

Follow the steps described below.

* Create the additional VLAN in the TP-Link switch (we set VLAN ID = 2 but anthing different from the VLAN IDs reserved for provider networks will be fine). As before, you do this in the `VLAN -> 802.1Q VLAN Configuration` tab.

<p align="center">
 <img src=images/tplink-vlan2.png width='60%' />
</p>

* Additionally, in the `VLAN -> 802.1Q PVID Setting` tab, set the VLAN ID to tag the traffic coming from the router (Linksys) to the switch on Port 5.

<p align="center">
 <img src=images/tplink-pvid2.png width='60%' />
</p>

* Now, uncomment VLAN2 declarations (the parts related to `[BridgeVLAN] VLAN=2`) in network files `02-ost-eth0.network`, `10-ost-net-ext-veth1br.network` and `10-ost-net-itf-veth0br.network` in the `/etc/systemd/network directory`.

* Finally, restart the RPis or the networking (via WiFi) as described in the previous subsection.

As before, you should now be able to access OpenStack via the dashboard or the OpenStack CLI.

### 7.ii Creating and using VLAN provider networks

#### 7.ii.a Provider network dedicated to a tenant

You have to connect to OpenStack using OpenStack command line tool as admin (source appropriate rc file).

Similarly to how a provider network is created in file `init-runonce.2025.1.cirros`, we can create a provider network dedicated to a tenant. First, create a VLAN based provider network and subnetwork. Note that this time the created provider network is not `shared`. In the example below, we use VLAN 101 for that.

```
# create the provider network
openstack network create --provider-physical-network physnet1 --provider-network-type vlan --provider-segment 101 dedicated-provider-net
# create subnetwork in the provider network
openstack subnet create --network dedicated-provider-net --subnet-range 192.168.100.0/24 dedicated-provider-subnet
```

Then use OpenStack RBAC to assign the provider network to a particular tenant (project):

```
openstack network rbac create --target-project <tenant_project_id> --action access --network dedicated-provider-net
```

One can do the same using also the OpenStack dashboard. You can try it on your own.

Once you have assigned the provider network to a tenant, log in as that tenant and check the network availability in the project.

#### 7.ii.b External network using VLAN provider network

Actually, this is similar to the example from file `init-runonce.2025.1.cirros` combined with the use of VLAN tag as in the dedicated provider network example described above. In the example below, we use VLAN 102 for that.

```
# create VLAN based (external) provider network 
openstack network create --share --external --provider-physical-network physnet1 --provider-network-type vlan --provider-segment 102 public2
# create subnetwork in the external network with a range of floating IP addresses
openstack subnet create --no-dhcp --ip-version 4 \
   --allocation-pool start=192.168.10.31,end=192.168.10.35 --network public2 \
   --subnet-range 192.168.10.0/24 --gateway 192.168.10.1 public2-subnet
```

Note that the size of the IP address pools for floating IP addresses above and in the init-runonce.x.y scripts match the size of the reservation range suggested in section 3.

## 8. ADDENDUM - accessing the cluster using VPN

You can set a VPN to access the cluster remotely. Any VPN platform of your choice can be used for that. In our lab, we have tested the [Zero Tier](https://my.zerotier.com/) VPN solution. You can install it according to the original instructions. A Polish version of the instructions (adapted to our environment) is available [here](https://github.com/dbursztynowski/k3s-taskforce/blob/master/zt-manual.md#poradnik-po%C5%82%C4%85czenia-klastra-z-sieci%C4%85-zerotier-vpn).

There are two main options for installing ZeroTier in the cluster.

If you have a spare (dedicated) computer to host the VPN connection to the cluster you can use it as the access node (jump host) without impacting the rest of your devices in any way. If your local router has been configured, and your Raspberry Pis are up and running with the newly installed SD cards, everyone in your team will be able to remotely complete all the remaining steps in this guide. Note that such a jump host has to be connected to the L2 network segment of your OpenStack. Also, using a jump host, you will be able to remotely shut down and restart again all RPis.

If you do not have a spare computer to set a VPN jump host on than you can assign selected OpenStack host (selected Raspberry Pi board) for this role. However, make sure you assign a board that will become a pure `compute node` in your OpenStack (do not overload OpenStack `control` or `network` nodes). Mind also that you should not shut down the RPi acting as the jump host, otherwise it will become inaccessible and the remote access to the cluster will fail. In such a case, the only option to start the cluster again will be to perform a local restart. In contrast, using a separate machine to host the VPN connection allows one to safely shut down and restart the entire cluster from a remote location. Of course, for remote access to work, both the jump host and the TP-Link switch must be powered on.

In either case, starting the cluster RPis can be forced by accessing the TP-Link switch dashboard and restarting the switch using the `System->Reboot` option (powering on the PoE ports of the switches turns on the RPi boards).

Enjoy remote access to your cluster!
