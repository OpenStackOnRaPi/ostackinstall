# VLAN provider network setup

This folder contains network configuration files to enhance the network setup of our OpenStack cluster by introducing VLAN-based provider networks. Please refer to respective [section](../README.md#7-vlan-provider-networks---part-2-enabling-and-using-vlan-provider-networks) in the main user guide for details.

> [!Warning]
> Remember that it is necessary to only replace the files provided here **keeping untouched remaining files** already present in the `/etc/systemd/network` directory. Otherwise you will cut off remote access to your RPis unless you have enabled WiFi on the boards. Or you will need to directly attach to the boards using monitor, keybord and mouse.
