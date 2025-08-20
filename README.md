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
