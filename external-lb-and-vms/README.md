# External Load-Balancer and VMs with Web Server

## Overview

2 VMs running NGINX web server behind an external Load-Balancer, provisioned with Azure CLI:

![](external-lb-and-vms.png)

## Notes

* Variables defined at start of script, change as required
* VMs provisioned with auto-shutdown configured for 22:00 UTC
* NSG rule provisioned to allow SSH access from anywhere for admin access
* NSG rule provisioned to allow HTTP access from anywhere for web testing through LB

## Provision

## Useful Commands

## Destroy


