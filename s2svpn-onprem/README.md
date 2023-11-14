## Overview

VNET Gateway configured for a route-based VPN. A second VNET simulates on-prem running an OPNsense firewall as the VPN termination device.

tba

## Notes

* Variables defined at start of script, change as required
* VMs provisioned with auto-shutdown at 22:00 UTC
* Provisions public IPs for VMs and NSG rule to allow SSH admin access (could use serial console instead!)

## Deploy
> The variables are only persistent within the azcli session. If you need you come back to this in a later session, rerun the variables section

<pre lang="...">
# define variables
resourcegroup="rg-ddd-001"
location="uksouth"
vnet="vnet-ddd-001"
vnetpfx="10.4.0.0/16"
snet="snet-ddd-001"
snetpfx="10.4.0.0/24"
gwpfx="10.4.255.0/27"
nsg="nsg-ddd-001"
nic="nic-vm-ddd-001"
privateip="10.4.0.10"
publicip="pip-vm-ddd-001"
vpnpublicip="pip-vpn-ddd-001"
vpngw="vpn-ddd-001"
vmname="vm-ddd-001"
vmimage="OpenLogic:CentOS:7.5:latest"
vmsize="Standard_B1ls"
vmuser="azureuser"
vmpassword="Msft123Msft123"

# create resource group
az group create -n $resourcegroup --location $location

# create vnet and subnet
az network vnet create -g $resourcegroup -n $vnet --location $location --address-prefixes $vnetpfx --subnet-name $snet --subnet-prefix $snetpfx

# create nsg
az network nsg create -g $resourcegroup -n $nsg

# create nsg rule to allow ssh
az network nsg rule create -g $resourcegroup --nsg-name $nsg -n AllowSSH --priority 1000 --source-address-prefixes '*' --source-port-ranges '*' --destination-address-prefixes $snetpfx --destination-port-range 22 --access Allow --protocol Tcp --description "Allow SSH"

# associate nsg with subnet
az network vnet subnet update -g $resourcegroup -n $snet --vnet-name $vnet --network-security-group $nsg

# create public ip for vm
az network public-ip create -n $publicip -g $resourcegroup --location $location --sku standard

# create nic for vm, create private ip and and assign public ip
az network nic create -g $resourcegroup -n $nic --location $location --subnet $snet --private-ip-address $privateip --vnet-name $vnet --public-ip-address $publicip

#create linux vm and associate with nic
az vm create -g $resourcegroup -n $vmname --image $vmimage --size $vmsize --admin-username $vmuser --admin-password $vmpassword --nics $nic

#auto-shutdown vm at 10:00 UTC
az vm auto-shutdown -g $resourcegroup -n $vmname --time 2200

# create gateway subnet
az network vnet subnet create -g $resourcegroup -n GatewaySubnet --vnet-name $vnet --address-prefix $gwpfx

# create public ip for vpn gateway
az network public-ip create -n $vpnpublicip -g $resourcegroup --location $location --sku standard

# create vpn gateway
az network vnet-gateway create -g $resourcegroup -n $vpngw -l $location --public-ip-address $vpnpublicip --vnet $vnet --gateway-type Vpn --sku VpnGw1 --vpn-type RouteBased --no-wait
</pre>  

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

