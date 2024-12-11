# OpenStack installation on Raspberry Pi
This repo describes how to install OpenStack on a Raspberry Pi cluster using Kolla-Ansible.

## Table of contents

1. [Introduction](#introduction)
2. [Assumptions](#assumptions)
3. [Raspberry Pi preparation](#raspberry-pi-preparation)
   1. [System configuration](#system-configuration)
   2. [Network configuration](#network-configuration)
5. [Management host preparation](#management-host-preparation)
6. [Kolla-ansible and OpenStack installation](#kolla-ansible-and-openstack-installation) 

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

### System configuration

1. Flash the OS (Raspberry Pi OS Lite (64bit), a port of Debian 12 (Bookworm) with no desktopp environment) onto microSD card. We recommend using Raspberry Pi imager.
   * make sure password authentication for ssh access is enabled (the instructions given below fit this authentication method)
   * it is recommended to set the value of host name, user name and password as you will use afterwards in Kolla-Ansible playbooks. In the examples below, we set "ubuntu" for both the user name and password, and use the convention ost01, ost02, ... to set Raspbbery Pi host name.

2. Assuming we set user name ubontu (otherwise, adapt the flollowing)

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

#check connectivity
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

#run to check CPU temperature
$ sensors
```

8. Enable packet forwarding on the RaPi

```
$ sudo nano /etc/sysctl.conf

# uncomment the line: net.ipv4.ip_forward=1
# save the file, quit and check the setting:

$ sudo sysctl -p
```

9. Install qemu-system-arm (qemu-kvm) - critical for enabling virtualization

   Note: you can check:
    sudo apt install --simulate qemu-kvm
    sudo apt show qemu-system-arm
```
$ sudo apt-get update && sudo apt-get upgrade
$ sudo apt-get install -y qemu-system-arm
```    

10. Upgrade for any case, reboot

```
$ sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get autoremove -y && sudo reboot    
```

### Network configuration



## Management host preparation

## Kolla-Ansible and OpenStack installation




 
