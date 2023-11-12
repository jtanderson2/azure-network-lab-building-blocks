# VNET / Subnet / NSG / VM

## Overview

A basic building block that will provision the following topology with az cli:

![](vnet-subnet-nsg-vm.png)

## Notes

* Variables defined at start of script, change as required
* VM is provisioned with auto-shutdown configured for 22:00 UTC
* NSG rule is provisioned to allow SSH access from anywhere 
