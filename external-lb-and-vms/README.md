# External Load-Balancer and VMs with Web Server

## Overview

2 VMs running a web server behind an external Load-Balancer.

![](external-lb-and-vms.png)

## Notes

* Variables defined at start of script, change as required
* VMs provisioned with auto-shutdown at 22:00 UTC
* NSGs rule provisioned to allow SSH admin access to VMs and HTTP testing through LB

## Deploy

<pre lang="...">
# define global variables
resourcegroup="rg-ccc-001"
location="uksouth"
vnet="vnet-ccc-001"
vnetpfx="10.3.0.0/16"
snet="snet-ccc-001"
snetpfx="10.3.0.0/24"
nsg="nsg-ccc-001"
vmuser="azureuser"
vmpassword="Msft123Msft123"
vmimage="OpenLogic:CentOS:7.5:latest"
vmsize="Standard_B1ls"

# define vm1 variables
vm1nic="nic-vm-ccc-001"
vm1privateip="10.3.0.10"
vm1publicip="pip-vm-ccc-001"
vm1name="vm-ccc-001"

# define vm2 variables
vm2nic="nic-vm-ccc-002"
vm2privateip="10.3.0.11"
vm2publicip="pip-vm-ccc-002"
vm2name="vm-ccc-002"

# define elb variables
elbname="elb-ccc-001"
elbpublicip="pip-elb-ccc-001"
elbfrontend="fe-elb-ccc-001"
elbbackend="be-elb-ccc-001"
elbprobe1="probe-http"
elbrule="rule-http"

# create resource group
az group create -n $resourcegroup --location $location

# create vnet and subnet
az network vnet create -g $resourcegroup -n $vnet --location $location --address-prefixes $vnetpfx --subnet-name $snet --subnet-prefix $snetpfx

# create nsg
az network nsg create -g $resourcegroup -n $nsg

# create nsg rule to allow ssh and http
az network nsg rule create -g $resourcegroup --nsg-name $nsg -n AllowSSH-HTTP --priority 1000 --source-address-prefixes '*' --source-port-ranges '*' --destination-address-prefixes $snetpfx --destination-port-ranges 22 80 --access Allow --protocol Tcp --description "Allow SSH and HTTP"

# associate nsg with subnet
az network vnet subnet update -g $resourcegroup -n $snet --vnet-name $vnet --network-security-group $nsg

# create public ip for vm1
az network public-ip create -n $vm1publicip -g $resourcegroup --location $location --sku standard

# create nic for vm1, create private ip and and assign public ip
az network nic create -g $resourcegroup -n $vm1nic --location $location --subnet $snet --private-ip-address $vm1privateip --vnet-name $vnet --public-ip-address $vm1publicip

#create linux vm1 and associate with nic
az vm create -g $resourcegroup -n $vm1name --image $vmimage --size $vmsize --admin-username $vmuser --admin-password $vmpassword --nics $vm1nic

#auto-shutdown vm1 at 10:00 UTC
az vm auto-shutdown -g $resourcegroup -n $vm1name --time 2200

# create public ip for vm2
az network public-ip create -n $vm2publicip -g $resourcegroup --location $location --sku standard

# create nic for vm2, create private ip and and assign public ip
az network nic create -g $resourcegroup -n $vm2nic --location $location --subnet $snet --private-ip-address $vm2privateip --vnet-name $vnet --public-ip-address $vm2publicip

#create linux vm2 and associate with nic
az vm create -g $resourcegroup -n $vm2name --image $vmimage --size $vmsize --admin-username $vmuser --admin-password $vmpassword --nics $vm2nic

#auto-shutdown vm2 at 10:00 UTC
az vm auto-shutdown -g $resourcegroup -n $vm2name --time 2200
  
# create external load-balancer
az network lb create -g $resourcegroup -n $elbname --location $location --sku standard --frontend-ip-name $elbfrontend --public-ip-address $elbpublicip --backend-pool-name $elbbackend

# update load-balancer backend pool
az network lb address-pool update -g $resourcegroup --lb-name $elbname -n $elbbackend --vnet $vnet --backend-addresses "[{name:addr1,ip-address:$vm1privateip},{name:addr2,ip-address:$vm2privateip,subnet:$snet}]"

# configure load-balancer http probe
az network lb probe create -g $resourcegroup --lb-name $elbname -n $elbprobe1 --protocol http --port 80 --path /

# configure load-balancer rule
az network lb rule create -g $resourcegroup --lb-name $elbname -n $elbrule --protocol Tcp --frontend-ip $elbfrontend --frontend-port 80 --backend-pool-name $elbbackend --backend-port 80
</pre>

## Install Web Server on VMs
SSH to each VM and run the following command (or use the serial console from the portal):

<pre lang="...">
sudo yum -y update && sudo yum -y install httpd && sudo systemctl start httpd
</pre>

This will take a little while to run through. Once complete, the Apache start page should be available via the Load-Balancers' public IP, load-balanced between the 2 VMs.

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


