# AWS Production VPC Setup & Highly Available Multi-Tier Architecture

<p align="center">
  <img src="https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonwebservices&logoColor=white" alt="AWS" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" alt="Ubuntu" />
  <img src="https://img.shields.io/badge/VPC-Multi--AZ-blue?style=for-the-badge&logo=amazon-vpc&logoColor=white" alt="VPC" />
  <img src="https://img.shields.io/badge/ALB-Layer_7-orange?style=for-the-badge&logo=awselasticloadbalancing&logoColor=white" alt="ALB" />
  <img src="https://img.shields.io/badge/Auto_Scaling-Self--Healing-green?style=for-the-badge&logo=amazon-ec2&logoColor=white" alt="ASG" />
  <img src="https://img.shields.io/badge/Status-Completed-success?style=for-the-badge" alt="Status" />
  <p align="center">
  <a href="https://youtu.be/JEoUEBt6pOc">
    <img src="https://img.shields.io/badge/YouTube-Watch%20Here-red?style=for-the-badge&logo=youtube&logoColor=white" alt="Watch Video on YouTube" />
  </a>
</p>
</p>

![AWS Production Infrastructure Architecture](public/infra.gif)

---

## Table of Contents
- [AWS Production VPC Setup \& Highly Available Multi-Tier Architecture](#aws-production-vpc-setup--highly-available-multi-tier-architecture)
  - [Table of Contents](#table-of-contents)
  - [1. Objective \& Architecture Overview](#1-objective--architecture-overview)
    - [Architecture Highlights](#architecture-highlights)
  - [2. AWS Infrastructure Components Reference](#2-aws-infrastructure-components-reference)
    - [1. AWS Region](#1-aws-region)
    - [2. AWS VPC (Virtual Private Cloud)](#2-aws-vpc-virtual-private-cloud)
    - [3. AWS Internet Gateway (IGW)](#3-aws-internet-gateway-igw)
    - [4. AWS Availability Zones (AZs)](#4-aws-availability-zones-azs)
    - [5. AWS Subnets (Public \& Private)](#5-aws-subnets-public--private)
    - [6. AWS NAT Gateway](#6-aws-nat-gateway)
    - [7. Application Load Balancer (ALB)](#7-application-load-balancer-alb)
    - [8. Route Tables \& Target Groups](#8-route-tables--target-groups)
    - [9. AWS Security Groups](#9-aws-security-groups)
    - [10. AWS EC2 \& Auto Scaling Group](#10-aws-ec2--auto-scaling-group)
  - [3. End-to-End Implementation Steps](#3-end-to-end-implementation-steps)
    - [Step 1: Create an AWS VPC \& Networking Foundation](#step-1-create-an-aws-vpc--networking-foundation)
      - [VPC Configuration Summary](#vpc-configuration-summary)
      - [Visual Reference](#visual-reference)
    - [Step 2: Create an EC2 Launch Template](#step-2-create-an-ec2-launch-template)
      - [Launch Template Configuration](#launch-template-configuration)
      - [Visual Reference](#visual-reference-1)
    - [Step 3: Deploy an EC2 Auto Scaling Group (ASG)](#step-3-deploy-an-ec2-auto-scaling-group-asg)
      - [Auto Scaling Group Configuration](#auto-scaling-group-configuration)
      - [Visual Reference](#visual-reference-2)
    - [Step 4: Provision a Bastion Host (Jump Server)](#step-4-provision-a-bastion-host-jump-server)
      - [Bastion Host Configuration](#bastion-host-configuration)
      - [Visual Reference](#visual-reference-3)
    - [Step 5: Create Target Group \& Register Targets](#step-5-create-target-group--register-targets)
      - [Target Group Parameters](#target-group-parameters)
      - [Visual Reference](#visual-reference-4)
    - [Step 6: Deploy Application Load Balancer (ALB)](#step-6-deploy-application-load-balancer-alb)
      - [ALB Configuration](#alb-configuration)
      - [Visual Reference](#visual-reference-5)
  - [4. Application Deployment \& Load Balancing Verification](#4-application-deployment--load-balancing-verification)
    - [1. Web Application Source Code](#1-web-application-source-code)
      - [Instance 1 Page (src/instance1.html)](#instance-1-page-srcinstance1html)
      - [Instance 2 Page (src/instance2.html)](#instance-2-page-srcinstance2html)
    - [2. Deployment via Bastion Jump Host](#2-deployment-via-bastion-jump-host)
    - [3. Verification \& Load Distribution Testing](#3-verification--load-distribution-testing)
  - [5. Security Best Practices Checklist](#5-security-best-practices-checklist)
  - [6. Troubleshooting \& Issues Faced (Lessons Learned)](#6-troubleshooting--issues-faced-lessons-learned)
    - [Issue 1: Browser Protocol Mismatch (`https://` instead of `http://`)](#issue-1-browser-protocol-mismatch-https-instead-of-http)
    - [Issue 2: Missing Inbound HTTP (Port 80) Rule in Security Group \& ALB Warning](#issue-2-missing-inbound-http-port-80-rule-in-security-group--alb-warning)
  - [7. Video Tutorial \& Walkthrough](#7-video-tutorial--walkthrough)

---

## 1. Objective & Architecture Overview

The objective of this project is to design, deploy, and validate a secure, highly available, and fault-tolerant production infrastructure on AWS spanning two Availability Zones (`us-east-1a` and `us-east-1b`).

### Architecture Highlights
- **Virtual Private Cloud (VPC):** `prod-main-vpc` (`10.0.0.0/16`) segmented into 2 Public Subnets and 2 Private Subnets.
- **Edge Connectivity:** An Internet Gateway (`prod-main-igw`) for public internet access, and dedicated NAT Gateways in each public subnet to provide private instances with outbound egress (e.g., for security updates and package installations) without inbound exposure.
- **Auto Scaling Group (ASG):** `prod-app-asg` maintaining desired EC2 capacity across private subnets for high availability and self-healing.
- **Administrative Access:** A hardened Bastion Host (`BASTION HOST`) deployed in a public subnet providing secure SSH jump access to backend private instances.
- **Traffic Distribution:** An Internet-Facing Application Load Balancer (`prod-app-alb`) listening on HTTP port 80 and forwarding incoming user traffic to the Target Group (`prod-app-tg`) on port 8000 across private application instances.

## 2. AWS Infrastructure Components Reference

<details>
<summary><strong>AWS Infrastructure Components Details (Click to Expand)</strong></summary>

### 1. AWS Region
- Defines the geographical location hosting AWS physical infrastructure (e.g., `us-east-1` N. Virginia).

### 2. AWS VPC (Virtual Private Cloud)
- A logically isolated private virtual network.
- Configured with IPv4 CIDR `10.0.0.0/16` providing 65,536 private IP addresses.

### 3. AWS Internet Gateway (IGW)
- A redundant, horizontally scaled VPC component allowing bidirectional communication between VPC public resources and the internet.

### 4. AWS Availability Zones (AZs)
- Physically separated, isolated data centers within a region (`us-east-1a` and `us-east-1b`) ensuring fault tolerance against single data center failure.

### 5. AWS Subnets (Public & Private)
- **Public Subnets:** Routed directly to the Internet Gateway (`0.0.0.0/0 → IGW`). Used for internet-facing resources like ALBs and Bastion hosts.
- **Private Subnets:** Isolated from direct inbound internet traffic. Outbound internet egress is routed via NAT Gateways (`0.0.0.0/0 → NAT Gateway`). Used for application workloads and databases.

### 6. AWS NAT Gateway
- Managed Network Address Translation service allowing private subnet instances to initiate outbound connections to the internet while blocking unsolicited incoming traffic.

### 7. Application Load Balancer (ALB)
- Layer 7 OSI load balancer that inspects HTTP/HTTPS headers and content to route incoming traffic intelligently across target groups and instances.

### 8. Route Tables & Target Groups
- **Route Tables:** Determine network packet routing destinations (`0.0.0.0/0 → IGW` for public subnets; `0.0.0.0/0 → NAT-GW` for private subnets).
- **Target Groups:** Register and perform continuous health checks on backend compute targets (EC2 instances).

### 9. AWS Security Groups
- Stateful virtual firewalls operating at the instance/ENI level controlling allowed inbound and outbound protocols, ports, and source IP/security groups.

### 10. AWS EC2 & Auto Scaling Group
- Reusable Launch Templates paired with Auto Scaling Groups automatically provision, monitor, and scale instance fleets to maintain target capacity across multiple AZs.

</details>

---

## 3. End-to-End Implementation Steps

### Step 1: Create an AWS VPC & Networking Foundation

The foundation consists of a dedicated VPC (`prod-main-vpc`) configured with a `/16` IPv4 CIDR range, spanning across 2 Availability Zones (`us-east-1a` and `us-east-1b`) with 4 subnets, route tables, an Internet Gateway, and NAT Gateways.

#### VPC Configuration Summary
| Parameter                      | Value                                                                                                              |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| **VPC Name**                   | `prod-main-vpc`                                                                                                    |
| **VPC ID**                     | `vpc-02d29df80a6a26efe`                                                                                            |
| **IPv4 CIDR**                  | `10.0.0.0/16`                                                                                                      |
| **Tenancy**                    | Default                                                                                                            |
| **DNS Resolution / Hostnames** | Enabled                                                                                                            |
| **Public Subnets**             | `prod-main-subnet-public1-us-east-1a` (`10.0.0.0/20`), `prod-main-subnet-public2-us-east-1b` (`10.0.16.0/20`)      |
| **Private Subnets**            | `prod-main-subnet-private1-us-east-1a` (`10.0.128.0/20`), `prod-main-subnet-private2-us-east-1b` (`10.0.144.0/20`) |
| **Gateways**                   | `prod-main-igw`, `prod-main-nat-public1-us-east-1a`, `prod-main-nat-public2-us-east-1b`                            |

#### Visual Reference
![VPC Details](public/01.png)
*Figure 1.1: VPC Details page for `prod-main-vpc` showing CIDR `10.0.0.0/16` and Available status.*

![VPC Resource Map](public/02.png)
*Figure 1.2: Complete VPC Resource Map showing 4 subnets, 4 route tables, Internet Gateway, and 2 NAT Gateways.*

---

### Step 2: Create an EC2 Launch Template

A launch template standardizes the virtual machine instance configuration for all application servers launched manually or by Auto Scaling.

#### Launch Template Configuration
- **Launch Template Name:** `prod-app-launch-template`
- **Template Version Description:** `Prod WEB SERVER`
- **Auto Scaling Guidance:** Checked
- **AMI:** Ubuntu Server 26.04 LTS (HVM), General Purpose SSD Volume (`ami-0b6d9d3d33ba97d99`, 64-bit x86)
- **Instance Type:** `t3.micro`
- **Key Pair:** `bastionhost01`
- **Security Group:** `prod-app-sg`
  - Inbound SSH (Port `22`)
  - Inbound Custom TCP (Port `8000`)
- **Storage:** 8 GiB General Purpose SSD (`gp3`)

#### Visual Reference
![Launch Template Name & Guidance](public/03.png)
*Figure 2.1: Defining template name `prod-app-launch-template` and enabling Auto Scaling guidance.*

![AMI Selection](public/04.png)
*Figure 2.2: Selecting Canonical Ubuntu Server 26.04 LTS (HVM).*

![Instance Type & Key Pair](public/05.png)
*Figure 2.3: Choosing `t3.micro` instance type and associating key pair `bastionhost01`.*

![Network Settings & Security Group](public/06.png)
*Figure 2.4: Specifying VPC `prod-main-vpc` and creating security group `prod-app-sg`.*

![Inbound Security Rules](public/07.png)
*Figure 2.5: Inbound firewall rules allowing SSH (port 22) and application traffic (port 8000).*

![Storage Configuration](public/08.png)
*Figure 2.6: Configuring EBS root volume (8 GiB gp3).*

![Template Creation Success](public/09.png)
*Figure 2.7: Confirmation banner confirming `prod-app-launch-template` creation.*

![Launch Template Details](public/10.png)
*Figure 2.8: Reviewing launch template details, version 1 (Default).*

---

### Step 3: Deploy an EC2 Auto Scaling Group (ASG)

The Auto Scaling Group ensures high availability by managing application instances across the private subnets.

#### Auto Scaling Group Configuration
| Parameter             | Setting                                                                        | Purpose                                         |
| --------------------- | ------------------------------------------------------------------------------ | ----------------------------------------------- |
| **ASG Name**          | `prod-app-asg`                                                                 | Identifies the Auto Scaling fleet               |
| **Launch Template**   | `prod-app-launch-template` (v1)                                                | Provisions identical instance configurations    |
| **VPC**               | `prod-main-vpc`                                                                | Network boundary                                |
| **Subnets**           | `prod-main-subnet-private1-us-east-1a`, `prod-main-subnet-private2-us-east-1b` | Keeps instances securely inside private subnets |
| **AZ Distribution**   | Balanced best effort                                                           | Distributes instances evenly across AZs         |
| **Desired Capacity**  | `2`                                                                            | Baseline running instances                      |
| **Minimum Capacity**  | `1`                                                                            | Lowest allowable scale-in threshold             |
| **Maximum Capacity**  | `4`                                                                            | Upper bound burst capacity                      |
| **Health Check Type** | EC2 (Grace period: 300s)                                                       | Replaces unhealthy instances automatically      |

> [!NOTE]
> During initial ASG setup, load balancing integration is skipped in the wizard. The target group and Application Load Balancer are created and attached in subsequent steps.

#### Visual Reference
![Auto Scaling Landing Page](public/11.png)
*Figure 3.1: EC2 Auto Scaling Groups dashboard.*

![ASG Step 1 - Choose Template](public/12.png)
*Figure 3.2: Setting ASG name `prod-app-asg` and selecting `prod-app-launch-template`.*

![ASG Step 1 - Template Preview](public/13.png)
*Figure 3.3: Verifying launch template version and hardware specifications.*

![ASG Step 2 - VPC Selection](public/14.png)
*Figure 3.4: Selecting the target VPC `prod-main-vpc`.*

![ASG Step 2 - Subnets Selection](public/15.png)
*Figure 3.5: Choosing private subnets across `us-east-1a` and `us-east-1b` with balanced best effort.*

![ASG Step 2 - Capacity Reservation](public/16.png)
*Figure 3.6: Additional capacity settings configured to default.*

![ASG Step 4 - Group Size Configuration](public/17.png)
*Figure 3.7: Configuring group sizing (Desired: 2, Min: 1, Max: 4).*

![ASG Step 4 - Scaling Policies](public/18.png)
*Figure 3.8: Setting scaling policy and instance maintenance policies.*

![ASG Step 4 - Additional Settings](public/19.png)
*Figure 3.9: Additional settings for monitoring, warmup, and deletion protection.*

![ASG Review 1](public/20.png)
*Figure 3.10: Reviewing Step 1 Launch template and Step 2 Network options.*

![ASG Review 2](public/21.png)
*Figure 3.11: Reviewing capacity reservation and verifying load balancing is deferred.*

![ASG Review 3](public/22.png)
*Figure 3.12: Reviewing health check grace period (300 seconds) and desired capacity (2).*

![ASG Review 4](public/23.png)
*Figure 3.13: Reviewing scaling limits and notification settings.*

![ASG Review 5 & Create](public/24.png)
*Figure 3.14: Final review before clicking "Create Auto Scaling group".*

![ASG Active Dashboard](public/25.png)
*Figure 3.15: Auto Scaling group active status with 2/2 healthy instances.*

![Running EC2 Instances](public/26.png)
*Figure 3.16: Running EC2 instances automatically launched by the ASG in private subnets.*

---

### Step 4: Provision a Bastion Host (Jump Server)

To securely administer EC2 instances running in private subnets without exposing them to the internet, a Bastion Host is deployed in a public subnet.

#### Bastion Host Configuration
- **Instance Name:** `BASTION HOST`
- **AMI:** Ubuntu Server 26.04 LTS
- **Instance Type:** `t3.micro`
- **Key Pair:** `bastionhost01`
- **VPC:** `prod-main-vpc`
- **Subnet:** `prod-main-subnet-public1-us-east-1a` (Public Subnet)
- **Auto-assign Public IP:** `Enable`
- **Security Group:** `bastion-host-sg` (Inbound SSH Port 22)
- **Allocated Public IP:** `100.52.181.146` (Private IP: `10.0.1.50`)

#### Visual Reference
![Bastion Host Name & AMI](public/27.png)
*Figure 4.1: Launching `BASTION HOST` with Ubuntu 26.04 LTS.*

![Bastion Instance Type](public/28.png)
*Figure 4.2: Selecting instance type `t3.micro`.*

![Bastion Network Settings](public/29.png)
*Figure 4.3: Attaching to public subnet `prod-main-subnet-public1-us-east-1a` with Public IP enabled.*

![Bastion Security Group](public/30.png)
*Figure 4.4: Creating dedicated security group `bastion-host-sg` with port 22 access.*

![Bastion Storage](public/31.png)
*Figure 4.5: Storage volume configuration (8 GiB gp3).*

![Bastion Launching Progress](public/32.png)
*Figure 4.6: AWS console launching the bastion host instance.*

![Bastion Running Details](public/33.png)
*Figure 4.7: Bastion host active with public IP `100.52.181.146` and private IP `10.0.1.50`.*

---

### Step 5: Create Target Group & Register Targets

Before deploying the Application Load Balancer, a target group is created to route requests to backend instances on port 8000 and monitor their health.

#### Target Group Parameters
- **Target Group Name:** `prod-app-tg`
- **Target Type:** `Instances`
- **Protocol:** `HTTP`
- **Port:** `80` (with targets registered on application port `8000`)
- **VPC:** `prod-main-vpc`
- **Health Check Protocol / Path:** `HTTP` / `/`
- **Registered Targets:**
  - `SRV01` (`i-0983d3b372a30275f`) in `us-east-1a` (Port: `8000`)
  - `SRV02` (`i-0e3fcd6cf7a9b7871`) in `us-east-1b` (Port: `8000`)

#### Visual Reference
![Target Group Step 1](public/34.png)
*Figure 5.1: Creating target group `prod-app-tg` with target type "Instances".*

![Target Group Network](public/35.png)
*Figure 5.2: Specifying HTTP protocol, port, and VPC `prod-main-vpc`.*

![Target Group Health Checks](public/36.png)
*Figure 5.3: Configuring HTTP health checks targeting root path `/`.*

![Register Targets](public/37.png)
*Figure 5.4: Selecting backend application instances `SRV01` and `SRV02`.*

![Targets Included as Pending](public/38.png)
*Figure 5.5: Reviewing targets included as pending registration.*

![Target Group Review 1](public/39.png)
*Figure 5.6: Reviewing target group configuration and health check thresholds.*

![Target Group Review 2 & Create](public/40.png)
*Figure 5.7: Reviewing registered targets before clicking "Create target group".*

![Target Group Success Banner](public/41.png)
*Figure 5.8: Target group `prod-app-tg` successfully created.*

![Target Group Registered Targets](public/42.png)
*Figure 5.9: Target group targets tab showing `SRV01` and `SRV02` ready for load balancer association.*

---

### Step 6: Deploy Application Load Balancer (ALB)

The ALB distributes incoming internet HTTP traffic across the registered targets in the public subnets.

#### ALB Configuration
| Parameter                    | Setting                                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------ |
| **Load Balancer Name**       | `prod-app-alb`                                                                       |
| **Type**                     | Application Load Balancer (ALB)                                                      |
| **Scheme**                   | Internet-facing                                                                      |
| **IP Address Type**          | IPv4                                                                                 |
| **VPC**                      | `prod-main-vpc`                                                                      |
| **Mappings (AZs & Subnets)** | `us-east-1a` (`prod-main-subnet-public1`), `us-east-1b` (`prod-main-subnet-public2`) |
| **Security Group**           | `prod-app-sg`                                                                        |
| **Listeners & Routing**      | `HTTP:80` → Forward to Target Group `prod-app-tg`                                    |
| **DNS Name**                 | `prod-app-alb-1598181641.us-east-1.elb.amazonaws.com`                                |

#### Visual Reference
![ALB Basic Config](public/43.png)
*Figure 6.1: Configuring ALB name, Internet-facing scheme, and IPv4 address type.*

![ALB Network Mapping](public/44.png)
*Figure 6.2: Mapping ALB across public subnets in `us-east-1a` and `us-east-1b`.*

![ALB Security Groups & Listener](public/45.png)
*Figure 6.3: Associating security group and configuring listener on HTTP port 80.*

![ALB Forward Action](public/46.png)
*Figure 6.4: Setting default routing action to forward traffic to target group `prod-app-tg`.*

![ALB Summary Review](public/47.png)
*Figure 6.5: Reviewing load balancer configuration before final deployment.*

![ALB Active List](public/48.png)
*Figure 6.6: Load balancers dashboard displaying `prod-app-alb` with Active status.*

![ALB Details & DNS](public/49.png)
*Figure 6.7: ALB details page showing ARN, hosted zone, and public DNS name.*

![ALB Listener Rules](public/50.png)
*Figure 6.8: Verifying listener rule forwarding 100% of HTTP:80 traffic to `prod-app-tg`.*

---

## 4. Application Deployment & Load Balancing Verification

### 1. Web Application Source Code
The private instances run lightweight HTTP web servers serving distinct content from [src/instance1.html](./src/instance1.html) and [src/instance2.html](./src/instance2.html) to demonstrate round-robin load distribution.

#### Instance 1 Page ([src/instance1.html](./src/instance1.html))

#### Instance 2 Page ([src/instance2.html](./src/instance2.html))


### 2. Deployment via Bastion Jump Host
Using SSH Agent Forwarding from the administrator workstation:

```bash
# Change the permisson for the file
chmod 400 bastionhost01.pem

# 1. Connect to Bastion Host
ssh -i bastionhost01.pem ubuntu@100.52.181.146

# 2. From Bastion, SSH into Private Instance 1
ssh -i bastionhost01.pem ubuntu@10.0.135.64

# Start web server on port 8000
python3 -m http.server 8000

# 3. From Bastion, SSH into Private Instance 2
ssh ubuntu@10.0.157.63
# Start web server on port 8000
python3 -m http.server 8000
```

### 3. Verification & Load Distribution Testing
Navigating to the ALB DNS endpoint (`http://prod-app-alb-1598181641.us-east-1.elb.amazonaws.com`) distributes incoming client requests across the healthy instances:

![Instance 1 Web Page](public/51.png)
*Figure 7.1: First request routed by ALB to Instance 1 (`Welcome to Instance1`).*

![Instance 2 Web Page](public/52.png)
*Figure 7.2: Subsequent request routed by ALB to Instance 2 (`Welcome to Instance2`), validating high availability and load distribution.*

---

## 5. Security Best Practices Checklist

- [x] **No Direct Public IP for Application Servers:** EC2 instances reside exclusively in private subnets.
- [x] **Least-Privilege Security Groups:**
  - ALB Security Group permits inbound HTTP (80) and HTTPS (443) from `0.0.0.0/0`.
  - Application Security Group restricts port 8000 inbound traffic strictly to the ALB security group.
  - SSH (port 22) to private instances is restricted strictly to the Bastion host security group (`bastion-host-sg`).
- [x] **Bastion Jump Server Hardening:** Bastion is placed in a public subnet with restricted IP whitelisting for administrative access.
- [x] **Redundant Multi-AZ Egress:** Dedicated NAT Gateways deployed across both Availability Zones (`us-east-1a` and `us-east-1b`) prevent cross-AZ data transfer fees and single-point-of-failure egress bottlenecks.
- [x] **Auto Scaling Self-Healing:** The ASG maintains desired capacity across AZs, replacing degraded nodes seamlessly.

---

## 6. Troubleshooting & Issues Faced (Lessons Learned)

During the deployment and testing of this production VPC infrastructure, the following real-world issues were encountered and resolved:

### Issue 1: Browser Protocol Mismatch (`https://` instead of `http://`)
- **Symptoms:** Entering the Load Balancer DNS name into the browser address bar initially resulted in a site unreachable error (`ERR_SSL_PROTOCOL_ERROR` or connection refused).
- **Root Cause:** Modern web browsers often auto-complete or default web addresses to `https://`. However, our Application Load Balancer listener is configured exclusively on **HTTP (Port 80)** without a TLS/SSL certificate attached from AWS Certificate Manager (ACM). Since there was no HTTPS listener active on port 443, the connection was rejected.
- **Solution:** Explicitly typed `http://` before the Load Balancer DNS name in the web browser:
  ```text
  http://prod-app-alb-1598181641.us-east-1.elb.amazonaws.com
  ```
  > [!TIP]
  > For a live domain in production, request a public certificate in AWS Certificate Manager (ACM), configure an HTTPS (port 443) listener, and set up an HTTP-to-HTTPS redirect rule on port 80.

---

### Issue 2: Missing Inbound HTTP (Port 80) Rule in Security Group & ALB Warning
- **Symptoms:** In the AWS Console, the Application Load Balancer displayed a warning stating that its security group does not permit inbound traffic on the listener port. Additionally, external HTTP requests to port 80 were timed out / dropped.
- **Root Cause:** Both the ALB listener and Target Group were set to **HTTP (Port 80)**. However, the associated Security Group (`prod-app-sg`) initially only had inbound rules for SSH (port 22) and application traffic (port 8000). Because security groups are stateful firewalls that drop all traffic not explicitly allowed, incoming client requests on port 80 were blocked at the load balancer level.
- **Solution:** Updated the Security Group rules in the VPC dashboard:
  1. Selected the Security Group associated with the ALB.
  2. Added an Inbound Rule:
     - **Type:** `HTTP`
     - **Protocol:** `TCP`
     - **Port Range:** `80`
     - **Source:** `0.0.0.0/0` (Anywhere IPv4)
  3. Saved the rules.
  
  This instantly cleared the warning in the ALB console and enabled external browser traffic to reach the load balancer and distribute to the target group instances.

---

## 7. Video Tutorial & Walkthrough

> [!TIP]
> 📺 **Watch the complete step-by-step video walkthrough on YouTube:**
> 
> [![Watch Video on YouTube](https://img.shields.io/badge/YouTube-Watch%20Here-red?style=for-the-badge&logo=youtube)](https://youtu.be/JEoUEBt6pOc)
> 
> **Video Link:** [https://youtu.be/JEoUEBt6pOc](https://youtu.be/JEoUEBt6pOc)  
