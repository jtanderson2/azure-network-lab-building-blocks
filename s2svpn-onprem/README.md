## Overview

VNET Gateway configured for a route-based VPN. A second VNET simulates on-prem running an OPNsense firewall as the VPN termination device.

tba

## Notes

* Variables defined at start of script, change as required
* VMs provisioned with auto-shutdown at 22:00 UTC
* Provisions public IPs for VMs and NSG rule to allow SSH admin access (could use serial console instead!)

## Deploy OPNSense in 'OnPrem' VNET
Use [this](https://github.com/dmauser/opnazure) excellent template from Daniel Mauser to deploy the OPNsense NVA in its' own VNET with Trusted and Untrusted subnets.

Hit the 'Deploy to Azure' button from the repo which just works out the box. The only parameter you need to add in the portal is the resource group; create a new group named **rg-vpn-001** then accept all other defaults then **Create**

## Deploy the Rest
> The variables are only persistent within the azcli session. If you need you come back to this in a later session, rerun the variables section

<pre lang="...">
# define global variables
resourcegroup="rg-vpn-001"
location="uksouth"
vmimage="OpenLogic:CentOS:7.5:latest"
vmsize="Standard_B1ls"
vmuser="azureuser"
vmpassword="Msft123Msft123"

# define azure-side variables
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
routetable="route-ddd"

# define variables for 'on-prem' vm
opnvnet="OPN-VNET"
opnsnet="Trusted-Subnet"
opnnsg="nsg-opn-001"
opnnic="nic-vm-opn-001"
opnprivateip="10.0.1.10"
opnpublicip="pip-vm-opn-001"
opnvmname="vm-opn-001"
opnroutetable="route-opn"

# create azure-side vnet and subnet
az network vnet create -g $resourcegroup -n $vnet --location $location --address-prefixes $vnetpfx --subnet-name $snet --subnet-prefix $snetpfx

# create azure-side nsg
az network nsg create -g $resourcegroup -n $nsg

# create azure-side nsg rule to allow ssh
az network nsg rule create -g $resourcegroup --nsg-name $nsg -n AllowSSH --priority 1000 --source-address-prefixes '*' --source-port-ranges '*' --destination-address-prefix $snetpfx --destination-port-range 22 --access Allow --protocol Tcp --description "Allow SSH"

# associate azure-side nsg with subnet
az network vnet subnet update -g $resourcegroup -n $snet --vnet-name $vnet --network-security-group $nsg

# create azure-side public ip for vm
az network public-ip create -n $publicip -g $resourcegroup --location $location --sku standard

# create azure-side nic for vm, create private ip and and assign public ip
az network nic create -g $resourcegroup -n $nic --location $location --subnet $snet --private-ip-address $privateip --vnet-name $vnet --public-ip-address $publicip

# create azure-side linux vm and associate with nic
az vm create -g $resourcegroup -n $vmname --image $vmimage --size $vmsize --admin-username $vmuser --admin-password $vmpassword --nics $nic

# auto-shutdown azure-sidevm at 10:00 UTC
az vm auto-shutdown -g $resourcegroup -n $vmname --time 2200

# create an azure-side route-table
az network route-table create -g $resourcegroup -n $routetable

# create an azure-side route
az network route-table route create -g $resourcegroup --route-table-name $routetable -n OPN --next-hop-type VirtualAppliance --address-prefix 10.0.1.0/24 --next-hop-ip-address 10.4.255.4

# create azure-side gateway subnet
az network vnet subnet create -g $resourcegroup -n GatewaySubnet --vnet-name $vnet --address-prefix $gwpfx

# create azure-side public ip for vpn gateway
az network public-ip create -n $vpnpublicip -g $resourcegroup --location $location --sku standard

# create azure-side vpn gateway
az network vnet-gateway create -g $resourcegroup -n $vpngw -l $location --public-ip-address $vpnpublicip --vnet $vnet --gateway-type Vpn --sku VpnGw1 --vpn-type RouteBased --no-wait

# create onprem nsg
az network nsg create -g $resourcegroup -n $opnnsg

# create onprem nsg rule to allow ssh
az network nsg rule create -g $resourcegroup --nsg-name $opnnsg -n AllowSSH --priority 1000 --source-address-prefixes '*' --source-port-ranges '*' --destination-address-prefixes $opnprivateip --destination-port-range 22 --access Allow --protocol Tcp --description "Allow SSH"

# associate onprem nsg with subnet
az network vnet subnet update -g $resourcegroup -n $opnsnet --vnet-name $opnvnet --network-security-group $opnnsg

# create onprem public ip for vm
az network public-ip create -n $opnpublicip -g $resourcegroup --location $location --sku standard

# create onprem nic for vm, create private ip and and assign public ip
az network nic create -g $resourcegroup -n $opnnic --location $location --subnet $opnsnet --private-ip-address $opnprivateip --vnet-name $opnvnet --public-ip-address pip-vm-opn-001

# create onprem linux vm and associate with nic
az vm create -g $resourcegroup -n $opnvmname --image $vmimage --size $vmsize --admin-username $vmuser --admin-password $vmpassword --nics $opnnic

# auto-shutdown azure-sidevm at 10:00 UTC
az vm auto-shutdown -g $resourcegroup -n $opnvmname --time 2200

# create an onprem route-table
az network route-table create -g $resourcegroup -n $opnroutetable

# create an azure-side route
az network route-table route create -g $resourcegroup --route-table-name $opnroutetable -n DDD --next-hop-type VirtualAppliance --address-prefix 10.4.0.0/24 --next-hop-ip-address 10.0.1.4
</pre>  

## Useful Commands

<pre lang="...">
# get public ip of vms & lb
az network public-ip show -g $resourcegroup -n $publicip --query "{address: ipAddress}"
az network public-ip show -g $resourcegroup -n $opnpublicip --query "{address: ipAddress}"
az network public-ip show -g $resourcegroup -n $vpnpublicip --query "{address: ipAddress}"
az network public-ip show -g $resourcegroup -n OPNsense-PublicIP --query "{address: ipAddress}"

  
# stop vm
az vm deallocate -g $resourcegroup -n $vmname --no-wait
az vm deallocate -g $resourcegroup -n $opnvmname --no-wait

# start vm
az vm start -g $resourcegroup -n $vmname --no-wait
az vm start -g $resourcegroup -n $opnvmname --no-wait
</pre>

## Destroy

<pre lang="...">
# delete all resources
az group delete -n $resourcegroup
</pre>

