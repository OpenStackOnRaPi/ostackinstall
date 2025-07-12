# OpenStack on Raspberry Pi
In this repo we describe how to install OpenStack on Raspberry Pi cluster using Kolla-Ansible.

As of mid July 2025, SW/HW cluster configurations containing Kolla-Ansible OpenStack release 2023.1, and both Raspberry Pi 4B and Raspberry Pi 5 have been tested successfully. We recommended for deployment of OpenStack on the Raspberry Pi HW platform. For each SW/HW configuration mentioned, OpenStack has been tested in two variants: with QEMU emulation and with KVM enabled. Both variants work well on RPi4/5 and K-A 2023.1, with KVM being the prefered choice. However, newer releases of Kolla-Ansible/OpenStack can not be recommended at present. Therefore, even if some parts of this guide mention releases other than 2023.1, they are currently for illustrative purposes only and not for practical use. We will update this guide once newer releases pass our tests.

The overall process consists of several steps including: **system configuration on the Raspberries** (OS installation, setting needed permissions, configuring network settings that are required by Kolla-Ansible, etc.), **configuration of the management host** (this will be a separate host where we will run Kolla-Ansible installation commands and later access OpenStack using command line tool), **installation and customization of Kolla-Ansible** and its configuration files on the management host as needed by our deployment, **actual deploymnet of OpenStack** using Kolla-Ansible, then **creation of basic elements of virtualized infrastructure** (VM image of CirrOS, external and tenant networks, public and private routers, security groups, etc.) and finally **creation of the first VM instance** in our cloud. These steps are detailed in the remainder of this document.

## Table of contents

1. [Introduction](#1-introduction)
2. [Assumptions](#2-assumptions)
3. [Raspberry Pi preparation](#3-raspberry-pi-preparation)
   1. [RaPi system configuration](#rapi-system-configuration)
   2. [RaPi network configuration](#rapi-network-configuration)
4. [Management host preparation](#4-management-host-preparation)
   1. [General notes](#general-notes)
   2. [Management host system configuration](#management-host-system-configuration)
5. [Kolla-ansible and OpenStack installation](#5-kolla-ansible-and-openstack-installation)
   1. [Koll-Ansible installation](#kolla-ansible-installation)
   2. [Generate configuration files for Kolla-Ansible (default templates)](#generate-configuration-files-for-kolla-ansible-default-templates))
   3. [Configure Kolla-Ansible files for specific OpenStack depolyment](#configure-kolla-ansible-files-for-specific-openstack-depolyment)
   4. [Deploy OpenStack instance](#deploy-openstack-instance)
   5. [Postdeployment and first instance](#postdeployment-and-first-instance)
6. [Managing your cluster](#managing-your-cluster)
   1. [Stop the cluster and start again](#stopstart-the-cluster-switch-off-not-destroy--and-start-again)
   2. [Destroy your cluster](#destroy-your-cluster)


   
## 1. Introduction

The scope of application of our clusters is education. A cluster of this type allows us to present/explain various features/concepts of OpenStack, some of them being hard or impossible to show using AIO or virtualized OpenStack setups. Many of such features are related to the administration of OpenStack data centre - a domain of activity that is hidden from regular users in "normal" DCs. For example, the management of provider networks where the admin needs to configure VLANs in the physical network of the data centre and declare them in OpenStack and Kolla-Ansible config files. Considering that the time needed to practice and learn even basic things is non negigible, a decent amount of resources is needed to serve a dozen or more student teams in limited time. Our approach allows one achieve these goals without the need to allocate dozens of servers each worth thousands of $.

Currently, Raspberry Pi 4 is assumed as the HW base. It is expected that extensions (if any) needed for Raspberry Pi 5 will be added to this guide once we test RaPi 5 setup sufficiently well.

Worth of noting is also that our clusters are a perfect base for experimenting with Kubernetes. In our lab, we use bare-metal setup of K3s which is perfect for Raspbbery Pi.

## 2. Assumptions

All procedures described in this guide assume HW and SW setup of the cluster as specified below:

1. Raspberry Pi 4
   * recommended set: [2x4GB RAM + 2x8GB RAM] or [3/4x8GB RAM] per cluster
     * at least [1x4GB RAM + 1x8GB RAM]
     * control node should run on 8GB machine
   * all Pi are equipped with 32GB SD disk
   * Note: we believe that a single RaPi 8GB RAM in all-in-one setup of Kolla-Ansible OpenStack could work well for very basic evaluation, but we have not tested this option thoroughly. However, we've observed that in -all-in-one OpenStack with default Kolla-Ansible settings, one can successfully create a single cirros instance with 512MB memory, but creating second similar instance breaks OpenStack because of lack of memory.
2. SW
   * OS: Raspberry Pi OS Lite (64bit), a port of Debian 12 (Bookworm) with no desktopp environment
   * Kolla-Ansible 2023.1; respective environment components according to 
   * Note: newer releases of Kolla-Ansible will be tried in the future (2023.1 has got status "unmaintained") and we'll update this guide accordingly after completing the tests 
3. Network:
   * the Pis are equipped with 802.3af/at PoE HAT from Waveshare (PoE is optional but simplifies cluster wiring) 
   * they are powered form TP-Link TL-SG105PE switch (it supports 802.1Q which can be used to set multiple VLAN provider networks in OpenStack)
   * TP-Link switch is connected to a local router with DHCP enabled to isolate the network segment of OpenStack DC from the rest of local network infrastructure
   * **reserve a pool of IP addresses for the use by OpenStack** on your local router; 20 addresses will be sufficient for our purposes. They **MUST NOT** be managed by the DHCP server. Four of them (two in case of two-board cluster)) will be assigned by you to the RbPis using netplan (see [here](https://github.com/OpenStackOnRaPi/OStackInstallRaPi/blob/main/README.md#configuration-description)), and one will be allocated as the so-called ```kolla_internal_vip_address``` (see [here](https://github.com/OpenStackOnRaPi/OStackInstallRaPi/blob/main/README.md#configure-kolla-ansible-files-for-specific-openstack-depolyment)). Remaining addresses will serve as ```floating IP addresses``` for accessing created instances from the outside of your cloud.
4. Virtualization
   * Currently, we use qemu emulation for instances as we have not managed to get KVM working on Raspberry Pi under Kolla-Ansible OpenStack. This may change in the future.
5. Notes
   * other PoE HATs for Raspberry Pi 4 and other PoE switches should work, too
   * for general education purposes, we use setups with at least 3 RaPis and a managed switch (802.1Q) in the cluster to demonstrate how VLAN-based provider networks can be used in OpenStack; this is impossible to show using AIO (all-in-one) OpenStack setups. But if one does not need VLAN provider networks, unmanaged switch can be used as well. Note that this guide does NOT cover configuring VLAN provider networks (we shall provide this addition in the future).
   * other details that may be relevant are explained in the description that follows
   * trials with Raspberry Pi 5 are planned for the near future
  
## 3. Raspberry Pi preparation

The following has to be done for each Rasppbery Pi in your cluster. The instructions will be given one by one, but you are free to gather them in bash scripts if you wish (sometimes a reboot is needed so you will have to prepare a couple of such scripts or you could prepare Ansible playbook to automate the installation completely, but how to do it is out of the scope of this guide). The configurations include two phases: system configuration (installs, upgrades, etc.) and host network configuration (networkd, netplan).

### RaPi system configuration

1. Flash the OS (Raspberry Pi OS Lite (64bit), a port of Debian 12 (Bookworm) with no desktopp environment) onto microSD card. We recommend using Raspberry Pi imager.
   * make sure password authentication for ssh access is enabled (the instructions given below fit this authentication method)
   * it is recommended to set the value of host name, user name and password as you will use afterwards in Kolla-Ansible playbooks. In the examples below, we set "ubuntu" for both the user name and password, and use the convention ost01, ost02, ... to set Raspbbery Pi host name.
  
2. After switching on the RaPi, find its IP address. In our setup, check the ```Device list``` panel in the Linksys router GUI (access on, e.g., 192.168.1.1, login as root, pwd admin). SSH to the RaPi using the credentials from step 1 above.

3. Assuming your user name on the RaPi is ubuntu (otherwise, adapt the following) run

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

   Note: ```lm-sensors``` does not serve OpenStack purposes directly, but can be used to monitor CPU temperature (one has to ssh onto the RaPi) 

  ```
$ sudo apt-get install net-tools -y && sudo apt-get install lm-sensors -y

# run to check CPU temperature
$ sensors
  ```

   * check temperature every 20 sec.
```
while :; do sensors; sleep 20; done
```

7. Enable packet forwarding on the RaPi

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

### RaPi network configuration

#### Configuration description

Network devices on our RaPi have to meet Kolla-Ansible requirements for network interfaces. In particular, Kolla-Ansible requires that there are at least two network interfaces available on each OpenStack host (Kolla-Ansible user will then assign OpenStack roles to those interfaces). As Raspbbery Pi comes with only one network card we have to use virtual interfaces to fulfill the above requirement. We will create veth pairs and a linux bridge, and we will put them together in desired configuration. This is depicted in the figure below where also the role of respective interfaces is shown. In our setup, interfaces ```veth0``` and ```veth1``` correspond to physical interfaces of OpenStack host. They will be configured by Kolla-Ansible according to Kolla-Ansible/OpenStack networking principles and we assume that ```veth0``` and ```veth1``` will serve as Kolla-Ansible ```network_interface``` and ```neutron_external_interface```, respectively. For more information on Kolla-Ansible networking for OpenStack, please refer to respective [documentation](https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html).

  ```
network_interface                 neutron_external_interface 
(OStack svcs, tenant nets)        (provider networks, tetnant routers/floating IPs)
static IP 192.168.1.6x/24         no IP addr assigned (Kolla-Ansible requires that)
    +---------+                     +---------+
    |  veth0  |                     |  veth1  | <=== intfcs to be declared in globals.yml, used by Kolla-Ansible and OpenStack
    +---------+                     +---------+
         |                               |           HOST NETWORK domain ("host-internal" - under Nova/Neutron governance)
    - - -|- - - - - - - - - - - - - - - -|- - - - - - - - - - - - - - - - - - - - - - - - -                      
         | <-------- veth pairs -------> |           DATA CENTRE NETWORK domain ("physical" - under DC admin governance)
    +---------+                     +---------+ 
    | veth0br |                     | veth1br |      tagged VLANs have to be configured by the admin between eth0 and veth1
    +---------+                     +---------+        in case of using provider VLAN networks
    +----┴-------------------------------┴----+
    |                   brmux                 |      L2 device, IP address not needed here, tagged VLANs have to be configured here
    +---------------------┬-------------------+        (towards veth1) in case of using provider VLAN networks
                     +---------+
                     |  eth0   |      physical interface of RaPi (taken by brmux), no IP address is needed, but tagged VLANs 
                     +---------+      have to be configured in case of using provider VLAN networks
  ```

To make sure the above structure is persistent (survives system reboots), we use ```networkd``` and ```netplan``` files to define our network setup. Basically, networkd files allow to define tagged VLANs on ```eth0```, ```brmux```, ```veth0br``` and ```veth1br```, while neplan complements the definitions with the rest of needed information. In fact, the use of both levels (networkd and netplan files) was necessary a time ago when it was not possible to configure tagged VLANs solely in netplan. This may have changed since then and it may happen that with newer releases of netplan all needed configurations (including tagged VLANs on eth0, brmux, veth0br and veth1br) are possible using netplan (interested user can check it on her/his own). For more details on how to configure network devices in networkd and netplan, please refer to respective documentation.

#### Configuration implementation

**Note: steps 1 and 2 below are needed only in case of Debian and can be skipped for Ubuntu**

1. Stop NetworkManager, and and start systemd-networkd

   _Note: for some historical reasons, we use networkd to have persistent configuration of network devices on our RaPis; one can use NetworkManager for this, but it will be necessary to convert respective network constructs from networkd to NetworkManager notation (NetworkManager notation is different form the one used by networkd)._

  ```
$ sudo systemctl stop NetworkManager && sudo systemctl disable NetworkManager
$ sudo systemctl enable systemd-networkd && sudo systemctl start systemd-networkd
$ sudo systemctl status systemd-networkd                  <= should be Active: active (running) ... 
  ```

2. Install and enable netplan (ref. https://installati.one/install-netplan.io-debian-12/?expand_article=1)

  ```
$ sudo apt-get update && sudo apt-get -y install netplan.io
  ```

3. Main host network configurations

   _**NOTE: this setup is prepared for flat provider network only in your OpenStack DC. To allow for VLAN provider networks, additional configurations are needed for ```eth0```, ```brmux``` and ```veth1br``` to set VLANs that should be served by those devices. Respective configurations of VLANs should also be introduced in the TP-Link switch. If you are interested in setting VLAN provider networks in your cluster, please contact the instructor for more info.**_

  * for `networkd` backend, for `veth0-veth0br` pair

```
sudo tee /etc/systemd/network/veth-openstack-net-itf-veth0.netdev << EOT
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
sudo tee /etc/systemd/network/veth-openstack-neu-ext-veth1.netdev << EOT
#neutron_external_interface w globals kolla-ansible
[NetDev]
Name=veth1
Kind=veth
[Peer]
Name=veth1br
EOT
```

  * for `netplan` utility, remaining settings of the network

    **NOTE 1: adjust the IP address of veth0 in each RaPi according to you network setup.**

    **NOTE 2:** in case of problems during `netplan generate` (or `netplan apply`) check the format of your file `/etc/netplan/50-cloud-init.yaml` - it's YAML and spaces matter.

    **Warning:** this part sometimes does not copy-paste correctly. If command `sudo netplan generate` reports formatting errors then edit file /etc/netplan/50-cloud-init.yaml and manually format it to the form visible below by adding missing spaces. 

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

# Interfaces

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
    veth1:                  # neutron_external_interface for kolla-ansible
      dhcp4: false
      dhcp6: false
    veth1br:
      dhcp4: false
      dhcp6: false

# Bridge brmux 
# logically, this is a switch belonging to the physical infrastructure of the
# data centre provider network (normally, physical network)

  bridges:
    brmux:
      interfaces:
        - eth0
        - veth0br
        - veth1br
EOT
```
  > [!IMPORTANT]
  > Now edit file `/etc/netplan/50-cloud-init.yaml` and adjust IP, DHCP server and DNS server addresses for your RPi (see the Netplan code above).

```
sudo nano /etc/netplan/50-cloud-init.yaml
```
  * deploy the changes permanently - create and configure devices
    
    Note: you will loose connectivity to your RaPi because of the change of IP address. To continue, ssh again using the new address.

```
$ sudo netplan generate
$ sudo netplan apply

(if ssh disconnects do ssh again)
# check the connectivity
$ ping wp.pl
$ sudo reboot
```

## 4. Management host preparation

### General notes

1. Maintaining 100% consistency between the version of Kolla-Ansible used and the OpenStack release deployed is key for successfull installation of the OpenStack cloud.
2. This guide refers to OpenStack release ```2023.1``` and respective Kolla-Ansible guide is available under the link ```https://docs.openstack.org/kolla-ansible/2023.1/user/quickstart.html)```. Please, note the ```2023.1``` discriminator of OpenStack release in the Kolla-Ansible URI.
3. Basically, this guide instructs how to _**install**_ OpenStack cloud with Kolla-Ansible. For information on how to _**manage**_ OpenStack cloud using Kolla-Ansible, please refer to the original documentation of the Kolla-Ansible project.

### Management host system configuration

#### Assumption: the management host is implemented as Ubuntu 22.04 desktop in VirtualBox. Other solutions will work too after appropriate adaptations.

1. Basic configs

  * Create the VM as desktop machine. There are not high requirements for the resources (4GB RAM, 20GB disk, 2vCPU should be sufficient)
  * Configure the network card of the VM in VirtualBox as ```Bridged```. This is well suited for Ansible as it may not work well running behind NAT.
  * Your host should work in the same network as the cluster (in our lab setup, attach it to the Linksys router)
  * After launching the VM, the copy-paste feature will probably not work. You will have have to install GuestAdditions. This can be done in a while. First follow the next steps.
  * Disable the automatic upgrade option; in desktop search for this in Options
  * If the terminal suspends/does not open, the screen is flickering or the cursor takes the form of a black rectangle, disable Wayland display server protocol, see e.g., [this](https://linuxconfig.org/how-to-enable-disable-wayland-on-ubuntu-22-04-desktop)
  * If your user (we assume username ```ubuntu``` in the following) has not sudo privileges
  ```
$ sudo usermod -aG sudo $USER
$ reboot    <= reboot is necessary if you make all the installation in one attempt for guaranteeing Ansible permissions
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

#### Docker installation

_**Note: docker can be installed according to [this](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) original guide. You can safely refer to that document. Below, we replicate it only for sake of completeness of this guide.**_

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

$ sudo groupadd docker   <=== to dla pewności
$ sudo usermod -aG docker $USER
$ newgrp docker          <=== dodaje usera do grupy w bieżącej powłoce (bez reboot)
  ```
  * Verify if docker works (check if you can run docker without sudo - should be possible)
  ```
$ docker run hello-world
  ```

## 5. Kolla-Ansible and OpenStack installation

_**Note: in the following, we assume ```ubuntu@labs:~/labs/ostack$``` to be the working (current) directory. When copy-pasting, adjust the commands according to your environment.**_

### Kolla-Ansible installation

The overall installatiuon procedure is the same as in the original [Kolla-Ansible guide for the 2023.1 release](https://docs.openstack.org/kolla-ansible/2023.1/user/quickstart.html). A couple of exceptions are a direct consequence of changing the status of this release to "unmaintained" by the Kolla-Ansible project withoud updating the status name in the original Kolla-Ansible guide and in one Kolla-Ansible configuration file (```stable``` is used instead of ```unmaintained```). Corrective changes in respective instructions are commented in the following.

  * Install Python packages, create and activate virtual environment
```
$ sudo apt-get update
$ sudo apt-get install -y git python3-dev libffi-dev gcc libssl-dev python3-venv
$ python3 -m venv kolla-2023.1
$ source kolla-2023.1/bin/activate
```

  - Note: deactivate active venv / completely delete given venv
```
$ deactivate
$ rm -r <venv-root-folder-name->
```

**Note: from now on, we work in virtual environment kolla-2023.1. The versions of all conponents installed are compatible with kolla-2023.1 and should be changed appropriately for other releases of kolla-ansible (follow respective Kolla-Ansible Quickstart section for each release https://docs.openstack.org/kolla-ansible/YOUR-RELEASE/user/quickstart.html)**

  * Install Kolla-Ansible in the active venv
```
$ pip install -U pip
$ pip install 'ansible>=6,<8'     ==>   for 2024.2: pip install 'ansible-core>=2.16,<2.17.99'
                                        for 2024.1: pip install 'ansible-core>=2.15,<2.16.99'
                                        for 2025.1: installing ansible is included in kolla-ansible installation so we skip this step for 2025.1
$ pip install git+https://opendev.org/openstack/kolla-ansible@unmaintained/2023.1
      ==> for 2024.1: pip install git+https://opendev.org/openstack/kolla-ansible@stable/2024.1
          for 2024.2: pip install git+https://opendev.org/openstack/kolla-ansible@stable/2024.2
```

> [!WARNING]
> If you see error message similar to _```error: pathspec 'stable/2023.1' did not match any file(s) known to git ERROR! Failed to switch a cloned Git repo `https://opendev.org/openstack/ansible-collection-kolla` to the requested revision `stable/2023.1`.```_, do not panic. You should go to [this repo](https://opendev.org/openstack/kolla-ansible) and check the name of the branch where your release is currently stored and where you will find its current name. Then change the name of your (supposed) branch (```@stable``` in the example) to the right one. To this end, edit local file: ```nano kolla-2023.1/share/kolla-ansible/requirements.yml```. Most probably you will have to change the name from ```stable/2023.1``` to ```unmaintained/2023.1```. In case of using other release than 2023.1 similar procedure will apply.

### Prepare configuration files for Kolla-Ansible

Now, generated by Kolla-Ansible there are several configuraton files with default settings. We have to customize them (and even add new files) to make the configuration appropriate for our cluster.

```
$ sudo mkdir -p /etc/kolla
$ sudo chown $USER:$USER /etc/kolla
```

  * Copy globals.yml and passwords.yml to /etc/kolla directory, and kolla-ansible inventory file to our working directory. We will customize these file for our deployment soon.

    We remind ```~/labs/ostack``` is current working directory.
    
```
$ sudo cp -r kolla-2023.1/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
$ sudo cp kolla-2023.1/share/kolla-ansible/ansible/inventory/* .
```

  * Install Ansible Galaxy dependencies (bez sudo)
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

  * Once we have prepared networking on our RaPis we should update the file ```/etc/hosts``` (on the management host) by adding our OpenStack hosts (RaPis); this information will be used by Ansible. For example:
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

### Configure Kolla-Ansible files for specific OpenStack depolyment

1. Prepare passwords.yml file

  * Generate passwors in file ```/etc/kolla/passwords.yml```

    **Note:** In case of `PermissionError: [Errno 13] Permission denied: '/etc/kolla/passwords.yml'` change file permissions `chown a+wr /etc/kolla/passwords.yml`. You can restore original permissions after that.
```
$ kolla-genpwd
```
  * change for human readable the admin password (search ```keystone_admin_password```) in file ```/etc/kolla/passwords.yml```
    (you can set it simple, e.g. ```admin``` as in our case)
```
$ sudo nano /etc/kolla/passwords.yml
```

2. Prepare inventory file ```multinode``` and feature configuration file ```globals.yml```

> [!IMPORTANT]
> Kolla-Ansible expects the user to assign OpenStack roles (`control`, `network`, `compute`, etc.) to hosts and specify role allocation in inventory file. Multiple experiments have shown that the roles `compute` and `network` MUST be separated when using with RPis 8GB RAM, i.e., allocated to different hosts. Otherwise the cluster becomes very unstable. In all-in-one installation (one RPi in the cluster) the node runs out of memory very soon after creating the first cirros VM and OpenStack crushes immediately. Even if one can login to the hosts after rebooting and some Kolla-Ansible containers run in healthy state, OpenStack as a whole is not functional anymore. Moreover, when there are dedicated compute nodes and the control/network node has not been assigned compute role, the compute/network node runs with almost zero free memory after creating second cirros VM. So even in multinode configuration merging `compute` and `network` should be avoided. These conclusions are reflected below in the example fragment of inventory file where `copmute` nad `network` roles are assigned to different hosts (ost4 and ost3, respectively).

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

# check if ansible can reach target hosts (our RaPis):
$ ansible -i multinode all -m ping
```
**Warning:** in case of Ansible having problems with ssh to reach your RPis check /etc/hosts for presence of the resoultion data. If you reinstalled the OS on the RPis, you probably have to delete file ~/.ssh/known_hosts. 

  * file globals.yml is in ```/etc/kolla/globals.yml```

    Take care of adjusting the attributes listed below. Activating an attribute requires uncommenting respective line (deleting the '#' sign opening the line). **DO NOT TOUCH the field ```#openstack_release: "some-identifier"```**.

    **NOTE 1: The value of ```kolla_internal_vip_address```** should be adjusted according to your environment. It must be an unused address in your network, and among others it will serve for accessing Horizon (OpenStack dashboard) to manage the OpenStack by the admin and to manage user stacks by the users (tenants).
    
    Note 2: Enabling ```enable_neutron_provider_networks``` is not required in this case as we will use only flat provider network (we do not configure VLANs in our example setup). But we can leave this feature enabled to mark where the setting has to be done should one want to use VLANs in the future.

    **Attributes to update**

    **Warning: kolla_internal_vip_address MUST be anassigned (unused) IP address in your network (in particular different from the address assigned to veth0)**
    
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
```
  Note: more details about OpenStack networking with Kolla-Ansible can be found [here](https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html).

  * prepare file `/etc/kolla/conf/nova/nova-compute.conf`
    Here, we actually only request that created virtual machines should be automatically brought to their previous state (e.g., ACTIVE) when the host is booted. This is convenient when we stop OpenStack (e.g., to shut down the cluster for some time) and start it again afterwards. The latter is described in section [Stop the cluster and start again](#stopstart-the-cluster-switch-off-not-destroy--and-start-again).

> [!NOTE]
> File /etc/kolla/config/nova/nova-compute.conf can overwrite the settings given in file `/etc/kolla/globals.yml`. In particular, we can set non-default virtualization type and cmu mode and type in [libvirt] section. Below, such customization for Raspberry Pi 4 is shown but commented out. It actually works for RPi4, but as of this writing (July 2025) it has not counter part for Raspberry Pi 5 (libvirt seems not to support cortex-a74 cpu model). QEMU emulation is slower than KVM and in fact the latter is the default option in Kolla-Ansible when processor architecture of hosts is aarch64. In such a case (aarch64) libvirt driver in nova-compute selects host-passthrough CPU mode as the only choice (see here for more on that). All in all, we use KVM for both Raspberry Pi 4 and 5 for which it iss sufficient to not change default settings for  in globals.yml and not specify [libvirt] section in `nova-compute.conf'.

<pre>
sudo mkdir -p /etc/kolla/config/nova
sudo tee /etc/kolla/config/nova/ << EOT
[DEFAULT]
resume_guests_state_on_host_boot = true

#for RaPi 5 enable cortex-a76 <== currently lack of support of RPi5 board in qemu
#for RaPi 4 enable cortex-a72
#[libvirt]
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
network_vlan_ranges = physnet1:100:200

[ml2_type_flat]
flat_networks = physnet1
EOT
</pre>

### Deploy OpenStack instance

**Warning:** In case of kolla-ansible below has problems with ssh to reach your RPis ("WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!"), check file /etc/hosts for the presence of resoultion data. If you have reinstalled the OS on the RPis, you have to **do delete** `rm ~/.ssh/known_hosts`.

```
kolla-ansible bootstrap-servers -i multinode 
kolla-ansible prechecks -i multinode
kolla-ansible deploy -i multinode
```

### Postdeployment and first instance

1. Postdeployment

  * Install `python-openstackclient` to access OpenStack commands in console
    ```bash
    pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2023.1
    #other options
      #pip install python-openstackclient -c https://opendev.org/openstack/requirements/raw/branch/unmaintained/2023.1/upper-constraints.txt
      #pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2024.1
    ```

  * Run post-deploy script, create appropriate directories and copy cloud.config file (it will be needed by install-runonce stript in a while to create your first instance): 

    ```
    kolla-ansible post-deploy
    sudo mkdir ~/.config 
    mkdir ~/.config/openstack
    #if you see error notification about unreachability of a file, do: $ chmod -R u+r <unreachable-directory-name>
    sudo cp /etc/kolla/clouds.yaml /etc/openstack/clouds.yaml
    cp /etc/kolla/clouds.yaml ~/.config/openstack/clouds.yaml
    ```
    
2. First checks - create first instance

   Your first instance will be created from openstak client command line. First, you will use a prepared script to create tenant network and other artifacts (e.h., VM image to create form) needed to create instances. Then, you will enable python-openstack client tool and create the instance from command line using openstack command line client (similar to creating pods in Kubernetes using kubctl).

  * use init-runonce script to create external network, subnetwork, router, download the image of cirros VM, and create a couple of flavors of VM in the admin project
    - init-runonce script files for various kolla-ansible/OpenStack releases are available in this repo. Select appropriate <kolla-ansible-release> and <vm-os-name>, possibly edit the file to change some settings according to your environment (e.g., external network address range). The file name format of those scripts is as follows:
   
      `init-runonce.<kolla-ansible-release>.<flavor-os-name>`
      
      Note 1: the original version of init-runonce script is generated by the command `kolla-ansible post-deploy`. We have to use slightly modified version of the script to match the requirements/limitations of our Raspberry Pi platform (e.g., we need aarch64 image, we can afford only quite small VM flavors, etc.).
      
      Note 2: currently tested and recommended settings are `<kolla-ansible-release>=2023.1` and `<flavor-os-name>=cirros`, so the recommended init-runonce script is available here:
      add the link ?????????

      - run the script
      ```bash
      ./init-runonce.2023.1.cirros
      ```
    Note: script init-runonce illustrates the use of several openstack client commands in bash. It is recommended to analyse it, possibly even echo-ing openstack client commands to see how they look like in plain. It may help you write your own commands if such a need arises in the future. Remember that `init-runonce` is designed to be run "as is" only once and subsequent runs will generate errors for all constructs having already been created. In such a case experimenting with it should better depend on only echo-ing openstack commands (as ready-to-run strings) but at the same time suppressing in the script their execution.

    After running `./init-runonce.20xy.z` the external and tenant networks, VM image, VM flavors and the default security group settings are all ready. The first instance can now be created in the following two steps:

  * source `admin-openrc.sh` script to enable python-openstackclient
    ``` bash
    source /etc/kolla/admin-openrc.sh
    ```
  * run the VM from openstack command line client
    ```
    openstack --os-cloud=kolla-admin server create --image cirros --flavor m1.tiny --key-name mykey --network demo-net cirros1
    ```
    - the following is only for our record, do not run it
      ```bash
      ./init-runonce.2023.1.alpine
      openstack --os-cloud=kolla-admin server create --image alpine --flavor m1.large --key-name mykey --network demo-net alpine1
      ```
      ```bash
      ./init-runonce.2023.1.ubuntu
      openstack --os-cloud=kolla-admin server create --image ubuntu --flavor m1.medium --key-name mykey --network demo-net ubuntu1
      ```
      
  * check the status of the instance
    ```
    openstack server list
    ```

## Managing your cluster 

### Stop/start the cluster (switch off, not destroy / and start again)

1. Switch-off
  * Stop containers
    ```
    kolla-ansible stop -i multinode --yes-i-really-really-mean-it
    ```
    Note: if there is a notification about certain hosts being ureachable, run the stop command again.
    
  * Switch off RbPis
    - write and run a bash command containing the following script (we assume $CLUSTER_INVENTORY_FILE is a parameter passed to the command)
    ```bash
    #!/bin/bash
    # check the following links for the ansible command options used:
    # https://www.jeffgeerling.com/blog/2018/reboot-and-wait-reboot-complete-ansible-playbook
    # https://www.middlewareinventory.com/blog/ansible_wait_for_reboot_to_complete/
    CLUSTER_FILE="multinode"
    ansible all -i $CLUSTER_INVENTORY_FILE --limit 'baremetal' -b -B 1 -P 0 -m shell -a "sleep 5 && shutdown now" -b   
    ```
    
 2. Start the cluster
  * Power on
  * Management host: activate virtual environment of your installation and run
    ```
    kolla-ansible deploy-containers -i multinode 
    ```

### Destroy your cluster

To reinstall your cluster in case of failure, first destroy currnet installation. To this end, stop or delete all running instances in the cluster and then run
```
kolla-ansible destroy --yes-i-really-really-mean-it -i multinode
```

