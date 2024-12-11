# OpenStack installation on Raspberry Pi
This repo describes how to install OpenStack on a Raspberry Pi cluster using Kolla-Ansible.

## Table of contents

1. [Introduction](#introduction)
2. [Assumptions](#assumptions)
3. [Raspberry Pi preparation](#raspberry-pi-preparation)
4. [Management host preparation](#management-host-preparation)
5. [Kolla-ansible and OpenStack instalation](#kolla-ansible-and-openstack-instalation) 


## Introduction

The scope of application of our clusters is education. A cluster of this type allows us to present/explain various features/concepts of OpenStack that are hard or impossible to show using AIO or virtualized setups. Many of them are related to the administration of OpenStack data centre - a domain of activity that is hidden from regular users in "normal" DCs. For example, the management of provider networks where the admin needs to configure VLANs in the physical network of the data centre and declare them in OpenStack config file. Considering that the time needed to practice and learn even basic things is non negigible, a decent amount of resources is needed to serve a dozen or more student teams. One does not need to allocate dozens of servers worth thousands of $ each for that.

Currently, Raspberry Pi 4 is assumed as the HW implementation base. It is expected that extensions (if any) needed for Rraspberry Pi 5 will be added to this guide in the future once we test RaPi 5 setup sufficiently well.

## Assumptios

All procedures described herein refer to HW and SW setup of the cluster as specified below:

1. Raspberry Pi 4
   * [1x4GB RAM + 1x8GB RAM] or [2x4GB RAM+2*8GB RAM]
   * all Pi are equipped with 32GB SD disk
   * Note: we believe that a single RaPi 8GB RAM and the default configuration of Kolla-Ansible OpenStack should work for basic evaluation, but we have not tested this option.
2. SW
   * OS: Raspberry Pi OS Light - root of Debian 12 (Bookworm)
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

## Management host preparation

## Kolla-Ansible and OpenStack instalation




 
