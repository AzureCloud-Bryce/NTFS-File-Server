# SOP: Lab 1 NTFS File Server Lab (Build Guide)

A self-contained, copy-paste guide to building this lab from scratch. Every file's full contents are below; create each one in a new project folder as you go. You don't need anything from a repo to follow this; it's written to stand on its own.

All commands are PowerShell. Everything was built and run from a PowerShell terminal (VS Code), so that's what's documented here.

## Placeholders used in this guide

A few values are specific to whoever is running the lab and can't be copy-pasted as-is. Replace these everywhere they appear before running anything:

| Placeholder | What it is | Example |
|---|---|---|
| `REGION` | The Azure region you're deploying to | `West US 2` |
| `STATE_STORAGE_ACCOUNT` | A name you choose for the Terraform state storage account. Must be globally unique across all of Azure, lowercase letters/numbers only, 3–24 characters | `jsmithtfstatelab` |
| `YOUR_PUBLIC_IP` | Your own public IP address, found with `Invoke-RestMethod ifconfig.me/ip` | `203.0.113.45` |
| `YourStrongPassword!` | A password you choose for the VM admin account. Minimum 12 characters, upper + lower + number + symbol | N/A |

Everything else (resource group names, VM names, the domain name `lab.local`, test usernames) are just this lab's example names. You can rename them if you want, as long as you keep the same name consistent everywhere it's referenced.

## Prerequisites

- Terraform, installed and on your PATH. Check your version with `terraform version`; this project's floor is `>= 1.5.0`, but there's no reason not to use whatever's current.
- Azure CLI, logged in (`az login`) with an account that has Owner (or Contributor + User Access Administrator) on the subscription.
- Your public IP in CIDR form, e.g. `203.0.113.45/32`:

```powershell
Invoke-RestMethod ifconfig.me/ip
```

## Step 1: Create remote state storage (one-time)

### What this is for

Every time Terraform runs, it needs to know what it already created, so it can compare that against your `.tf` files and figure out what to add, change, or remove. That record is the **state file**: a JSON document mapping each resource in your config to the real Azure resource ID it corresponds to. Without it, Terraform has no memory between runs and can't safely manage anything that already exists.

State can live as a plain local file (`terraform.tfstate` in the project folder), but that's fragile for anything beyond a single throwaway run: it's easy to delete by accident, it isn't shared if you ever work from a second machine, and it can contain sensitive values in plain text, which isn't something you want sitting in a folder that gets pushed to GitHub. Instead, this project uses a **remote backend**: an Azure Storage blob holds the state file, and Terraform reads and writes it over the network every time you run `plan` or `apply`.

### Why a separate resource group

The state storage account lives in its own resource group (`RG-TerraformState`), not the one the lab itself runs in (`RG-FileServerLab`). Terraform can only track resources through its state file, so the state file can't be one of the resources it's tracking. If the storage account holding the state lived inside `RG-FileServerLab`, running `terraform destroy` on the lab would delete the very state file Terraform needs to safely perform that destroy. Keeping the two resource groups separate means you can `terraform destroy` and rebuild `RG-FileServerLab` as many times as you want, and the state survives every cycle untouched.

This is also why this step is run with plain Azure CLI commands instead of Terraform: you're bootstrapping the thing Terraform depends on before Terraform can run at all.

### Commands

```powershell
az group create --name RG-TerraformState --location "REGION"

az storage account create `
  --name STATE_STORAGE_ACCOUNT `
  --resource-group RG-TerraformState `
  --location "REGION" `
  --sku Standard_LRS

az storage container create `
  --account-name STATE_STORAGE_ACCOUNT `
  --name tfstate `
  --auth-mode login
```

What each command does:

1. **`az group create`**: creates the resource group that will hold only the state storage account. Nothing else should ever go in here.
2. **`az storage account create`**: creates the actual storage account. `--sku Standard_LRS` (locally redundant storage) is the cheapest replication tier Azure offers; a lab's state file doesn't need geo-redundancy.
3. **`az storage container create`**: creates a container inside the storage account. A storage account doesn't hold files directly; it holds one or more containers (think of a container as a top-level folder), and files (blobs) live inside those. This lab's state file will be uploaded as a single blob inside the `tfstate` container. `--auth-mode login` runs this command using your own `az login` identity rather than a storage account access key, consistent with how `backend.tf` authenticates later.

**If `terraform init` later fails with `AuthorizationPermissionMismatch`:** being Owner on the subscription is a management-plane permission and does not grant data-plane access to blobs inside the storage account. Grant yourself the data-plane role explicitly, then wait a minute or two for it to propagate before retrying:

```powershell
$me = az ad signed-in-user show --query id -o tsv
$storageId = az storage account show --name STATE_STORAGE_ACCOUNT --resource-group RG-TerraformState --query id -o tsv

az role assignment create `
  --assignee $me `
  --role "Storage Blob Data Contributor" `
  --scope $storageId
```

## Step 2: Create the project files

Create a new folder for the project and add each of the following files.

### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
    time = {
      source  = "hashicorp/time"
      version = "~> 0.11"
    }
  }
}

provider "azurerm" {
  features {}
}
```

`required_version` is a floor, not a pin: run `terraform version` and use whatever you actually have installed, as long as it's 1.5.0 or newer. `~> 3.0` on azurerm pins the major version to 3.x (never jumps to 4.x on you) but still floats across every 3.x minor release; that's a deliberate, common way to write it.

### `backend.tf`

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "RG-TerraformState"
    storage_account_name = "STATE_STORAGE_ACCOUNT"
    container_name        = "tfstate"
    key                    = "ntfs-lab.terraform.tfstate"

    # Authenticate to the state blob with your az login identity instead of
    # a shared access key. Works when the storage account has shared key
    # access disabled, which is increasingly the default.
    use_azuread_auth = true
  }
}
```

### `variables.tf`

```hcl
variable "location" {
  type        = string
  default     = "REGION"
  description = "Azure region."
}

variable "resource_group_name" {
  type        = string
  default     = "RG-FileServerLab"
  description = "Must stay consistent across runs; later steps reference this name directly."
}

variable "vnet_name" {
  type    = string
  default = "VNET-FileServerLab"
}

variable "subnet_name" {
  type    = string
  default = "Subnet-Servers"
}

variable "vnet_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subnet_cidr" {
  type    = string
  default = "10.0.1.0/24"
}

variable "nsg_name" {
  type    = string
  default = "NSG-RDP"
}

# No default on purpose. A default of "*" would publish RDP to the entire
# internet if you forget to set it. With no default, Terraform prompts for it.
variable "rdp_source" {
  type        = string
  description = "Your public IP in CIDR format, e.g. 1.2.3.4/32. Get it with: Invoke-RestMethod ifconfig.me/ip"

  validation {
    condition     = can(cidrhost(var.rdp_source, 0))
    error_message = "rdp_source must be valid CIDR, e.g. 1.2.3.4/32."
  }
}

variable "admin_username" {
  type    = string
  default = "azureadmin"
}

variable "admin_password" {
  type        = string
  sensitive   = true
  description = "Set as the TF_VAR_admin_password environment variable; never in a file."
}

variable "server_vm_size" {
  type    = string
  default = "Standard_B2as_v2"
}

variable "client_vm_size" {
  type    = string
  default = "Standard_B2as_v2"
}
```

`rdp_source` has no default on purpose: a default of `"*"` would open RDP to the entire internet if you forget to set it, so Terraform prompts for it instead. `server_vm_size` / `client_vm_size` default to `Standard_B2as_v2` here, but see the note on SKU availability in Step 3. This default may not be available in your region or subscription, and this lab actually ran on `Standard_D2s_v3` instead.

### `main.tf`

```hcl
# Resource Group
resource "azurerm_resource_group" "rg" {
  name     = var.resource_group_name
  location = var.location
}

# Virtual Network: 10.0.0.0/16 provides 65,536 addresses
resource "azurerm_virtual_network" "vnet" {
  name                = var.vnet_name
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = [var.vnet_cidr]
}

resource "time_sleep" "wait_after_vnet" {
  create_duration = "45s"
  depends_on      = [azurerm_virtual_network.vnet]
}

# Subnet: 10.0.1.0/24 provides 251 usable addresses
resource "azurerm_subnet" "subnet" {
  name                 = var.subnet_name
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = [var.subnet_cidr]
  depends_on           = [time_sleep.wait_after_vnet]
}

# NSG: only your IP can reach port 3389. All other inbound is denied by default.
resource "azurerm_network_security_group" "nsg" {
  name                = var.nsg_name
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "Allow-RDP-3389"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = var.rdp_source
    source_port_range          = "*"
    destination_port_range     = "3389"
    destination_address_prefix = "*"
  }

  depends_on = [time_sleep.wait_after_vnet]
}

resource "time_sleep" "wait_after_nsg" {
  create_duration = "45s"
  depends_on      = [azurerm_network_security_group.nsg]
}

# Public IPs: Standard SKU required for static allocation
resource "azurerm_public_ip" "dc01" {
  name                = "dc01-pip"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
  depends_on          = [time_sleep.wait_after_nsg]
}

resource "azurerm_public_ip" "fs01" {
  name                = "fs01-pip"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
  depends_on          = [time_sleep.wait_after_nsg]
}

resource "azurerm_public_ip" "client01" {
  name                = "client01-pip"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
  depends_on          = [time_sleep.wait_after_nsg]
}

# DC01 NIC: static IP 10.0.1.4 so FS01/CLIENT01 DNS never breaks after restarts
resource "azurerm_network_interface" "dc01" {
  name                = "dc01-nic"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.0.1.4"
    public_ip_address_id          = azurerm_public_ip.dc01.id
  }

  depends_on = [time_sleep.wait_after_nsg]
}

# FS01 and CLIENT01 depend on DC01's NIC existing first; see the note below.
resource "azurerm_network_interface" "fs01" {
  name                = "fs01-nic"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.fs01.id
  }

  depends_on = [time_sleep.wait_after_nsg, azurerm_network_interface.dc01]
}

resource "azurerm_network_interface" "client01" {
  name                = "client01-nic"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.client01.id
  }

  depends_on = [time_sleep.wait_after_nsg, azurerm_network_interface.dc01, azurerm_network_interface.fs01]
}

# Attach NSG to each NIC; without this the NSG exists but applies to nothing
resource "azurerm_network_interface_security_group_association" "dc01" {
  network_interface_id       = azurerm_network_interface.dc01.id
  network_security_group_id  = azurerm_network_security_group.nsg.id
  depends_on                 = [time_sleep.wait_after_nsg]
}

resource "azurerm_network_interface_security_group_association" "fs01" {
  network_interface_id       = azurerm_network_interface.fs01.id
  network_security_group_id  = azurerm_network_security_group.nsg.id
  depends_on                 = [time_sleep.wait_after_nsg]
}

resource "azurerm_network_interface_security_group_association" "client01" {
  network_interface_id       = azurerm_network_interface.client01.id
  network_security_group_id  = azurerm_network_security_group.nsg.id
  depends_on                 = [time_sleep.wait_after_nsg]
}

# DC01: Windows Server 2022 (Azure Edition includes the VM agent pre-installed)
resource "azurerm_windows_virtual_machine" "dc01" {
  name                = "DC01"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  size                = var.server_vm_size
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  network_interface_ids = [azurerm_network_interface.dc01.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-datacenter-azure-edition"
    version   = "latest"
  }

  depends_on = [time_sleep.wait_after_nsg]
}

# FS01: Windows Server 2022
resource "azurerm_windows_virtual_machine" "fs01" {
  name                = "FS01"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  size                = var.server_vm_size
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  network_interface_ids = [azurerm_network_interface.fs01.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-datacenter-azure-edition"
    version   = "latest"
  }

  depends_on = [time_sleep.wait_after_nsg]
}

# CLIENT01: Windows 11 Pro (ships with RDP disabled by default; fixed by the extension below)
resource "azurerm_windows_virtual_machine" "client01" {
  name                = "CLIENT01"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  size                = var.client_vm_size
  admin_username      = var.admin_username
  admin_password      = var.admin_password
  network_interface_ids = [azurerm_network_interface.client01.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsDesktop"
    offer     = "windows-11"
    sku       = "win11-25h2-pro"
    version   = "latest"
  }

  depends_on = [time_sleep.wait_after_nsg]
}

# Enable RDP on CLIENT01: Windows 11 has it disabled by default.
# Sets fDenyTSConnections=0 and opens the firewall rule.
# Without this, RDP attempts to CLIENT01 time out with no error.
resource "azurerm_virtual_machine_extension" "client01_enable_rdp" {
  name                 = "enable-rdp"
  virtual_machine_id   = azurerm_windows_virtual_machine.client01.id
  publisher            = "Microsoft.Compute"
  type                 = "CustomScriptExtension"
  type_handler_version = "1.10"

  settings = jsonencode({
    commandToExecute = "powershell -Command \"Set-ItemProperty -Path 'HKLM:\\System\\CurrentControlSet\\Control\\Terminal Server' -Name 'fDenyTSConnections' -Value 0; Enable-NetFirewallRule -DisplayGroup 'Remote Desktop'\""
  })

  depends_on = [azurerm_windows_virtual_machine.client01]
}
```

**Known issue: NIC IP allocation race.** DC01's NIC is the only one with a static private IP (`10.0.1.4`). If `fs01`'s and `client01`'s NIC resources don't carry the `depends_on` chain shown above, Terraform is free to create all three NICs in parallel, and Azure's dynamic allocator can occasionally land on `10.0.1.4` before DC01's static claim goes through, failing the deployment with `PrivateIPAddressIsAllocated`. Forcing DC01's NIC to exist first removes the race. `fs01` doesn't need to depend on `client01` (or vice versa): two dynamic allocations don't compete for a specific address the way a static claim does, so ordering between them only costs deploy time for no benefit.

### `keyvault.tf`

```hcl
data "azurerm_client_config" "current" {}

# random_id generates an 8-character hex suffix; Key Vault names must be globally unique.
resource "random_id" "kv_suffix" {
  byte_length = 4
}

resource "azurerm_key_vault" "lab_kv" {
  name                        = "kv-fslab-${random_id.kv_suffix.hex}"
  location                    = azurerm_resource_group.rg.location
  resource_group_name         = azurerm_resource_group.rg.name
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  sku_name                    = "standard"
  enable_rbac_authorization   = true # required; without this, every secret operation 403s
  soft_delete_retention_days  = 7
  purge_protection_enabled    = false

  tags = {
    Environment = "Lab"
    ManagedBy   = "Terraform"
  }
}

# Grants whoever ran az login permission to write secrets.
# current.object_id resolves dynamically to the logged-in identity.
resource "azurerm_role_assignment" "kv_deployer_access" {
  scope                = azurerm_key_vault.lab_kv.id
  role_definition_name = "Key Vault Secrets Officer"
  principal_id         = data.azurerm_client_config.current.object_id
}

# depends_on ensures the role assignment propagates before Terraform tries to
# write the secret; without it you get a 403.
resource "azurerm_key_vault_secret" "admin_password" {
  name         = "vm-admin-password"
  value        = var.admin_password
  key_vault_id = azurerm_key_vault.lab_kv.id
  depends_on   = [azurerm_role_assignment.kv_deployer_access]

  tags = {
    ManagedBy  = "Terraform"
  }
}
```

### `outputs.tf`

```hcl
output "dc01_public_ip" {
  value       = azurerm_public_ip.dc01.ip_address
  description = "DC01 public IP."
}

output "dc01_private_ip" {
  value       = azurerm_network_interface.dc01.private_ip_address
  description = "Always 10.0.1.4, DC01's static DNS address."
}

output "fs01_public_ip" {
  value       = azurerm_public_ip.fs01.ip_address
  description = "FS01 public IP."
}

output "client01_public_ip" {
  value       = azurerm_public_ip.client01.ip_address
  description = "RDP here as a test user to verify the lab."
}

output "key_vault_name" {
  value       = azurerm_key_vault.lab_kv.name
  description = "Pass this to configure-lab.ps1 with -KeyVaultName."
}
```

### `terraform.tfvars`

Not committed to git; create it locally with your own values:

```hcl
location       = "REGION"
rdp_source     = "YOUR_PUBLIC_IP/32"
server_vm_size = "Standard_D2s_v3"
client_vm_size = "Standard_D2s_v3"
```

`server_vm_size` / `client_vm_size` default to `Standard_B2as_v2` in `variables.tf`, but SKU availability varies by region and subscription. This deployment used `Standard_D2s_v3` after the smaller B-series and D1 SKUs weren't available. Check what's actually available in your region before assuming a default will deploy cleanly:

```powershell
az vm list-skus --location "REGION" --size "Standard_D" --output table
```

Set the admin password as an environment variable; never in a file:

```powershell
$env:TF_VAR_admin_password = "YourStrongPassword!"
```

Minimum 12 characters, upper + lower + number + symbol.

## Step 3: Deploy infrastructure

```powershell
terraform init
terraform plan
terraform apply
```

## Step 4: Post-deployment configuration

Terraform provisions the VMs; it doesn't configure AD, shares, or GPOs. `configure-lab.ps1` retrieves the admin password from Key Vault and runs each script below against the right VM via `az vm run-command invoke`, a control-plane channel that reaches the VM regardless of NSG rules, since it doesn't go over the network path RDP does.

### `configure-lab.ps1`

```powershell
param([Parameter(Mandatory=$true)][string]$KeyVaultName,[string]$ResourceGroup="RG-FileServerLab")
$startTime=Get-Date
Write-Host "`n[$(Get-Date -Format "HH:mm:ss")] Retrieving credentials from Key Vault..." -ForegroundColor Cyan
$AdminPassword=az keyvault secret show --vault-name $KeyVaultName --name "vm-admin-password" --query "value" -o tsv
if (-not $AdminPassword -or $LASTEXITCODE -ne 0) { throw "Could not retrieve password from Key Vault. Run az login first." }
Write-Host "  Credentials retrieved." -ForegroundColor Green

function Invoke-VMScript { param([string]$VMName,[string]$ScriptPath,[string]$Description,[hashtable]$Replacements=@{})
    Write-Host "`n[$(Get-Date -Format "HH:mm:ss")] >>> $Description" -ForegroundColor Cyan
    $script=Get-Content $ScriptPath -Raw
    foreach ($key in $Replacements.Keys) { $script=$script -replace $key,[regex]::Escape($Replacements[$key]) }
    $tempFile=[System.IO.Path]::GetTempPath()+[System.IO.Path]::GetRandomFileName()+".ps1"
    $script | Out-File -FilePath $tempFile -Encoding UTF8
    try {
        $jsonLines=az vm run-command invoke --resource-group $ResourceGroup --name $VMName --command-id RunPowerShellScript --scripts "@$tempFile" --output json --only-show-errors
        if ($LASTEXITCODE -ne 0) { throw "az vm run-command failed on $VMName" }
        $result=($jsonLines -join "`n") | ConvertFrom-Json
        $stdout=($result.value | Where-Object{$_.code -like "*StdOut*"}).message
        $stderr=($result.value | Where-Object{$_.code -like "*StdErr*"}).message
        if ($stdout) { Write-Host $stdout }
        if ($stderr -and $stderr.Trim() -ne "") { Write-Warning "  VM StdErr: $stderr" }
    } finally { Remove-Item $tempFile -ErrorAction SilentlyContinue }
}

function Wait-VMOnline { param([string]$VMName,[int]$TimeoutSeconds=360)
    Write-Host "  Waiting for $VMName..." -ForegroundColor Yellow
    $elapsed=0
    do { Start-Sleep -Seconds 15; $elapsed+=15
        try { $state=(az vm show -g $ResourceGroup -n $VMName -d --query "powerState" -o tsv --only-show-errors 2>$null).Trim() } catch { $state="" }
        Write-Host "    $VMName -> $state ($elapsed s)" -ForegroundColor DarkGray
    } while ($state -ne "VM running" -and $elapsed -lt $TimeoutSeconds)
    if ($state -ne "VM running") { throw "Timeout: $VMName did not return within ${TimeoutSeconds}s" }
    Write-Host "  $VMName is online." -ForegroundColor Green
}

Write-Host "`n[STAGE 1] Promoting DC01 to Domain Controller" -ForegroundColor Magenta
try { Invoke-VMScript -VMName "DC01" -ScriptPath ".\scripts\00-promote-dc.ps1" -Description "Promoting DC01" -Replacements @{"SAFE_MODE_PASSWORD"=$AdminPassword} }
catch { Write-Host "  DC01 disconnected -- expected after promotion." -ForegroundColor Yellow }
Start-Sleep -Seconds 60; Wait-VMOnline -VMName "DC01"; Start-Sleep -Seconds 90

Write-Host "`n[STAGE 2] Creating OUs, Groups, and Users" -ForegroundColor Magenta
Invoke-VMScript -VMName "DC01" -ScriptPath ".\scripts\01-create-ad-users-groups.ps1" -Description "Creating AD objects"

Write-Host "`n[STAGE 3] Joining FS01 to lab.local" -ForegroundColor Magenta
Invoke-VMScript -VMName "FS01" -ScriptPath ".\scripts\04-domain-join.ps1" -Description "Joining FS01" -Replacements @{"ADMIN_PASSWORD"=$AdminPassword}
Start-Sleep -Seconds 30; Wait-VMOnline -VMName "FS01"; Start-Sleep -Seconds 30

Write-Host "`n[STAGE 4] Configuring shares and NTFS on FS01" -ForegroundColor Magenta
Invoke-VMScript -VMName "FS01" -ScriptPath ".\scripts\02-configure-shares-and-permissions.ps1" -Description "Creating shares and NTFS"

Write-Host "`n[STAGE 5] Joining CLIENT01 to lab.local" -ForegroundColor Magenta
Invoke-VMScript -VMName "CLIENT01" -ScriptPath ".\scripts\04-domain-join.ps1" -Description "Joining CLIENT01" -Replacements @{"ADMIN_PASSWORD"=$AdminPassword}
Start-Sleep -Seconds 30; Wait-VMOnline -VMName "CLIENT01"; Start-Sleep -Seconds 30

Write-Host "`n[STAGE 5b] Granting Domain Users RDP on CLIENT01" -ForegroundColor Magenta
Invoke-VMScript -VMName "CLIENT01" -ScriptPath ".\scripts\06-add-rdp-users.ps1" -Description "Adding Domain Users to RDP group"

Write-Host "`n[STAGE 6] Configuring RDP GPO on DC01" -ForegroundColor Magenta
Invoke-VMScript -VMName "DC01" -ScriptPath ".\scripts\03-configure-rdp-gpo.ps1" -Description "Creating RDP GPO"

Write-Host "`n[STAGE 7] Running automated verification" -ForegroundColor Magenta
Invoke-VMScript -VMName "DC01" -ScriptPath ".\scripts\05-verify-ad.ps1" -Description "Verifying AD"
Invoke-VMScript -VMName "FS01" -ScriptPath ".\scripts\05-verify-shares.ps1" -Description "Verifying shares"

$duration=(Get-Date)-$startTime
Write-Host "`n=== LAB FULLY CONFIGURED ($([math]::Round($duration.TotalMinutes,1)) min) ===" -ForegroundColor Green
Write-Host "RDP into CLIENT01 as: LAB\sarah.jones  Password: P@ssw0rd123!"
```

Create a `scripts\` folder next to it and add the following files:

### `scripts\00-promote-dc.ps1`

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
Write-Host "Installing AD DS..." -ForegroundColor Yellow
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools -Verbose:$false
$safeModePassword = ConvertTo-SecureString "SAFE_MODE_PASSWORD" -AsPlainText -Force
Import-Module ADDSDeployment
Install-ADDSForest `
    -DomainName                    "lab.local" `
    -DomainNetbiosName             "LAB" `
    -ForestMode                    "WinThreshold" `
    -DomainMode                    "WinThreshold" `
    -InstallDns:                   $true `
    -SafeModeAdministratorPassword $safeModePassword `
    -Force:                        $true `
    -NoRebootOnCompletion:         $false
# DC01 reboots here. configure-lab.ps1 waits for it to come back online.
```

### `scripts\01-create-ad-users-groups.ps1`

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
$domain="lab.local"; $domainDN="DC=lab,DC=local"
$password=ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force
foreach ($ou in @("Lab Users","Lab Computers","Lab Groups")) {
    New-ADOrganizationalUnit -Name $ou -Path $domainDN -ProtectedFromAccidentalDeletion $false
    Write-Host "Created OU: $ou" -ForegroundColor Yellow
}
foreach ($group in @("GRP_Finance","GRP_HR","GRP_Sales","GRP_IT")) {
    New-ADGroup -Name $group -GroupScope Global -GroupCategory Security -Path "OU=Lab Groups,$domainDN"
    Write-Host "Created group: $group" -ForegroundColor Yellow
}
$users=@(
    @{First="John"; Last="Smith"; Username="john.smith";  Group="GRP_IT"     },
    @{First="Sarah";Last="Jones"; Username="sarah.jones"; Group="GRP_Finance" },
    @{First="Mike"; Last="Brown"; Username="mike.brown";  Group="GRP_Finance" },
    @{First="Lisa"; Last="White"; Username="lisa.white";  Group="GRP_HR"      },
    @{First="Tom";  Last="Davis"; Username="tom.davis";   Group="GRP_Sales"   })
foreach ($user in $users) {
    New-ADUser -GivenName $user.First -Surname $user.Last -Name "$($user.First) $($user.Last)" `
               -SamAccountName $user.Username -UserPrincipalName "$($user.Username)@$domain" `
               -Path "OU=Lab Users,$domainDN" -AccountPassword $password -Enabled $true -PasswordNeverExpires $true
    Add-ADGroupMember -Identity $user.Group -Members $user.Username
    Write-Host "Created: $($user.Username) -> $($user.Group)" -ForegroundColor Cyan
}
Write-Host "`nDone." -ForegroundColor Green
```

Test users are created with the password `P@ssw0rd123!`, hardcoded here. That's fine for a disposable lab domain, not something to reuse anywhere real.

### `scripts\04-domain-join.ps1`

Used for both FS01 and CLIENT01.

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
$adapter=Get-NetAdapter | Where-Object{$_.Status -eq "Up"} | Select-Object -First 1
Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses "10.0.1.4"
$retries=0; $resolved=$false
do { Start-Sleep -Seconds 15; $retries++
    $resolved=[bool](Resolve-DnsName "lab.local" -ErrorAction SilentlyContinue)
    Write-Host "  Attempt $retries -- resolved: $resolved"
} while (-not $resolved -and $retries -lt 12)
if (-not $resolved) { throw "lab.local did not resolve after 3 minutes." }
$domainCred=New-Object PSCredential("LAB\azureadmin",
    (ConvertTo-SecureString "ADMIN_PASSWORD" -AsPlainText -Force))
Add-Computer -DomainName "lab.local" -Credential $domainCred -Restart -Force
# VM restarts here. configure-lab.ps1 waits for it to come back.
```

The VNet itself has no DNS servers configured, so each VM needs to be pointed at DC01 (`10.0.1.4`) explicitly before it can resolve `lab.local` and join the domain. That's what the `Set-DnsClientServerAddress` / `Resolve-DnsName` retry loop is doing.

### `scripts\02-configure-shares-and-permissions.ps1`

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
$domain="LAB"; $basePath="C:\Shares"
New-Item -Path $basePath -ItemType Directory -Force
foreach ($folder in @("Finance","HR","Sales","IT")) {
    New-Item -Path "$basePath\$folder" -ItemType Directory -Force
    New-SmbShare -Name $folder -Path "$basePath\$folder" -FullAccess "Everyone"
}
function Set-FolderPermissions { param($path,$permissions)
    icacls $path /inheritance:d
    icacls $path /remove "BUILTIN\Users"
    icacls $path /remove "Everyone"
    icacls $path /remove "NT AUTHORITY\Authenticated Users"
    foreach ($p in $permissions) { icacls $path /grant "$($p.Identity)`:$($p.Rights)" }
}
Set-FolderPermissions -path "$basePath\Finance" -permissions @(
    @{Identity="$domain\GRP_Finance";Rights="(OI)(CI)M"},
    @{Identity="$domain\GRP_HR";Rights="(OI)(CI)R"},
    @{Identity="$domain\GRP_IT";Rights="(OI)(CI)F"},
    @{Identity="BUILTIN\Administrators";Rights="(OI)(CI)F"})
Set-FolderPermissions -path "$basePath\HR" -permissions @(
    @{Identity="$domain\GRP_HR";Rights="(OI)(CI)M"},
    @{Identity="$domain\GRP_IT";Rights="(OI)(CI)F"},
    @{Identity="BUILTIN\Administrators";Rights="(OI)(CI)F"})
Set-FolderPermissions -path "$basePath\Sales" -permissions @(
    @{Identity="$domain\GRP_Sales";Rights="(OI)(CI)M"},
    @{Identity="$domain\GRP_IT";Rights="(OI)(CI)F"},
    @{Identity="BUILTIN\Administrators";Rights="(OI)(CI)F"})
Set-FolderPermissions -path "$basePath\IT" -permissions @(
    @{Identity="$domain\GRP_IT";Rights="(OI)(CI)F"},
    @{Identity="BUILTIN\Administrators";Rights="(OI)(CI)F"})
Write-Host "`nDone." -ForegroundColor Cyan
```

Share-level permissions are left wide open (`Everyone: Full Control`) on purpose: NTFS permissions are what actually restrict access here. The share ACL and the NTFS ACL are two independent layers; whichever one is more restrictive wins for a given user, so leaving the share open and doing the real control at the NTFS layer is a deliberate simplification, not an oversight.

### `scripts\06-add-rdp-users.ps1`

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
$rdpGroup="Remote Desktop Users"; $domainUsers="LAB\Domain Users"
$existing=Get-LocalGroupMember -Group $rdpGroup -ErrorAction SilentlyContinue | Where-Object{$_.Name -eq $domainUsers}
if ($existing) { Write-Host "$domainUsers already in $rdpGroup." -ForegroundColor Yellow }
else { Add-LocalGroupMember -Group $rdpGroup -Member $domainUsers
    Write-Host "Added $domainUsers to $rdpGroup." -ForegroundColor Green }
Get-LocalGroupMember -Group $rdpGroup | Select-Object Name,ObjectClass | Format-Table -AutoSize
```

### `scripts\03-configure-rdp-gpo.ps1`

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
$gpoName="Lab - Allow RDP for Domain Users"; $ouPath="OU=Lab Computers,DC=lab,DC=local"
New-GPO -Name $gpoName | Out-Null
New-GPLink -Name $gpoName -Target $ouPath
Set-GPRegistryValue -Name $gpoName -Key "HKLM\System\CurrentControlSet\Control\Terminal Server" -ValueName "fDenyTSConnections" -Type DWord -Value 0
$computer=Get-ADComputer -Filter { Name -eq "CLIENT01" } -ErrorAction SilentlyContinue
if ($computer) { $computer | Move-ADObject -TargetPath $ouPath; Write-Host "Moved CLIENT01 to Lab Computers OU." -ForegroundColor Cyan }
else { Write-Warning "CLIENT01 not yet in AD, may still be joining." }
Write-Host "`nGPO configuration complete." -ForegroundColor Green
```

### `scripts\05-verify-ad.ps1`

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
Import-Module ActiveDirectory
$pass=$true
Write-Host "`n=== Active Directory Verification ===" -ForegroundColor Cyan
foreach ($ou in @("Lab Users","Lab Groups","Lab Computers")) {
    $exists=[bool](Get-ADOrganizationalUnit -Filter "Name -eq '$ou'" -ErrorAction SilentlyContinue)
    if ($exists) { Write-Host "  [PASS] OU: $ou" -ForegroundColor Green }
    else { Write-Host "  [FAIL] OU missing: $ou" -ForegroundColor Red; $pass=$false }
}
foreach ($g in @("GRP_Finance","GRP_HR","GRP_Sales","GRP_IT")) {
    $exists=[bool](Get-ADGroup -Filter "Name -eq '$g'" -ErrorAction SilentlyContinue)
    if ($exists) { Write-Host "  [PASS] Group: $g" -ForegroundColor Green }
    else { Write-Host "  [FAIL] Group missing: $g" -ForegroundColor Red; $pass=$false }
}
$expectedUsers=@(
    @{Username="john.smith";Group="GRP_IT"},@{Username="sarah.jones";Group="GRP_Finance"},
    @{Username="mike.brown";Group="GRP_Finance"},@{Username="lisa.white";Group="GRP_HR"},
    @{Username="tom.davis";Group="GRP_Sales"})
foreach ($u in $expectedUsers) {
    $user=Get-ADUser -Filter "SamAccountName -eq '$($u.Username)'" -ErrorAction SilentlyContinue
    if (-not $user) { Write-Host "  [FAIL] User missing: $($u.Username)" -ForegroundColor Red; $pass=$false; continue }
    $members=Get-ADGroupMember -Identity $u.Group | Select-Object -ExpandProperty SamAccountName
    if ($members -contains $u.Username) { Write-Host "  [PASS] $($u.Username) -> $($u.Group)" -ForegroundColor Green }
    else { Write-Host "  [FAIL] $($u.Username) not in $($u.Group)" -ForegroundColor Red; $pass=$false }
}
Write-Host "`n=== AD Verification $(if ($pass){"PASSED"}else{"FAILED"}) ===" -ForegroundColor $(if ($pass){"Green"}else{"Red"})
```

### `scripts\05-verify-shares.ps1`

```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
$basePath="C:\Shares"; $pass=$true
Write-Host "`n=== Share and NTFS Verification ===" -ForegroundColor Cyan
$expectedACLs=@{
    "Finance"=@(@{Identity="LAB\GRP_Finance";Right=[System.Security.AccessControl.FileSystemRights]::Modify}
               @{Identity="LAB\GRP_HR";Right=[System.Security.AccessControl.FileSystemRights]::Read}
               @{Identity="LAB\GRP_IT";Right=[System.Security.AccessControl.FileSystemRights]::FullControl})
    "HR"=@(@{Identity="LAB\GRP_HR";Right=[System.Security.AccessControl.FileSystemRights]::Modify}
            @{Identity="LAB\GRP_IT";Right=[System.Security.AccessControl.FileSystemRights]::FullControl})
    "Sales"=@(@{Identity="LAB\GRP_Sales";Right=[System.Security.AccessControl.FileSystemRights]::Modify}
               @{Identity="LAB\GRP_IT";Right=[System.Security.AccessControl.FileSystemRights]::FullControl})
    "IT"=@(@{Identity="LAB\GRP_IT";Right=[System.Security.AccessControl.FileSystemRights]::FullControl})}
foreach ($share in $expectedACLs.Keys) {
    Write-Host "`n[ $share ]" -ForegroundColor White
    $smb=Get-SmbShare -Name $share -ErrorAction SilentlyContinue
    if ($smb) { Write-Host "  [PASS] SMB share exists" -ForegroundColor Green }
    else { Write-Host "  [FAIL] Share missing" -ForegroundColor Red; $pass=$false; continue }
    $acl=(Get-Acl "$basePath\$share").Access
    foreach ($expected in $expectedACLs[$share]) {
        $ace=$acl|Where-Object{$_.IdentityReference.Value -eq $expected.Identity -and $_.AccessControlType -eq "Allow"}
        if (-not $ace) { Write-Host "  [FAIL] $($expected.Identity) has no entry" -ForegroundColor Red; $pass=$false; continue }
        $hasRight=($ace.FileSystemRights -band $expected.Right) -eq $expected.Right
        if ($hasRight) { Write-Host "  [PASS] $($expected.Identity) -> $($expected.Right)" -ForegroundColor Green }
        else { Write-Host "  [FAIL] $($expected.Identity) wrong rights" -ForegroundColor Red; $pass=$false }
    }
}
Write-Host "`n=== Verification $(if ($pass){"PASSED"}else{"FAILED"}) ===" -ForegroundColor $(if ($pass){"Green"}else{"Red"})
```

### Running it

```powershell
terraform output key_vault_name

.\configure-lab.ps1 -KeyVaultName "kv-fslab-XXXXXXXX"
```

### Known issue: silent step failure

`Invoke-VMScript`'s error handling in `configure-lab.ps1` only logs a warning (via StdErr) when a step fails: it does not stop later steps from running. A failure early in the sequence can go unnoticed while every downstream step still executes against a VM that isn't in the state it expects.

If RDP login as a domain user fails with "the logon attempt failed," check whether the account actually exists before assuming a typo'd password:

```powershell
az vm run-command invoke --resource-group RG-FileServerLab --name DC01 `
  --command-id RunPowerShellScript `
  --scripts "Get-ADUser -Filter {SamAccountName -eq 'sarah.jones'}"
```

An empty result means the AD user/group creation step failed silently. Re-run it standalone against DC01:

```powershell
az vm run-command invoke --resource-group RG-FileServerLab --name DC01 `
  --command-id RunPowerShellScript --scripts "@.\scripts\01-create-ad-users-groups.ps1"
```

### Known issue: the DC promotion step is not re-runnable

Do not re-run `configure-lab.ps1` from scratch once DC01 has already been promoted. `Install-ADDSForest` fails against a domain controller that's already promoted, with an unhelpful error (`"The specified argument 'DomainLevel' was not recognized"`) that doesn't actually say the domain already exists: it's a side effect of `Install-ADDSForest`'s prerequisite-check module, not a script bug.

If a later stage needs to be redone, invoke that individual script against the target VM directly rather than re-running the whole orchestrator:

```powershell
az vm run-command invoke --resource-group RG-FileServerLab --name FS01 `
  --command-id RunPowerShellScript --scripts "@.\scripts\02-configure-shares-and-permissions.ps1"
```

## Step 5: Verify

RDP into CLIENT01's public IP (`terraform output client01_public_ip`) as a domain user, for example `LAB\sarah.jones`. Use "More choices" / "Use a different account" in the credential prompt rather than whatever Microsoft account Windows defaults to; CLIENT01 is domain-joined, not Entra-joined, so it won't offer domain credentials automatically.

Confirm access matches the intended permission model:

| Share | Finance access | HR access | Sales access | IT access |
|---|---|---|---|---|
| `sarah.jones` (GRP_Finance) | Modify | none | none | none |
| `lisa.white` (GRP_HR) | Read | Modify | none | none |
| `tom.davis` (GRP_Sales) | none | none | Modify | none |
| `john.smith` (GRP_IT) | Full Control | Full Control | Full Control | Full Control |

`05-verify-ad.ps1` and `05-verify-shares.ps1` (Stage 7 above) automate the AD-object and NTFS-ACL side of this and print PASS/FAIL per item.

## Cost management

Standard SKU public IPs bill whether or not the attached VM is running: deallocating the VM does not stop that charge. To avoid overnight costs, deallocate the VMs (this does not touch OS disk state, including the AD database; deallocating and reallocating VMs will not undo any AD configuration):

```powershell
az vm deallocate --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait
```

And bring them back:

```powershell
az vm start --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait
```

`az vm allocate` is not the reverse of `az vm deallocate`: it's a different operation (moving a deallocated VM to new hardware) and will not start a VM back up. Use `az vm start`.

Consider setting a budget alert on the state storage account and the lab resource group so unexpected charges surface early:

```powershell
az consumption budget create `
  --budget-name "lab-budget" `
  --amount 20 `
  --time-grain Monthly `
  --category cost `
  --resource-group RG-FileServerLab
```

## Step 6: Tear down

```powershell
terraform destroy
```

This removes everything in `RG-FileServerLab`, including the Key Vault. `RG-TerraformState` is untouched, so re-running `terraform apply` later doesn't require recreating the backend from scratch. If the state storage account itself is no longer needed, delete `RG-TerraformState` separately:

```powershell
az group delete --name RG-TerraformState --yes
```
