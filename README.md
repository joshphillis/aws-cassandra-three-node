# aws-cassandra-three-node

# 🧭 Cassandra 3-Node Cluster Deployment (Script 3 Edition)

This guide walks you through deploying a rack-aware, multi-AZ Cassandra 3.11.3 cluster on AWS EC2 using Ubuntu 22.04. It includes hardened setup steps, a modular configuration script, manual provisioning instructions, and troubleshooting insights based on real-world deployment experience.

---

## 🖥️ Infrastructure Overview

- **Cloud Provider**: AWS EC2  
- **Instances**: 3 (Ubuntu 22.04, t3.small)  
- **Availability Zones**: AZ1, AZ4, AZ6  
- **Security Group**: Open TCP ports `7000`, `7001`, `7199`, `9042`, `9160`  
- **Cluster Name**: `flipbasket-cluster`  
- **Datacenter**: `DC1`  
- **Racks**: `r1`, `r2`, `r3`  

---

## ☁️ AWS Infrastructure Provisioning (Manual via AWS Console)

Before configuring Cassandra, provision the following resources using the AWS Management Console:

### ✅ Step 0: Prerequisites

- AWS account with EC2 access  
- Existing VPC and subnets across 3 Availability Zones  
- SSH key pair created and downloaded (`.pem` file)  
- Security group configured with proper inbound rules  

### 🖥️ 1. Launch 3 EC2 Instances

Provision three EC2 instances with the following settings:

- **AMI**: Ubuntu Server 22.04 LTS (HVM), SSD Volume Type  
- **Instance Type**: `t3.small`  
- **Availability Zones**:
  - Instance 1 → AZ1  
  - Instance 2 → AZ4  
  - Instance 3 → AZ6  
- **Key Pair**: Select your existing key pair  
- **Security Group**: Use the Cassandra SG (see below)  


### 🔐 2. Create a Security Group

Create a new security group named `cassandra-sg` with the following **inbound rules**:

| Type         | Protocol | Port Range | Source           | Purpose                      |
|--------------|----------|------------|------------------|------------------------------|
| SSH          | TCP      | 22         | `0.0.0.0/0`      | Remote access                |
| Custom TCP   | TCP      | 7000       | `172.31.0.0/16`  | Cassandra gossip             |
| Custom TCP   | TCP      | 7001       | `172.31.0.0/16`  | SSL gossip (if enabled)      |
| Custom TCP   | TCP      | 7199       | `172.31.0.0/16`  | JMX (nodetool)               |
| Custom TCP   | TCP      | 9042       | `172.31.0.0/16`  | CQL client access            |
| Custom TCP   | TCP      | 9160       | `172.31.0.0/16`  | Thrift (optional)            |

**Outbound rules**: Allow all traffic (`0.0.0.0/0`)

### 📡 3. Retrieve Private IPs

After launching, go to **EC2 → Instances → Networking tab** and note the **private IP address** of each instance. You’ll use these during Script 3 setup:
- Instance 1 → Seed IP + Private IP
- Instance 2 → Private IP
- Instance 3 → Private IP

---

## 📦 Step 1: Install Dependencies

Run on each node:

```bash
sudo apt update
sudo apt install -y openjdk-8-jdk curl wget unzip
```

---

## 📁 Step 2: Download and Extract Cassandra

```bash
wget https://archive.apache.org/dist/cassandra/3.11.3/apache-cassandra-3.11.3-bin.tar.gz
tar -xvzf apache-cassandra-3.11.3-bin.tar.gz
sudo mv apache-cassandra-3.11.3 /opt/
```

---

## 🛠️ Step 3: Configure with Script 3

Create and run the modular setup script:

```bash
nano cassandra-setup.sh
```

Paste the full Script 3 contents, then:

```bash
chmod 777 cassandra-setup.sh
./cassandra-setup.sh
```

### Script Prompts:
- **This node's private IP**: e.g., `172.31.11.187`  
- **Seed node IP**: Use Instance 1’s private IP  
- **Datacenter name**: `DC1`  
- **Rack name**:
  - Instance 1 → `r1`  
  - Instance 2 → `r2`  
  - Instance 3 → `r3`  

---

## 🧹 Step 4: Wipe Metadata (if rerunning)

```bash
sudo rm -rf /opt/apache-cassandra-3.11.3/data/*
```

---

## 🚀 Step 5: Start Cassandra

```bash
cd /opt/apache-cassandra-3.11.3/bin
./cassandra -R
```

Let it run in the foreground. You should see:

```
No host ID found, created ...
```

---

## 🧪 Step 6: Validate Cluster

Once all nodes are up:

```bash
./nodetool -Dcom.sun.jndi.rmiURLParsing=legacy status
```

Expected output:

```
Datacenter: DC1
=======================
Status=Up/Down
|/ State=Normal/Joining/Leaving/Moving
--  Address         Load       Tokens  Owns (effective)  Host ID                               Rack
UN  172.31.11.187   ...         256     ~33.3%            ...                                   r1
UN  172.31.80.27    ...         256     ~33.3%            ...                                   r2
UN  172.31.31.252   ...         256     ~33.3%            ...                                   r3
```

---

## 🧠 Troubleshooting: Why Nodes Showed 100% Ownership

### Problem
Each node was started before others were reachable, causing isolated bootstrapping.

### Fixes
- Use a **shared seed IP** across all nodes:
  ```yaml
  seeds: "172.31.11.187"
  ```
- Start the **seed node first**
- Ensure **gossip ports (7000, 7001)** are open between nodes
- Avoid listing each node as its own seed

### Best Practice Summary

| Node | Private IP       | Seed IP(s)             | Rack |
|------|------------------|------------------------|------|
| R1   | 172.31.11.187     | 172.31.11.187          | r1   |
| R2   | 172.31.80.27      | 172.31.11.187          | r2   |
| R3   | 172.31.31.252     | 172.31.11.187          | r3   |

---

## ✅ Outcome

- Cassandra ring formed successfully  
- Rack-aware topology across AZs  
- Ownership distributed evenly  
- `nodetool` and `cqlsh` fully operational  

---

## 🪖 Built By

**Joshua Phillis**  
Retired U.S. Army Major | Cloud Infrastructure Engineer | Mentor & Educator  
Specializing in secure Azure/AWS deployments, reproducible labs, and veteran advocacy  
This deployment is part of my [GitHub portfolio](https://github.com/) and reflects my commitment to building resilient, mission-ready cloud infrastructure.
