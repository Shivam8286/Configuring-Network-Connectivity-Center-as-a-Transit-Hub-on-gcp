# Configuring Network Connectivity Center as a Transit Hub on GCP

Google Cloud's **Network Connectivity Center (NCC)** enables organizations to interconnect their on-premises networks, branch offices, and other clouds using Google’s global backbone. This guide will walk you through configuring NCC as a transit hub to route traffic securely between two simulated non-Google networks using Google Cloud Platform (GCP).

---


## Table of Contents
1. [Overview](#overview)
2. [Architecture & Terminology](#architecture--terminology)
3. [Prerequisites](#prerequisites)
4. [Step-by-Step Lab Instructions](#step-by-step-lab-instructions)
    - [Task 1: Create the Transit Hub VPC](#task-1-create-the-transit-hub-vpc)
    - [Task 2: Create Remote Branch Office VPCs](#task-2-create-remote-branch-office-vpcs)
    - [Task 3: Configure HA VPNs](#task-3-configure-ha-vpns)
    - [Task 4: Set Up the NCC Hub and Spokes](#task-4-set-up-the-ncc-hub-and-spokes)
    - [Task 5: Deploy VMs and Test End-to-End Connectivity](#task-5-deploy-vms-and-test-end-to-end-connectivity)
5. [Troubleshooting & Verification](#troubleshooting--verification)
6. [References & Further Reading](#references--further-reading)

---

## Overview

Network Connectivity Center (NCC) provides a unified way to manage complex network topologies, allowing seamless and secure communication between distributed environments through Google’s highly available and low-latency backbone. In this guide, you’ll simulate two branch offices (VPCs) connected to a central transit hub, enabling communication between remote sites via HA VPN tunnels managed by NCC.

👉 Think of NCC like a big airport hub.

The hub airport = NCC.
The planes (spokes) = your networks (VPCs, VPNs, Interconnects).
Instead of every city (network) needing a direct flight to every other city, they just fly to the hub.
From the hub, they can reach any other city.
✨ So, NCC = a central airport that makes travel (network connectivity) easy and organized.
---

## Architecture & Terminology

### **Key Components**

- **Hub:**  
  A global GCP resource that acts as a central connection point for multiple spokes (networks or resources). The hub enables communication between attached spokes.

- **Spoke:**  
  A network resource (e.g., HA VPN tunnel, VLAN attachment, or router appliance) that connects to the hub. Spokes are used to link different networks or appliances to the hub.

### **Lab Topology**

- **Transit Hub:**  
  VPC named `vpc-transit` serving as the central GCP network.
- **Branch Offices:**  
  Two VPCs, `vpc-a` and `vpc-b`, simulating remote branch networks.
- **HA VPNs:**  
  Highly available VPN tunnels established between each branch office VPC and the transit hub VPC, using BGP for dynamic routing.
- **NCC Hub and Spokes:**  
  The NCC hub is configured in `vpc-transit`, with HA VPNs to `vpc-a` and `vpc-b` attached as spokes.

---

## Prerequisites

- **Google Cloud account** with billing enabled.
- **Basic knowledge of:**
  - Google VPC Networking
  - Compute Engine
  - Cloud Shell & gcloud CLI
---

## Step-by-Step Lab Instructions

### Task 1: Create the Transit Hub VPC

1. **Delete the Default Network:**
   ```sh
   gcloud compute networks delete default
   ```

2. **Create `vpc-transit` VPC:**
   - Go to **VPC network** in the GCP Console.
   - Click **CREATE VPC NETWORK**.
   - Name: `vpc-transit`
   - **No subnets** required for this transit VPC, so delete the new subnet entry.
   - Set **Dynamic routing mode** to `Global`.
   - Click **Create**.

---

### Task 2: Create Remote Branch Office VPCs

#### **Create `vpc-a`**
- Name: `vpc-a`
- Subnet name: `vpc-a-sub1-use4`
- Region: *Region 1* (e.g., `us-east4`)
- IP range: `10.20.10.0/24`
- Dynamic routing mode: `Regional`

#### **Create `vpc-b`**
- Name: `vpc-b`
- Subnet name: `vpc-b-sub1-usw2`
- Region: *Region 2* (e.g., `us-west2`)
- IP range: `10.20.20.0/24`
- Dynamic routing mode: `Regional`

*You should now see all three VPCs (`vpc-transit`, `vpc-a`, `vpc-b`) listed in the VPC networks console.*

---

### Task 3: Configure HA VPNs

#### **Step 1: Create Cloud Routers (one per VPC per region)**

| Name                   | Network      | Region      | ASN   |
|------------------------|--------------|-------------|-------|
| cr-vpc-transit-use4-1  | vpc-transit  | Region 1    | 65000 |
| cr-vpc-transit-usw2-1  | vpc-transit  | Region 2    | 65000 |
| cr-vpc-a-use4-1        | vpc-a        | Region 1    | 65001 |
| cr-vpc-b-usw2-1        | vpc-b        | Region 2    | 65002 |

*In the console, go to **Network Connectivity > Cloud Router** and create each router as per the table above. Set “Advertise all subnets” as default.*

🔹 Why we need Cloud Router for HA VPN

HA VPN (High Availability VPN) has two tunnels per gateway for redundancy.
To make traffic automatically switch (failover) when one tunnel goes down, the system needs to know which paths are available.
Cloud Router provides this intelligence by running BGP (a routing protocol).
Without Cloud Router, you would have to manually add static routes → no automatic failover.
👉 So, Cloud Router = the air traffic controller that keeps track of all the available routes and tells planes (packets) where to go.
---

#### **Step 2: Create HA VPN Gateways**

| VPN Gateway Name     | VPC Network  | Region      |
|----------------------|--------------|-------------|
| vpc-transit-gw1-use4 | vpc-transit  | Region 1    |
| vpc-transit-gw1-usw2 | vpc-transit  | Region 2    |
| vpc-a-gw1-use4       | vpc-a        | Region 1    |
| vpc-b-gw1-usw2       | vpc-b        | Region 2    |

*Go to **Network Connectivity > VPN > Cloud VPN Gateways** and create each gateway.*


🔹 What is a Gateway in VPN
A gateway is like the entry and exit door for your network traffic.
In cloud VPN, the Cloud VPN Gateway is the device on Google Cloud’s side that terminates (ends) the VPN tunnel.
On your side (on-premises), your firewall/router also acts as a VPN gateway.

🔹 Why we need Gateways
Secure entry/exit point → All encrypted traffic must start and end at a trusted device (gateway).
Tunnel management → Gateways set up and maintain the VPN tunnel (keys, encryption, authentication).
High Availability (HA VPN) → You get two gateways (redundant pairs) in Google Cloud, so if one fails, the other takes over.
Routing integration → Gateways work with Cloud Router + BGP to announce and learn dynamic routes.

✈️ Airport Analogy 
A Gateway is like the airport terminal gate ✈️ → it’s the official point where passengers (data packets) enter or leave the airport (network).
You need gates at both airports (on-prem & cloud).
HA VPN = having two gates so flights can still depart/arrive if one gate is closed.

---

#### **Step 3: Establish VPN Tunnels and BGP Sessions**

For **each direction** (hub to branch and branch to hub), create a pair of tunnels for high availability, and configure BGP sessions for dynamic route exchange.

##### **Example: vpc-transit ↔ vpc-a**

- **From `vpc-transit` to `vpc-a`:**
  - Peer VPN Gateway: `vpc-a-gw1-use4`
  - Cloud Router: `cr-vpc-transit-use4-1`
  - Tunnel names: `transit-to-vpc-a-tu1`, `transit-to-vpc-a-tu2`
  - IKE Pre-shared key: `gcprocks`

- **BGP sessions for above tunnels:**
  - For `transit-to-vpc-a-tu1`:  
    - Session name: `transit-to-vpc-a-bgp1`
    - Peer ASN: `65001`
    - Cloud Router IP: `169.254.1.1`
    - Peer IP: `169.254.1.2`
  - For `transit-to-vpc-a-tu2`:  
    - Session name: `transit-to-vpc-a-bgp2`
    - Cloud Router IP: `169.254.1.5`
    - Peer IP: `169.254.1.6`

- **From `vpc-a` to `vpc-transit`:**
  - Peer VPN Gateway: `vpc-transit-gw1-use4`
  - Cloud Router: `cr-vpc-a-use4-1`
  - Tunnel names: `vpc-a-to-transit-tu1`, `vpc-a-to-transit-tu2`
  - IKE Pre-shared key: `gcprocks`

- **BGP sessions for above tunnels:**
  - For `vpc-a-to-transit-tu1`:  
    - Session name: `vpc-a-to-transit-bgp1`
    - Peer ASN: `65000`
    - Cloud Router IP: `169.254.1.2`
    - Peer IP: `169.254.1.1`
  - For `vpc-a-to-transit-tu2`:  
    - Session name: `vpc-a-to-transit-bgp2`
    - Cloud Router IP: `169.254.1.6`
    - Peer IP: `169.254.1.5`

*Repeat the above steps for `vpc-transit` ↔ `vpc-b` using the appropriate gateways, routers, regions, and BGP IPs.*


🔹 Why we need BGP for Dynamic Routing

BGP (Border Gateway Protocol) lets your network and Google’s network exchange routes automatically.
If a VPN tunnel or connection fails → BGP immediately updates the routes and shifts traffic to the healthy path.
Without BGP, you’d need to update routes manually (slow, risky, and no auto-recovery).
👉 Think of BGP as the live flight schedule system — it updates instantly if a runway (route) is closed, so flights (traffic) are redirected without delay

---

#### **Step 4: Verify VPN Connection Status**

- Navigate to **Network Connectivity > VPN**.
- Ensure all VPN tunnels show **Status: Established** and BGP sessions show **BGP established**.

---

### Task 4: Set Up the NCC Hub and Spokes

#### **Step 1: Enable the Network Connectivity API**

- Go to **APIs & Services > Library**.
- Search for and enable **Network Connectivity API**.

#### **Step 2: Create the NCC Hub**

```sh
gcloud alpha network-connectivity hubs create transit-hub \
   --description="Transit hub"
```

#### **Step 3: Create Spokes for Branch Offices**

- **Branch Office 1:**
  ```sh
  gcloud alpha network-connectivity spokes create bo1 \
      --hub=transit-hub \
      --description="Branch Office 1" \
      --vpn-tunnel=transit-to-vpc-a-tu1,transit-to-vpc-a-tu2 \
      --region=<Region 1>
  ```
- **Branch Office 2:**
  ```sh
  gcloud alpha network-connectivity spokes create bo2 \
      --hub=transit-hub \
      --description="Branch Office 2" \
      --vpn-tunnel=transit-to-vpc-b-tu1,transit-to-vpc-b-tu2 \
      --region=<Region 2>
  ```

---

### Task 5: Deploy VMs and Test End-to-End Connectivity

#### **Step 1: Create Firewall Rules**

- Allow **SSH** and **ICMP (ping)** ingress traffic for both branch office subnets.

| Rule Name | VPC    | Target Subnet          | Ports/Protocols | Source IP Ranges |
|-----------|--------|-----------------------|-----------------|------------------|
| fw-a      | vpc-a  | vpc-a-sub1-use4       | tcp:22, icmp    | 0.0.0.0/0        |
| fw-b      | vpc-b  | vpc-b-sub1-usw2       | tcp:22, icmp    | 0.0.0.0/0        |

#### **Step 2: Launch VM Instances**

- **For `vpc-a`:**
  - Name: `vpc-a-vm-1`
  - Region: `<Region 1>`
  - Zone: `<Zone 1>`
  - Machine: `e2-medium`
  - OS: Debian GNU/Linux 11 (bullseye) x86/64
  - Network: `vpc-a`
  - Subnetwork: `vpc-a-sub1-use4`
  - Boot Disk: 10 GB balanced persistent disk

- **For `vpc-b`:**
  - Name: `vpc-b-vm-1`
  - Region: `<Region 2>`
  - Zone: `<Zone 2>`
  - Machine: `e2-medium`
  - OS: Debian GNU/Linux 11 (bullseye) x86/64
  - Network: `vpc-b`
  - Subnetwork: `vpc-b-sub1-usw2`
  - Boot Disk: 10 GB balanced persistent disk

*Wait for both instances to be running. Note the internal IP of `vpc-b-vm-1`.*

#### **Step 3: Test Connectivity**

- SSH into `vpc-a-vm-1` from the console.
- Ping `vpc-b-vm-1` using its internal IP:

  ```sh
  ping -c 5 <INTERNAL_IP_OF_VPC-B-VM-1>
  ```

- You should see successful ping responses, confirming end-to-end connectivity via NCC.

---

## Troubleshooting & Verification

- If VPN tunnels or BGP sessions are not established, double-check:
  - IKE pre-shared keys match at both ends.
  - BGP ASN and peer IPs are correct.
  - Firewall rules allow necessary protocols.
- Use **Cloud Logging** and **Monitoring** for advanced troubleshooting.

---

## References & Further Reading

- [Network Connectivity Center Documentation](https://cloud.google.com/network-connectivity/docs/network-connectivity-center)
- [Cloud HA VPN Documentation](https://cloud.google.com/network-connectivity/docs/vpn/how-to/creating-ha-vpn)
- [Cloud Router Overview](https://cloud.google.com/network-connectivity/docs/router/concepts/overview)
- [Google Cloud VPC Documentation](https://cloud.google.com/vpc/docs/overview)
- [gcloud CLI Reference](https://cloud.google.com/sdk/gcloud/reference)

---

**Congratulations!**  
You have successfully configured Network Connectivity Center as a transit hub on Google Cloud, enabling secure, highly available connectivity between branch office networks.
