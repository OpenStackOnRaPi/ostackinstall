# OStackInstallRaPi
This repo describes how to install OpenStack on raspberry Pi cluster using Kolla-Ansible.

Currently, Raspberry Pi 4 is assumed. It is expected that extensions (if any) needed for Rraspberry Pi 5 will be included in the future once we test this option sufficiently well.

## Assumptios

All procedures described herein refer to HW and SW setup of the cluster as specified below

1. Raspberry Pi 4
   * (1x4GB RAM + 1x8GB RAM) or (2x4GB RAM+2*8GB RAM)
   * all Pi are equipped with 32GB SD disk
   * Note: we believe that for testing purposes a basic configuration of Kolla-Ansible OpenStack should work but we have not tested this option.
2. SW
   * OS: Raspberry Pi OS Light - root of Debian 12 (Bookworm)
   * Kolla-Ansible 2023.1; respective envirnment components according to 
   * Note: newer releases of Kolla-Ansible will be tried in the future (2023.1 got status "unmaintained" during our trials) and we'll update these instructions accordingly after positive tests
4. Network:
   * the Pis are equipped with 802.3af/at PoE HAT from Waveshare
   * they are powered form TP-Link TL-SG105PE switch
   * TP-Link switch is connected to a local router with DHCP enabled to separate the network of OpenStack DC frome the rest of the network environment
   * Note: other PoE HATs for Raspberry Pi 4 and other PoE switches should work, too
  
   * 

 
