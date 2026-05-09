# Azure ARM Template – Ubuntu 24.04 LTS VM Deployment

**Student:** Mike  
**Date:** May 2026  
**Assignment:** Cloud VM Remote Access & Infrastructure as Code with Azure ARM Templates

---

## Overview

This project demonstrates the deployment of a Linux Virtual Machine on Microsoft Azure using an ARM (Azure Resource Manager) template. The deployment covers all aspects of cloud-based VM provisioning — networking, security configuration, remote access via SSH, and secure credential management. It reflects real-world DevOps practices where infrastructure is defined as code and managed remotely without physical access to servers.

---

## What This Template Deploys

All resources are deployed into the `Michael-group` resource group in the `westeurope` Azure region.

| Resource | Name | Purpose |
|---|---|---|
| Virtual Network | `Mike-VM-vnet` | Isolated private network (`10.1.0.0/16`) |
| Subnet | `default` | Segment within the VNet (`10.1.0.0/24`) |
| Network Security Group | `Mike-VM-nsg` | Firewall rules (SSH inbound + deny-all fallback) |
| Public IP Address | `Mike-VM-ip` | Static IPv4 external address (Zone 1) |
| Network Interface | `mike-vm591_z1` | Connects VM to VNet, Public IP, and NSG |
| Virtual Machine | `Mike-VM` | Ubuntu 24.04 LTS, Standard_D2als_v7, Zone 1 |
| OS Disk | `Mike-VM_OsDisk_1` | NVMe-backed disk (auto-deleted on VM delete) |

---

## Prerequisites

### 1. Install Azure CLI

Download and install the Azure CLI:  
👉 https://aka.ms/installazurecliwindows (Windows) or `brew install azure-cli` (macOS)

Verify installation:

```bash
az --version
```

### 2. Log in to Azure

```bash
az login
```

A browser window will open — sign in with your Azure credentials.

### 3. Set Your Active Subscription

```bash
az account list --output table
az account set --subscription "<your-subscription-name>"
```

### 4. Create or Confirm the Resource Group

```bash
az group create --name Michael-group --location westeurope
```

---

## Repository Structure

| File | Description |
|---|---|
| `template.json` | Main ARM template (parameters, variables, resources, outputs) |
| `parameters.json` | Parameter values used during deployment |
| `outputfile.txt` | Captured deployment outputs and resource IDs |
| `screenshots/` | Evidence screenshots of successful deployment and SSH session |
| `README.md` | This documentation file |

---

## Deployment Steps

### Step 1 – Clone the Repository

```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
```

### Step 2 – (Optional) Validate the Template

Always validate before deploying to catch errors early:

```bash
az deployment group validate \
  --resource-group Michael-group \
  --template-file template.json \
  --parameters parameters.json
```

### Step 3 – Deploy the Template

```bash
az deployment group create \
  --resource-group Michael-group \
  --name arm-vm-deployment \
  --template-file template.json \
  --parameters parameters.json \
  --verbose
```

> ⏱️ Deployment typically takes **3–5 minutes**. Live progress will display in the terminal.

### Step 4 – View Deployment Outputs

```bash
az deployment group show \
  --resource-group Michael-group \
  --name arm-vm-deployment \
  --query properties.outputs \
  --output json
```

This returns the following outputs (also captured in `outputfile.txt`):

| Output Key | Value |
|---|---|
| `vmName` | `Mike-VM` |
| `vmResourceId` | `/subscriptions/cc15b4c3-.../Microsoft.Compute/virtualMachines/Mike-VM` |
| `publicIPAddress` | `20.86.51.155` |
| `privateIPAddress` | `10.1.0.4` |
| `sshCommand` | `ssh nzemikez@20.86.51.155` |
| `deploymentLocation` | `westeurope` |

### Step 5 – Retrieve the Public IP Address

Since the Public IP is **Static**, it is allocated at deployment time. You can also retrieve it with:

```bash
az vm show \
  --resource-group Michael-group \
  --name Mike-VM \
  --show-details \
  --query publicIps \
  --output tsv
```

---

## Connecting to the VM via SSH

Use the `sshCommand` output directly:

```bash
ssh nzemikez@20.86.51.155
```

When prompted, enter the administrator password defined in `parameters.json`.

**First connection:** You will see a host authenticity warning. Type `yes` to continue and add the host to your known hosts file.

### Verify System Access

Once connected, confirm the environment with:

```bash
# Check OS version
cat /etc/os-release

# Check available disk space
df -h

# List installed packages
dpkg -l | head -20

# Check network interfaces
ip addr show

# View running processes
top
```

---

## Resource Dependency Order

The ARM template uses `dependsOn` to enforce correct creation order:

```
NSG  ─────────────────────────────────┐
Public IP  ──────────────────────────►  NIC  ──► Virtual Machine
Virtual Network (VNet + Subnet)  ─────┘
```

The NIC is only created after the NSG, Public IP, and VNet are all fully provisioned. The VM is then created last, once the NIC is ready.

---

## Network Security Group Configuration

The `Mike-VM-nsg` enforces the following inbound security rules:

| Rule Name | Direction | Port | Protocol | Priority | Source IP | Action |
|---|---|---|---|---|---|---|
| `SSH` | Inbound | 22 | TCP | 300 | `*` (configurable) | **Allow** |
| `DenyAllInbound` | Inbound | Any | Any | 4096 | `*` | **Deny** |

### Security Notes

- The `allowedSSHSourceIP` parameter accepts a specific IP or CIDR (e.g., `203.0.113.0/24`) to restrict SSH access to trusted sources only. Using `*` is acceptable for testing but **not recommended for production**.
- The `DenyAllInbound` rule at priority 4096 acts as a final catch-all, blocking any traffic not explicitly permitted by a higher-priority rule.
- No RDP (port 3389) rule exists — this is a Linux-only VM accessible exclusively via SSH.

---

## Template Parameters

All parameters are defined in `template.json` and overridden via `parameters.json` at deployment time.

| Parameter | Value Used | Description |
|---|---|---|
| `virtualMachines_Mike_VM_name` | `Mike-VM` | Name of the Virtual Machine |
| `publicIPAddresses_Mike_VM_ip_name` | `Mike-VM-ip` | Public IP Address resource name |
| `virtualNetworks_Mike_VM_vnet_name` | `Mike-VM-vnet` | Virtual Network name |
| `networkInterfaces_mike_vm591_z1_name` | `mike-vm591_z1` | Network Interface Card name |
| `networkSecurityGroups_Mike_VM_nsg_name` | `Mike-VM-nsg` | Network Security Group name |
| `location` | `westeurope` | Azure region for all resources |
| `vmSize` | `Standard_D2als_v7` | VM size/SKU |
| `adminUsername` | `nzemikez` | Administrator login username |
| `adminPassword` | *(secured)* | Administrator password (SecureString) |
| `vnetAddressPrefix` | `10.1.0.0/16` | Virtual Network address space |
| `subnetAddressPrefix` | `10.1.0.0/24` | Default subnet address prefix |

> ⚠️ **Security reminder:** Never commit real passwords to a public repository. Use Azure Key Vault references or environment variables in production deployments.

---

## VM Specifications

| Property | Value |
|---|---|
| OS Image | Ubuntu 24.04 LTS (`ubuntu-24_04-lts`, server SKU) |
| VM Size | `Standard_D2als_v7` |
| Availability Zone | Zone 1 |
| Disk Controller | NVMe |
| OS Disk Delete Behaviour | Deleted with VM |
| Security Type | Trusted Launch (Secure Boot + vTPM enabled) |
| Boot Diagnostics | Enabled (managed storage) |
| Password Authentication | Enabled |

---

## Terminating the Remote Session

Always exit SSH sessions properly to free up resources and maintain security:

```bash
# Option 1 – Exit command
exit

# Option 2 – Keyboard shortcut
Ctrl + D
```

Avoid closing the terminal window directly without exiting, as this can leave orphaned sessions.

---

## Cleanup (Delete Resources When Done)

> ⚠️ **Important:** Delete resources after the assignment to avoid unexpected Azure charges.

```bash
# Delete just the VM (OS disk auto-deletes due to deleteOption: Delete)
az vm delete --resource-group Michael-group --name Mike-VM --yes

# Or delete ALL resources in the resource group at once
az group delete --name Michael-group --yes --no-wait
```

---

## Troubleshooting

| Issue | Likely Cause | Resolution |
|---|---|---|
| SSH connection refused | Port 22 blocked by NSG | Verify the SSH inbound rule exists and is set to Allow |
| SSH connection timeout | Source IP not permitted | Check `allowedSSHSourceIP` matches your current public IP |
| Authentication failed | Wrong username or password | Confirm `adminUsername` and `adminPassword` match `parameters.json` |
| Deployment validation error | Template or parameter mismatch | Run the `validate` command and review the error message |
| VM unreachable after reboot | Public IP changed | Static IP is used — check Azure Portal to confirm IP is still assigned |

---

## Project Goals Achieved

- [x] Configured Network Security Groups (NSGs) with explicit inbound SSH rule and deny-all fallback
- [x] Identified and recorded the Public IP address (`20.86.51.155`) and DNS endpoint
- [x] Established a Linux SSH connection using administrative credentials
- [x] Verified system access by interacting with the VM file system and CLI
- [x] Secured the environment with source IP restrictions on SSH inbound rules
- [x] Demonstrated proper session termination procedures
- [x] Deployed all infrastructure as code using ARM templates

---

*Deployed to Microsoft Azure — West Europe | Ubuntu 24.04 LTS | Standard_D2als_v7*
