# OpenStack on Raspberry Pi
In this repo we describe how to install OpenStack on a Raspberry Pi cluster using Kolla-Ansible.

Note: currently, only Raspberry Pi 4B and Kolla-Ansible/OpenStack release 2023.1 can be used. Newer versions of Kolla-Ansible and RPi 5 can not be recommended at the time of this writing (Dec. 2024) for problems with virtualization. There seem to be incompatibilities between the RPi platform and qemu/libvirt and/or lack of support for RPi5 in qemu. Therefore do not use the parts in this guide that relate to release other than 2023.1.

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
   3. [Configure Kolla-Ansible files for specific OpenStack Depolyment](configure-kolla-ansible-files-for-specific-openstack-depolyment)
   4. [Deploy OpenStack instance](#deploy-openstack-instance)
   5. [Postdeployment and first instance](#postdeployment-and-first-instance)
   6. [Stop the cluster and start again](#stop-the-cluster-switch-off-not-destroy-and-start-again)
   7. [Destroy your cluster](#destroy-your-cluster)


   
## 1. Introduction

The scope of application of our clusters is education. A cluster of this type allows us to present/explain various features/concepts of OpenStack, some of them being hard or impossible to show using AIO or virtualized OpenStack setups. Many of such features are related to the administration of OpenStack data centre - a domain of activity that is hidden from regular users in "normal" DCs. For example, the management of provider networks where the admin needs to configure VLANs in the physical network of the data centre and declare them in OpenStack and Kolla-Ansible config files. Considering that the time needed to practice and learn even basic things is non negigible, a decent amount of resources is needed to serve a dozen or more student teams in limited time. With our approach, one does not need to allocate dozens of servers each worth thousands of $ for that.

Currently, Raspberry Pi 4 is assumed as the HW implementation base. It is expected that extensions (if any) needed for Raspberry Pi 5 will be added to this guide once we test RaPi 5 setup sufficiently well.

Worth of noting is also that our clusters are a perfect base for experimenting with Kubernetes. In our lab, we use bare-metal setup of K3s - an excellent match for Raspbbery Pi.

## 2. Assumptions

All procedures described in this guide assume HW and SW setup of the cluster as specified below:

1. Raspberry Pi 4
   * amount: [1x4GB RAM + 1x8GB RAM] or [2x4GB RAM + 2*8GB RAM] per cluster
   * all Pi are equipped with 32GB SD disk
   * Note: we believe that a single RaPi 8GB RAM in all-in-one setup of Kolla-Ansible OpenStack should work well for basic evaluation, but we have not tested this option.
2. SW
   * OS: Raspberry Pi OS Lite (64bit), a port of Debian 12 (Bookworm) with no desktopp environment
   * Kolla-Ansible 2023.1; respective envirnment components according to 
   * Note: newer releases of Kolla-Ansible will be tried in the future (2023.1 has got status "unmaintained" recently) and we'll update these instructions accordingly after completing the tests 
3. Network:
   * the Pis are equipped with 802.3af/at PoE HAT from Waveshare
   * they are powered form TP-Link TL-SG105PE switch
   * TP-Link switch is connected to a local router with DHCP enabled to separate the network of OpenStack DC frome the rest of the network environment
   * **reserve a pool of IP addresses for the use by OpenStack**; 20 addresses will be sufficient for our purposes. They **MUST NOT** be managed by the DHCP server. Four of them will be assigned by you to the RbPis using netplan (see [here](https://github.com/OpenStackOnRaPi/OStackInstallRaPi/blob/main/README.md#configuration-description)), and one will be allocated as the so-called ```kolla_internal_vip_address``` (see [here](https://github.com/OpenStackOnRaPi/OStackInstallRaPi/blob/main/README.md#configure-kolla-ansible-files-for-specific-openstack-depolyment)). Remaining addresses will serve as ```floating IP addresses``` for accessing created instances from the outside of your cloud.
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

  ```
$ sudo apt-get remove unattended-upgrades -y && sudo apt-get update -y && sudo apt-get dist-upgrade -y
  ```

5. Install for the use by Ansible

  ```
$ sudo apt-get install sshpass -y 
$ sudo apt-get install ufw -y     <=== needed on debian, not necessary on Ubuntu
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
 IP 192.168.1.6x/24               no IP addr assigned (Kolla-Ansible requires that)
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

1. Stop NetworkManager, and and start systemd-networkd

   _Note: for some historical reasons, we use networkd to have persistent configuration of network devices on our RaPis; one can use NetworkManager for this, but it will be necessary to convert respective network constructs from networkd to NetworkManager notation (NetworkManager notation is different form the one used by networkd)._

```
$ sudo systemctl stop NetworkManager
$ sudo systemctl disable NetworkManager
$ sudo systemctl enable systemd-networkd && sudo systemctl start systemd-networkd
$ sudo systemctl status systemd-networkd                  <= should be Active: active (running) ... 
```

2. Install and enable netplan (ref. https://installati.one/install-netplan.io-debian-12/?expand_article=1)

```
$ sudo apt-get update && sudo apt-get -y install netplan.io
```

3. Main host network configurations

   _**NOTE: this setup is prepared for flat provider network only in your OpenStack DC. To allow for VLAN provider networks, additional configurations are needed for ```eth0```, ```brmux``` and ```veth1br``` to set VLANs that should be served by those devices. Respective configurations of VLANs should also be introduced in the TP-Link switch. If you are interested in setting VLAN provider networks in your cluster, please contact the instructor for more info.**_

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

    **NOTE 1: adjust the IP address of veth0 in each RaPi according to you network setup.**

    **NOTE 1:** in case of problems during `netplan generate` (or `netplan apply`) check the format of your file `/etc/netplan/50-cloud-init.yaml` - it's YAML and spaces matter.

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
          - 192.168.1.1     # Linksys dhcp server
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

  * deploy the changes permanently - create and configure devices
    Note: you will loose connectivity to your RaPi because of the change of IP address. To continue, ssh again using the new address.

```
$ sudo netplan generate
$ sudo netplan apply
(ssh again)
# check the connectivity
$ ping wp.pl
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

**Note: from now on, we work in virtual environment kolla-2023.1**

  * Install Kolla-Ansible in the active venv
```
$ pip install -U pip
$ pip install 'ansible>=6,<8'        ==>    for 2024.2: pip install 'ansible-core>=2.16,<2.17.99'
                                            for 2024.1: pip install 'ansible-core>=2.15,<2.16.99'
$ pip install git+https://opendev.org/openstack/kolla-ansible@unmaintained/2023.1
      ==> for 2024.1: pip install git+https://opendev.org/openstack/kolla-ansible@stable/2024.1
          for 2024.2: pip install git+https://opendev.org/openstack/kolla-ansible@stable/2024.2
```
_**Note for the future:** if you see a notification simlar to "WARNING: Did not find branch or tag '@stable/2023.1', assuming revision or ref", you should go to [this repo](https://opendev.org/openstack/kolla-ansible) and check the name of the branch where your release is currently stored in. Then change the name of supposed branch (```@stable``` in the example) to the right one (probably it will be ```@unmaintained```)._

### Generate configuration files for Kolla-Ansible (default templates)

```
$ sudo mkdir -p /etc/kolla
$ sudo chown $USER:$USER /etc/kolla
```

  * Copy globals.yml and passwords.yml to /etc/kolla directory, and kolla-ansible inventory file to our working directory. We will configure these file for our deployment soon.

    We remind ```~/labs/ostack``` is current working directory.
    
```
$ cp -r kolla-2023.1/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
$ cp kolla-2023.1/share/kolla-ansible/ansible/inventory/* .
```

  * Install Ansible Galaxy dependencies (bez sudo)
```
$ kolla-ansible install-deps
```
  **WARNING:** if you see error message _```error: pathspec 'stable/2023.1' did not match any file(s) known to git ERROR! Failed to switch a cloned Git repo `https://opendev.org/openstack/ansible-collection-kolla` to the requested revision `stable/2023.1`.```_, then you need to edit a file: ```nano kolla-2023.1/share/kolla-ansible/requirements.yml``` and change the branch name from ```stable/2023.1``` to ```unmaintained/2023.1```).

  * Update Ansible configuration file ```ansible.cfg``` (it can be kept in the working directory or in directory ```/etc/ansible/ansible.cfg```)
```
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
```
$ kolla-genpwd
```
  * change for human readable the admin password (search ```keystone_admin_password```) in file ```/etc/kolla/passwords.yml```
    (you can set it simple, e.g. ```admin``` as in our case)
```
$ sudo nano /etc/kolla/passwords.yml
```

2. Prepare inventory file ```multinode``` and feature configuration file ```globals.yml```

  * file ```multinode``` is in the working directory
```
$ sudo nano multinode

# for every appearance of each node provide ansible directives ansible_user/password/become, e.g.:
[control]
ost04 ansible_user=ubuntu ansible_password=ubuntu ansible_become=true

[network]
ost04

[compute]
ost[01:03] ansible_user=ubuntu ansible_password=ubuntu ansible_become=true
ost04

[monitoring]
ost04

[storage]
#storage01

# check if ansible can reach target hosts (our RaPis):
$ ansible -i multinode all -m ping
```

  * file globals.yml is in ```/etc/kolla/globals.yml```

    Take care of adjusting the attributes listed below. Activating an attribute requires uncommenting respective line (deleting the '#' sign opening the line). **DO NOT TOUCH the field ```#openstack_release: "some-identifier"```**.

    **NOTE 1: The value of ```kolla_internal_vip_address```** should be adjusted according to your environment. It must be an unused address in your network, and among others it will serve for accessing Horizon (OpenStack dashboard) to manage the OpenStack by the admin and to manage user stacks by the users (tenants).
    
    Note 2: Enabling ```enable_neutron_provider_networks``` is not required in this case as we will use only flat provider network (we do not configure VLANs in our example setup). But we can leave this feature enabled to mark where the setting has to be done should one want to use VLANs in the future.

    **Attributes to update**
```
$ sudo nano /etc/kolla/globals.yml
...
kolla_base_distro: "debian"
openstack_tag_suffix: "-aarch64"
kolla_internal_vip_address: "192.168.1.60"
network_interface: "veth0"
neutron_external_interface: "veth1"
nova_compute_virt_type: "qemu" 
enable_neutron_provider_networks: "yes"
```
  Note: more details about OpenStack networking with Kolla-Ansible can be found [here](https://docs.openstack.org/kolla-ansible/latest/reference/networking/neutron.html).

  * prepare file `/etc/kolla/conf/nova/nova-compute.conf`

<pre>
sudo mkdir -p /etc/kolla/config/nova
sudo tee /etc/kolla/config/nova/nova-compute.conf << EOT
[DEFAULT]
resume_guests_state_on_host_boot = true

## according to OpenEuler needed for ARM
#for RaPi 5 enable cortex-a76 <== currently lack of support of RPi5 board in qemu
#for RaPi 4 enable cortex-a72
[libvirt]
virt_type = qemu
cpu_mode = custom
cpu_model = cortex-a72
#cpu_model = cortex-a76  
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

```bash
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

  * In not present, create appropriate directories. Run: 

    ```bash
    kolla-ansible post-deploy
    sudo mkdir ~/.config 
    mkdir ~/.config/openstack
    #if you see error notification about unreachability of a file, do: $ chmod -R u+r <unreachable-directory-name>
    sudo cp /etc/kolla/clouds.yaml /etc/openstack/clouds.yaml
    cp /etc/kolla/clouds.yaml ~/.config/openstack/clouds.yaml
    ```
    
2. First checks - create first instance

   First instance will be created from command line. To this end you have to enable python-openstackclient tool. After that, you will be able to create the instance.
   
  * source `admin-openrc.sh` script to enable python-openstackclient
    ``` bash
    source /etc/kolla/admin-openrc.sh
    ```

  * run init-runonce script that creates external network, subnetwork, router, downloads img image of cirros VM, and creates a couple of flavors in the admin project
    - init-runonce script files for various kolla-ansible/OpenStack releases are available under following links:
      - [init-runonce.2023.1](/init-runonce.2023.1)
      - [init-runonce.2024.1](/init-runonce.2024.1)
     
    After running ./init-runonce.20xy.z, the first instance can be created:

  * create first instance
    ```bash
    openstack --os-cloud=kolla-admin server create --image cirros --flavor m1.tiny --key-name mykey --network demo-net demo1

    # check the status of the instance
    openstack server list
    ```

### Stop the cluster (switch off, not destroy) and start again

1. Switch-off
  * Stop containers
    ```kolla-ansible stop -i multinode --yes-i-really-really-mean-it```
    Note: if there is a notification about certain hosts being ureachable, run the stop command again.
  * Switch off RbPis
    - adapt if needed and run the script
    ```bash
    #!/bin/bash
    # check the following links for the ansible command options used:
    # https://www.jeffgeerling.com/blog/2018/reboot-and-wait-reboot-complete-ansible-playbook
    # https://www.middlewareinventory.com/blog/ansible_wait_for_reboot_to_complete/
    CLUSTER_FILE="multinode"
    ansible all -i $CLUSTER_FILE --limit 'baremetal' -b -B 1 -P 0 -m shell -a "sleep 5 && shutdown now" -b   
    ```
    
 2. Start the cluster
  * Power on
  * Management host: activate virtual environment of your installation and run
    ```bash
    kolla-ansible deploy-containers -i multinode 
    ```

### Destroy your cluster

To reinstall your cluster in case of failure, first destroy currnet installation. Run
```bash
kolla-ansible destroy --yes-i-really-really-mean-it -i multinode
```

