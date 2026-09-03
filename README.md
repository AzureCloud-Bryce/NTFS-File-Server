# Lab 1: NTFS File Server Lab

Terraform-provisioned Active Directory domain with a file server and a domain-joined client, used to practice NTFS/share permission layering, AD object management, and Infrastructure-as-Code fundamentals on Azure.

## What this builds

- One resource group (`RG-FileServerLab`) containing a VNet, subnet, NSG, three VMs, and their networking.
- **DC01**: Windows Server 2022 Azure Edition, promoted to the domain controller for `lab.local` (NetBIOS `LAB`). Static private IP `10.0.1.4` so it always resolves as the DNS server for the other two VMs.
- **FS01**: Windows Server 2022 Azure Edition, domain-joined, hosts four SMB shares (`Finance`, `HR`, `Sales`, `IT`) with NTFS permissions layered by AD security group.
- **CLIENT01**: Windows 11 Pro, domain-joined, used to RDP in as a test user and confirm the share permissions behave as designed.
- A separate resource group (`RG-TerraformState`) holding the storage account for Terraform remote state. It's kept out of `RG-FileServerLab` on purpose: state has to survive a `terraform destroy` of the lab itself, or every teardown/rebuild cycle starts from a blank state file.
- An Azure Key Vault (RBAC-authorized) that stores the VM admin password as a secret, so it never sits in a `.tfvars` file or shell history.

Only one inbound rule exists on the NSG: TCP 3389 (RDP) from your own IP. Everything else inbound is denied by Azure's default NSG rules.

## Repository structure

```
ntfs-lab-terraform/ 
├── README.md                     # This file: overview and architecture
├── SOP.md                        # Full copy-paste build guide: every file, every command
├── backend.tf                    # Remote state config (azurerm backend)
├── versions.tf                   # Provider version constraints
├── variables.tf                  # Input variable declarations
├── terraform.tfvars.example      # Template for your own tfvars
├── terraform.tfvars              # Your actual values (gitignored, do not commit)
├── main.tf                       # RG, VNet, subnet, NSG, NICs, public IPs, VMs
├── keyvault.tf                   # Key Vault + RBAC role assignment + secret
├── outputs.tf                    # Public IPs, DC01 private IP, Key Vault name
├── configure-lab.ps1             # Orchestrates all post-deployment scripts
└── scripts/
    ├── 00-promote-dc.ps1
    ├── 01-create-ad-users-groups.ps1
    ├── 02-configure-shares-and-permissions.ps1
    ├── 03-configure-rdp-gpo.ps1
    ├── 04-domain-join.ps1
    ├── 05-verify-ad.ps1
    ├── 05-verify-shares.ps1
    └── 06-add-rdp-users.ps1
```

## Architecture notes

**NIC creation order.** DC01's NIC is the only one with a static private IP (`10.0.1.4`, the address the other two VMs point their DNS at). FS01 and CLIENT01 get dynamic private IPs, and both of their NIC resources carry an explicit `depends_on` back through DC01's NIC. Without that ordering, Terraform is free to create all three NICs in parallel, and Azure's dynamic allocator can race DC01's static claim on `10.0.1.4`, failing the deployment with `PrivateIPAddressIsAllocated`. `fs01` does not need to depend on `client01`: two dynamic allocations don't compete for a specific address the way a static claim does.

**`time_sleep` resources.** Azure's control plane needs a short window to fully propagate the VNet and NSG before dependent resources are created against them. Two 45-second `time_sleep` resources absorb that lag rather than relying on retries.

**Key Vault RBAC.** `enable_rbac_authorization = true`, with the deploying identity granted `Key Vault Secrets Officer` and an explicit `depends_on` from the secret to that role assignment, so Terraform doesn't try to write the secret before the role has propagated.

**Remote state.** `backend.tf` authenticates to the state storage account with your Azure AD identity (`use_azuread_auth = true`), not a shared access key. Being Owner on the subscription does not automatically grant data-plane access to the storage account; see `SOP.md` if `terraform init` fails with `AuthorizationPermissionMismatch`.

## Quick start

**[SOP.md](./SOP.md)** has the full walkthrough: every file's contents, every command, and the issues you're likely to hit. It's written to be followed from scratch, without needing anything from this repo directly.

```powershell
terraform init
terraform plan
terraform apply

terraform output key_vault_name
.\configure-lab.ps1 -KeyVaultName "kv-fslab-XXXXXXXX"
```

## Related

Lab 2 builds on this same pattern to practice Azure RBAC design, and shares the same Terraform state storage account and container (different state key, so the two don't collide).
