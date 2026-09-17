VNet & Storage Config

This project demonstrates hands-on Azure networking and storage security configuration: extending an existing virtual network with a dedicated subnet, deploying a storage account with public access fully disabled, and connecting it privately via a private endpoint with full DNS integration. The environment is defined as reusable Infrastructure-as-Code using Bicep, and builds directly on the VM/RBAC environment from Project 1.


Architecture
Virtual Network (extended from Project 1): vnet-vmrbac-project, now with a second subnet — subnet-private-endpoints (172.16.1.0/24) — dedicated to private endpoint traffic, kept separate from the VM's resource subnet per Azure's networking guidance for private endpoints
Network Security Group (extended from Project 1): nsg-vmrbac-project now includes explicit inbound and outbound Deny rules restricting the VM's subnet to VNet-only traffic — no unsolicited inbound from within the VNet, no outbound internet access at all
Storage Account: stvmrbacproject01, created with public network access fully disabled
Private Endpoint: pe-storage-vmrbac, connecting the storage account's blob service into subnet-private-endpoints
Private DNS Zone: privatelink.blob.core.windows.net, linked to the VNet and connected to the private endpoint via a DNS zone group — this is what makes the storage account's normal hostname resolve to its private IP (172.16.1.4) from inside the VNet, instead of a public address
Infrastructure as Code: the full second-subnet, storage account, NSG rules, private endpoint, and DNS chain are defined in storage-network.bicep, which references the existing VNet from Project 1 rather than redeclaring it
