VNet & Storage Config

This project demonstrates hands-on Azure networking and storage security configuration: extending an existing virtual network with a dedicated subnet, deploying a storage account with public access fully disabled, and connecting it privately via a private endpoint with full DNS integration. The environment is defined as reusable Infrastructure-as-Code using Bicep, and builds directly on the VM/RBAC environment from Project 1.


Architecture

Virtual Network (extended from Project 1): vnet-vmrbac-project, now with a second subnet — subnet-private-endpoints (172.16.1.0/24) — dedicated to private endpoint traffic, kept separate from the VM's resource subnet per Azure's networking guidance for private endpoints
Network Security Group (extended from Project 1): nsg-vmrbac-project now includes explicit inbound and outbound Deny rules restricting the VM's subnet to VNet-only traffic — no unsolicited inbound from within the VNet, no outbound internet access at all
Storage Account: stvmrbacproject01, created with public network access fully disabled
Private Endpoint: pe-storage-vmrbac, connecting the storage account's blob service into subnet-private-endpoints
Private DNS Zone: privatelink.blob.core.windows.net, linked to the VNet and connected to the private endpoint via a DNS zone group — this is what makes the storage account's normal hostname resolve to its private IP (172.16.1.4) from inside the VNet, instead of a public address
Infrastructure as Code: the full second-subnet, storage account, NSG rules, private endpoint, and DNS chain are defined in storage-network.bicep, which references the existing VNet from Project 1 rather than redeclaring it


<img width="612" height="432" alt="Screenshot 2026-09-17 at 12 26 17 PM" src="https://github.com/user-attachments/assets/65fd7cdd-25fb-4f60-a733-8461372f9887" />

- The VM sits in its own subnet with internet blocked, the private endpoint sits in its dedicated subnet with a real private IP, the solid arrow shows the actual private-link data path to the storage account, and the dashed arrow shows the DNS zone resolving the storage account's hostname to that private IP — the logical resolution path rather than a data connection.



Key decisions

Extended the existing VNet rather than building a new one. Since Project 2's storage/networking work is conceptually part of the same environment as Project 1's VM, I extended vnet-vmrbac-project with a second subnet rather than standing up an isolated VNet. The Bicep template reflects this by referencing the existing VNet with Bicep's existing keyword instead of redeclaring it — a more realistic pattern than most single-project tutorials show, since real environments build incrementally on shared infrastructure rather than starting fresh each time.

Chose the full VNet-isolation NSG posture over allowing outbound internet. Project 2's spec asked for explicit inbound and outbound restrictions. I chose the stricter interpretation — deny all outbound internet, allow only VNet-internal traffic — over a more permissive option that would have also allowed OS updates. This is a deliberate tradeoff: stronger isolation, at the cost of the VM being unable to reach the public internet for anything, including its own package manager.

Portal vs. CLI produced different results for the same task, twice. Creating the private endpoint through the Portal was blocked by the tag-enforcement Policy — the wizard couldn't apply a tag to an auto-generated child NIC resource it creates behind the scenes. The identical operation via Azure CLI succeeded with the same tag. Separately, the CLI's az network private-endpoint create command does not automatically configure DNS integration the way the Portal wizard does — this had to be built manually as three additional resources (a Private DNS Zone, a VNet link, and a DNS zone group) before the storage account's hostname would actually resolve to its private IP from inside the VNet. Both gaps are undocumented behavior differences between the two tools, discovered through direct troubleshooting rather than found in Microsoft's documentation.



Challenges & troubleshooting

resourceGroup().location returned a stale value. The Bicep function resourceGroup().location returns the resource group's own recorded location metadata — not necessarily where the actual resources inside it live. rg-vmrbac-project still carried canadaeast as its metadata location from its original creation (before the VM was rebuilt in North Central US after multiple capacity failures), even though every real resource in it lives in northcentralus. Using this function for the private endpoint's location caused a deployment failure trying to create a resource with a duplicate name in the wrong region. Fixed by using an explicit location parameter instead of relying on the function.

az deployment group create --what-if caught an unintended property change before it happened. Running --what-if against the real environment revealed the template would have flipped privateEndpointNetworkPolicies from Disabled back to Enabled on the private-endpoint subnet — a setting that was deliberately chosen earlier and not something I wanted overwritten. This was resolved by explicitly declaring the property in the template rather than letting it default. This is the clearest example in either project of why --what-if matters before running a real deployment against a live environment.


Verification
Public access disabled: the storage account rejects direct public network requests; it's only reachable through the private endpoint from inside the VNet
Private DNS resolution: from inside the VM, nslookup stvmrbacproject01.blob.core.windows.net resolves to 172.16.1.4 — the private endpoint's IP — rather than a public address
Outbound internet blocked: from inside the VM, curl https://www.google.com times out, confirming the NSG's outbound Deny rule is active
Inbound restricted to SSH from a known IP: unchanged from Project 1, still the only way into the VM
Bicep template accuracy: az deployment group create --what-if against the live environment showed the template matches reality, with only a minor tag standardization as an actual change — confirmed by a real deployment completing with provisioningState: Succeeded


<img width="863" height="613" alt="Private DNS res" src="https://github.com/user-attachments/assets/b2ef8a6d-2fec-4b92-9e48-25a18cc0e1ed" />
Private DNS Resolution 


<img width="558" height="71" alt="outbound internet " src="https://github.com/user-attachments/assets/bb03a3cf-7475-4885-8479-79576625c811" />

Outbound internet Blocked 


