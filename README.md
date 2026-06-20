# 🌐 AZ-104 Advanced Lab: DIY Virtual WAN (NVA & UDR)

> **Difficulty**: Advanced 🔴 | **Time**: 45-60 min | **Cost**: ~$2-5 | **Exam**: AZ-104

---

## 📖 Overview

Replicate Azure Virtual WAN's **"Any-to-Any"** routing using static components:
- Virtual Networks & Global Peering
- VPN Gateways (Site-to-Site, no BGP)
- Network Virtual Appliances (NVAs)
- User Defined Routes (UDRs)
- IP Forwarding on Network Interfaces

**Key Learning**: Master UDRs and IP Forwarding (guaranteed AZ-104 exam topics).

---

## 🏗️ Architecture

### Traffic Flow
```text
Miami → GW_East → NVA-East → (Peering) → NVA-West → GW_West → LA
```

### Key Peering Settings

| Direction | Allow Forwarded Traffic | Use Remote Gateways |
|-----------|------------------------|---------------------|
| Hub-East → Hub-West | ✅ **YES** (critical) | ❌ No |
| Hub-West → Hub-East | ✅ **YES** (critical) | ✅ Yes |
| Spoke-East → Hub-East | ❌ No | ✅ **YES** |
| Spoke-West → Hub-West | ❌ No | ✅ **YES** |

---

## ⚠️ Prerequisites & Cost

### Requirements
- ✅ Azure CLI installed
- ✅ Linux/SSH basics
- ✅ Active Azure subscription

### Cost Warning
| Resource | Quantity | Cost/Hour | 45-min Cost |
|----------|----------|-----------|-------------|
| VPN Gateway (VpnGw1) | 2 | ~$0.10 | ~$0.08 |
| Ubuntu VM (B1s) | 2 | ~$0.02 | ~$0.01 |
| **Total** | | | **~$2-5** |

> **🚨 Delete resource group immediately after completion!**

---

## 🚀 Deployment Guide

### Phase 1: Foundation

```bash
# Create Resource Group
az group create --name AZ104-Lab-RG --location eastus

# Create VNets
az network vnet create --name Hub-East --resource-group AZ104-Lab-RG --location eastus --address-prefixes 10.1.0.0/16 --subnet-name GatewaySubnet --subnet-prefixes 10.1.0.0/27
az network vnet subnet create --name NvaSubnet --resource-group AZ104-Lab-RG --vnet-name Hub-East --address-prefixes 10.1.1.0/26

az network vnet create --name Hub-West --resource-group AZ104-Lab-RG --location westus --address-prefixes 10.2.0.0/16 --subnet-name GatewaySubnet --subnet-prefixes 10.2.0.0/27
az network vnet subnet create --name NvaSubnet --resource-group AZ104-Lab-RG --vnet-name Hub-West --address-prefixes 10.2.1.0/26

az network vnet create --name Spoke-East --resource-group AZ104-Lab-RG --location eastus --address-prefixes 10.10.0.0/16
az network vnet create --name Spoke-West --resource-group AZ104-Lab-RG --location westus --address-prefixes 10.20.0.0/16
```

### Configure Peering (CRITICAL!)

```bash
# Hub-East → Hub-West (Allow forwarded traffic + gateway transit)
az network vnet peering create --name Hub-East-to-Hub-West --resource-group AZ104-Lab-RG --vnet-name Hub-East --peer-vnet Hub-West --allow-forwarded-traffic true --allow-gateway-transit true --use-remote-gateways false

# Hub-West → Hub-East (Allow forwarded traffic + use remote gateways)
az network vnet peering create --name Hub-West-to-Hub-East --resource-group AZ104-Lab-RG --vnet-name Hub-West --peer-vnet Hub-East --allow-forwarded-traffic true --allow-gateway-transit false --use-remote-gateways true

# Spoke → Hub (Use remote gateways)
az network vnet peering create --name Spoke-East-to-Hub-East --resource-group AZ104-Lab-RG --vnet-name Spoke-East --peer-vnet Hub-East --use-remote-gateways true
az network vnet peering create --name Spoke-West-to-Hub-West --resource-group AZ104-Lab-RG --vnet-name Spoke-West --peer-vnet Hub-West --use-remote-gateways true
```

---

### Phase 2: VPN Gateways

```bash
# Create Public IPs
az network public-ip create --name GW-East-PublicIP --resource-group AZ104-Lab-RG --location eastus --sku Standard
az network public-ip create --name GW-West-PublicIP --resource-group AZ104-Lab-RG --location westus --sku Standard

# Create VPN Gateways (NO BGP)
az network vnet-gateway create --name GW-East --resource-group AZ104-Lab-RG --vnet Hub-East --public-ip-addresses GW-East-PublicIP --type Vpn --vpn-type RouteBased --sku VpnGw1 --enable-bgp false
az network vnet-gateway create --name GW-West --resource-group AZ104-Lab-RG --vnet Hub-West --public-ip-addresses GW-West-PublicIP --type Vpn --vpn-type RouteBased --sku VpnGw1 --enable-bgp false

# Create Local Network Gateways
az network local-gateway create --name LGW-Miami --resource-group AZ104-Lab-RG --location eastus --gateway-ip-address $(az network public-ip show --name GW-East-PublicIP --resource-group AZ104-Lab-RG --query ipAddress -o tsv) --address-prefixes 192.168.1.0/24
az network local-gateway create --name LGW-LA --resource-group AZ104-Lab-RG --location westus --gateway-ip-address $(az network public-ip show --name GW-West-PublicIP --resource-group AZ104-Lab-RG --query ipAddress -o tsv) --address-prefixes 192.168.2.0/24

# Create S2S Connections
az network vnet-gateway connection create --name Conn-Miami-East --resource-group AZ104-Lab-RG --vnet-gateway GW-East --local-gateway LGW-Miami --connection-type Vnet2Vnet --shared-key AZ104LabSecretKey123
az network vnet-gateway connection create --name Conn-LA-West --resource-group AZ104-Lab-RG --vnet-gateway GW-West --local-gateway LGW-LA --connection-type Vnet2Vnet --shared-key AZ104LabSecretKey123
```

---

### Phase 3: NVAs (IP Forwarding - Exam Focus!)

```bash
# Deploy Ubuntu VMs
az vm create --name NVA-East --resource-group AZ104-Lab-RG --location eastus --image UbuntuLTS --admin-username azureuser --generate-ssh-keys --vnet-name Hub-East --subnet NvaSubnet --size Standard_B1s
az vm create --name NVA-West --resource-group AZ104-Lab-RG --location westus --image UbuntuLTS --admin-username azureuser --generate-ssh-keys --vnet-name Hub-West --subnet NvaSubnet --size Standard_B1s

# Enable IP Forwarding on vNIC (Portal steps below)
# Enable OS IP Forwarding
az vm run-command invoke --resource-group AZ104-Lab-RG --name NVA-East --command-id RunShellScript --scripts "sudo sysctl -w net.ipv4.ip_forward=1 && echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf"
az vm run-command invoke --resource-group AZ104-Lab-RG --name NVA-West --command-id RunShellScript --scripts "sudo sysctl -w net.ipv4.ip_forward=1 && echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf"

# Get NVA IPs
NVA_EAST_IP=$(az vm show --name NVA-East --resource-group AZ104-Lab-RG --show-details --query 'networkInterfaces.ipAddresses.privateIPAddress' -o tsv)
NVA_WEST_IP=$(az vm show --name NVA-West --resource-group AZ104-Lab-RG --show-details --query 'networkInterfaces.ipAddresses.privateIPAddress' -o tsv)
```

#### Enable IP Forwarding in Azure Portal (MANDATORY!)

**For NVA-East:**
1. Go to **NVA-East** → **Network settings** → Network interface
2. **IP configurations** → **IP forwarding** → Set to **Enabled** → Save

**For NVA-West:** Repeat same steps

> **⚠️ MUST DO**: Without this, Azure SDN drops forwarded packets. **Guaranteed exam topic.**

---

### Phase 4: UDRs (CORRECTED)

```bash
# ──────────────────────────────────────────────────────
# UDR for Spoke-East → NVA-East (for LA traffic)
# ──────────────────────────────────────────────────────
az network route-table create --name RT-Spoke-East --resource-group AZ104-Lab-RG --location eastus
az network route-table route create --resource-group AZ104-Lab-RG --route-table-name RT-Spoke-East --name To-LA-Via-NVA-East --address-prefix 192.168.2.0/24 --next-hop-type VirtualAppliance --next-hop-ip $NVA_EAST_IP
az network vnet subnet update --name default --resource-group AZ104-Lab-RG --vnet-name Spoke-East --route-table RT-Spoke-East

# ──────────────────────────────────────────────────────
# UDR for Spoke-West → NVA-West (for Miami traffic)
# ──────────────────────────────────────────────────────
az network route-table create --name RT-Spoke-West --resource-group AZ104-Lab-RG --location westus
az network route-table route create --resource-group AZ104-Lab-RG --route-table-name RT-Spoke-West --name To-Miami-Via-NVA-West --address-prefix 192.168.1.0/24 --next-hop-type VirtualAppliance --next-hop-ip $NVA_WEST_IP
az network vnet subnet update --name default --resource-group AZ104-Lab-RG --vnet-name Spoke-West --route-table RT-Spoke-West

# ──────────────────────────────────────────────────────
# UDR for NVA-West → GW_West (for LA traffic)
# ──────────────────────────────────────────────────────
az network route-table create --name RT-NVA-West-TO-GW --resource-group AZ104-Lab-RG --location westus
az network route-table route create --resource-group AZ104-Lab-RG --route-table-name RT-NVA-West-TO-GW --name To-LA-Via-GW-West --address-prefix 192.168.2.0/24 --next-hop-type VirtualNetworkGateway
az network vnet subnet update --name NvaSubnet --resource-group AZ104-Lab-RG --vnet-name Hub-West --route-table RT-NVA-West-TO-GW
```

> **⚠️ Critical**: 
> - Associate UDR with **Spoke subnets** (NOT GatewaySubnet)
> - GatewaySubnet UDR breaks VPN Gateway
> - NVA-East → NVA-West uses peering (no UDR needed)

---

## 🧪 Validation Tests

### 1. Verify VPN Gateway Connections
```bash
az network vnet-gateway connection show --name Conn-Miami-East --resource-group AZ104-Lab-RG --query "status"
# Expected: "Connected"
```

### 2. Verify Peering Status
```bash
az network vnet peering show --name Hub-East-to-Hub-West --resource-group AZ104-Lab-RG --vnet-name Hub-East --query "peeringState"
# Expected: "Connected"
```

### 3. Verify UDR Association
```bash
az network vnet subnet show --name default --resource-group AZ104-Lab-RG --vnet-name Spoke-East --query "routeTable.name"
# Expected: "RT-Spoke-East"
```

### 4. Test Traffic Flow (Simulated)
```bash
# SSH to NVA-East and traceroute to LA
az vm run-command invoke --resource-group AZ104-Lab-RG --name NVA-East --command-id RunShellScript --scripts "traceroute 192.168.2.100"

# Expected path:
# 1. NVA-East (10.1.1.10)
# 2. NVA-West (10.2.1.10) [via peering]
# 3. GW_West
# 4. LA (192.168.2.100)
```

---

## 🔧 Troubleshooting

| Issue | Check | Fix |
|-------|-------|-----|
|Traffic doesn't reach LA|Peering "Allow forwarded traffic"|Enable on both Hub-East ↔ Hub-West directions |
|NVA doesn't forward|vNIC IP Forwarding|Enable in Portal (Network Interface → IP configurations) |
|OS won't forward|`net.ipv4.ip_forward`|Run `sudo sysctl -w net.ipv4.ip_forward=1` |
|Spoke can't reach gateway|Peering "Use remote gateways"|Enable on Spoke → Hub peering |
|UDR not working|Associated subnet|Use Spoke subnets (NOT GatewaySubnet) |
|NSG blocking|Network Security Group|Allow inbound from GatewaySubnet, outbound to peered VNet |

---

## 🧹 Clean-Up

```bash
# Delete resource group (async)
az group delete --name AZ104-Lab-RG --yes --no-wait

# Verify deletion
az group exists --name AZ104-Lab-RG
# Expected: false
```

---

## 🎓 Exam Takeaways

| Concept | Why It Matters |
|---------|---------------|
| **IP Forwarding on NIC** | ✅ Guaranteed exam topic - NVA drops traffic without it |
| **OS IP Forwarding** | ✅ Required for Linux packet forwarding |
| **UDR Next Hop Types** | ✅ Know: `VirtualAppliance`, `VirtualNetworkGateway`, `VirtualNetwork` |
| **GatewaySubnet** | ✅ Cannot use UDR for routing Gateway traffic |
| **Allow Forwarded Traffic** | ✅ Critical for NVA across peering |
| **Use Remote Gateways** | ✅ Required for spokes to reach Hub's VPN Gateway |
| **Why Virtual WAN** | ✅ Automates manual UDRs/NVA for 50+ branches |

### Why Virtual WAN?
You wrote **3 UDRs** for just **2 branches**:
1. RT-Spoke-East → NVA-East
2. RT-Spoke-West → NVA-West
3. RT-NVA-West → GW_West

**50 branches?** → 200+ UDRs, 50 NVAs, complex BGP. Virtual WAN automates all of this in 3 clicks.

---

## 🔗 References

- [VNet Peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)
- [VPN Gateway Transit](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-peering-gateway-transit)
- [IP Forwarding](https://learn.microsoft.com/en-us/azure/virtual-network/manage-network-interface-user-defined-route)
- [UDR Tutorial](https://learn.microsoft.com/en-us/azure/virtual-network/tutorial-create-route-table)
- [Virtual WAN](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about)

---

Happy learning! 🚀
