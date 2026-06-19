# Advanced AZ-104 Lab: Building a "DIY Virtual WAN"

## 🎯 Goal
Replicate the **"Any-to-Any"** routing behavior of Azure Virtual WAN using only standard Azure networking components:
- VNet
- VPN Gateway
- Peering
- UDR (User Defined Routes)
- Azure Route Server

---

## 🏗️ Target Architecture

Allow the **Miami branch** to talk to the **Los Angeles branch** through Azure using **Azure Route Server (ARS)** as the central routing "brain" (normally hidden inside the managed Virtual WAN Hub).

### Logical Diagram

```text
[Miami Office] <--(VPN S2S BGP)--> [HUB VNET EAST US] <--(Global Peering)--> [HUB VNET WEST US] <--(VPN S2S BGP)--> [LA Office]
   (ASN 65001)                     - VPN Gateway (BGP)                        - VPN Gateway (BGP)                     (ASN 65002)
                                   - Azure Route Server                       - Azure Route Server
                                   - Spoke VNet A (Peered)                    - Spoke VNet B (Peered)
```

---

## 🛠️ Prerequisites and Ingredients

| Component | Quantity | Details |
|-----------|----------|---------|
| **Virtual Networks (VNets)** | 4 | 2 Hubs + 2 Spokes |
| **VPN Gateways** | 2 | Minimum `VpnGw1` SKU with **BGP enabled** |
| **Azure Route Server (ARS)** | 2 | One in each Hub |
| **On-Premises Devices** | 2 | Simulated routers (Linux VMs with FRRouting/Quagga for IPsec + BGP) |

---

## 🚀 Lab Deployment Steps

### Phase 1: Foundation (Hub & Spoke and Peering)

#### Create the Networks

| VNet | Address Space | Subnets |
|------|---------------|---------|
| **Hub-East** | `10.1.0.0/16` | `GatewaySubnet`, `RouteServerSubnet` |
| **Hub-West** | `10.2.0.0/16` | `GatewaySubnet`, `RouteServerSubnet` |
| **Spoke-East** | `10.10.0.0/16` | — |
| **Spoke-West** | `10.20.0.0/16` | — |

#### Configure Local Peering
- Connect **Spoke-East** → **Hub-East**
- Enable **"Use remote gateways"** option
- Repeat for **Spoke-West** → **Hub-West**

#### Configure Global Peering
- Interconnect **Hub-East** ↔ **Hub-West**

---

### Phase 2: Gateways and BGP (The Core Logic)

**BGP (Border Gateway Protocol)** is mandatory. It automatically announces:
- To LA router: Miami network is accessible
- To Miami router: LA network is accessible

#### Steps
1. Deploy **VPN Gateway** in **Hub-East**
   - Enable **"Enable BGP"**
   - Assign ASN: `65515`
2. Deploy **VPN Gateway** in **Hub-West**
   - ASN: `65515`
3. Configure **Local Network Gateways**:
   - **Miami**: ASN `65001` + BGP peer IP addresses
   - **LA**: ASN `65002` + BGP peer IP addresses

---

### Phase 3: The Secret Ingredient (Azure Route Server)

This replicates the Virtual WAN magic.

#### Steps
1. Deploy **Azure Route Server** in **Hub-East**
2. Deploy **Azure Route Server** in **Hub-West**
3. Enable **"Branch-to-Branch"** option on **both** ARS instances

> **AZ-104 Explanation**: This option tells ARS to learn routes from Miami VPN and dynamically inject them via Global Peering to LA VPN.

4. Configure **BGP route exchange** between VPN Gateway and Azure Route Server

---

### Phase 4: Routing Tables (UDR - User Defined Routes)

Force Spoke traffic toward the Hub (optionally through NVA/Firewall):

1. Create a **Route Table**
2. Add rule:
   - **Address prefix**: `0.0.0.0/0` (or specific IP ranges of other region)
   - **Next hop**: Central appliance of the Hub
3. Associate table with **Spoke subnets**

---

## 🧪 Validation Tests

Once infrastructure is deployed (~45 minutes):

1. Log in to a machine in the **Miami subnet**
2. Run `traceroute` to a machine in the **Los Angeles subnet**

### ✅ Success Indicator

Traffic path will show:

```text
VPN S2S → Hub East → Microsoft Backbone (Peering) → Hub West → VPN S2S → Los Angeles
```

---

## ⚖️ Conclusion and Reality Check (AZ-104 Exam)

You've functionally built an **Azure Virtual WAN**.

### Drawbacks of the DIY Approach

| Issue | DIY Approach | Virtual WAN |
|-------|--------------|-------------|
| **Deployment time** | Very long (manual BGP, ASNs, ARS, Global Peering, UDRs) | Fast (managed service) |
| **Complexity** | Advanced networking skills needed for BGP troubleshooting among 4 components | Simplified, managed |
| **Scalability** | Manual recreation of objects, BGP table updates for each new office | 3 clicks to connect new site |

### Why Virtual WAN Exists

Virtual WAN takes all the complex architecture you built:
- Gateways + BGP + Route Server + Global Peering

And replaces it with a **single managed service** where connecting a new site takes just **3 clicks**.

---

## 📝 Notes for AZ-104 Exam

- Understand the role of **Azure Route Server** in route exchange
- Know why **BGP** is mandatory for any-to-any routing
- Recognize when to use **Virtual WAN** vs. DIY approach
- Be familiar with **Global Peering** between Hub VNets
- Understand **UDR** purpose for forcing traffic through hubs
