## Overview

VNET Gateway configured for a route-based VPN. A second VNET simulates on-prem running an OPNsense firewall as the VPN termination device.

## Notes

* Variables defined at start of script, change as required
* VMs provisioned with auto-shutdown at 22:00 UTC
* Provisions public IPs for VMs and NSG rule to allow SSH admin access (could use serial console instead!)

## Deploy
> The variables are only persistent within the azcli session. If you need you come back to this in a later session, rerun the variables section

## Useful Commands

<pre lang="...">
# get public ip of vms & lb
az network public-ip show -g $resourcegroup -n $vm1publicip --query "{address: ipAddress}"
az network public-ip show -g $resourcegroup -n $vm2publicip --query "{address: ipAddress}"
az network public-ip show -g $resourcegroup -n $elbpublicip --query "{address: ipAddress}"
  
# stop vm
az vm deallocate -g $resourcegroup -n $vm1name --no-wait
az vm deallocate -g $resourcegroup -n $vm2name --no-wait

# start vm
az vm start -g $resourcegroup -n $vm1name --no-wait
az vm start -g $resourcegroup -n $vm2name --no-wait
</pre>

## Destroy

<pre lang="...">
# delete all resources
az group delete -n $resourcegroup
</pre>

