# WordPress on AWS — Highly Available, Multi-Tier Architecture

A production-style deployment of a self-hosted WordPress site on AWS, built to mirror 
real-world infrastructure rather than a single-server setup. The architecture spans 
multiple Availability Zones with a three-tier VPC, shared file storage, load balancing, 
auto scaling, and a custom domain secured with HTTPS.

---

## 📋 Overview

This project hosts a fully functional WordPress site using a layered, fault-tolerant 
AWS architecture. Rather than running WordPress on a single EC2 instance, the app tier 
is horizontally scalable — any number of identical web servers can serve traffic 
simultaneously, all reading and writing to the same shared file system and database.

**Key design goals:**
- No single point of failure across compute, network, or storage
- Web servers never directly exposed to the internet
- Automatic scaling based on demand
- Free, auto-renewing SSL via AWS Certificate Manager

---

## 🏗️ Architecture
<img width="616" height="384" alt="image" src="https://github.com/user-attachments/assets/595619f4-e510-4f2a-b94e-4016b45b5ba4" />


- **Internet Gateway** is the front door of the entire VPC — without it, no traffic 
  from the internet can reach the ALB, and the Bastion Host couldn't be SSH'd into. 
  It handles both inbound traffic (visitors reaching the site) and outbound traffic 
  (Bastion Host reaching the internet). It is attached to the VPC and referenced in 
  the public route table as the target for `0.0.0.0/0`.
- **EFS** lets every EC2 instance — including new ones the Auto Scaling Group launches — 
  share the exact same WordPress files (themes, plugins, uploads) with zero manual sync.
- **RDS** is fully managed, removing the need to maintain a self-hosted MySQL server.
- **Private subnets** keep both the app and data tiers unreachable from the internet — 
  all traffic must pass through the ALB.
- **NAT Gateways** let private instances reach the internet (for package installs/updates) 
  without being reachable *from* the internet.

---

## 🛠️ Tools & Technologies

| Category | Tools |
|---|---|
| **Compute** | Amazon EC2, Auto Scaling Groups, Launch Templates |
| **Networking** | Amazon VPC, NAT Gateway, Internet Gateway, Route Tables, Security Groups |
| **Load Balancing** | Application Load Balancer, Target Groups |
| **Storage** | Amazon EFS (shared file system), Amazon RDS (MySQL) |
| **DNS & SSL** | Amazon Route 53, AWS Certificate Manager (ACM) |
| **Monitoring** | Amazon CloudWatch, Amazon SNS |
| **Application** | WordPress, Apache HTTP Server, PHP, MySQL |
| **Scripting** | Bash (EC2 User Data automation) |

---

## 🔐 Security Design

Traffic flows through a strict, layered chain — each tier only trusts the tier directly 
above it:

| Security Group | Open Ports | Allowed Source |
|---|---|---|
| ALB SG | 80, 443 | Anywhere (0.0.0.0/0) |
| SSH SG | 22 | My IP only |
| Webserver SG | 80, 443, 22 | ALB SG / SSH SG |
| Database SG | 3306 | Webserver SG only |
| EFS SG | 2049 (NFS), 22 | Webserver SG / SSH SG |

This means the web servers, database, and shared file system are **never directly 
reachable from the internet** — even if someone discovered a private IP address, the 
security groups would block the connection.

---

## 🚀 Key Features Implemented

- ✅ Custom three-tier VPC across two Availability Zones
- ✅ Highly available NAT Gateways for outbound-only private subnet internet access
- ✅ Shared application storage via EFS — any EC2 instance can serve the same WordPress files
- ✅ Managed MySQL database via Amazon RDS
- ✅ Application Load Balancer distributing traffic across multiple AZs
- ✅ Auto Scaling Group with CPU-based scaling policies and SNS notifications
- ✅ Custom domain registered and managed through Route 53
- ✅ Free SSL/TLS certificate via ACM with automatic DNS validation and renewal
- ✅ HTTP → HTTPS redirect enforced at the load balancer
- ✅ WordPress configured to correctly detect SSL termination at the ALB

---

## 📸 Screenshots

| | |
|---|---|
| ![Live site](screenshots/live-site.png) | ![VPC Resource Map](screenshots/vpc-resource-map.png) |
| *Live WordPress site over HTTPS* | *VPC architecture (auto-generated resource map)* |
| ![ASG Healthy Targets](screenshots/asg-targets.png) | ![ACM Certificate Issued](screenshots/acm-issued.png) |
| *Auto Scaling Group — healthy targets* | *ACM Certificate — Issued status* |

*(Additional screenshots available in the `/screenshots` folder)*

---

## 💡 Key Lessons Learned

- A **three-tier VPC** is the industry-standard pattern for separating public-facing, 
  application, and data layers — each with its own access controls.
- **NAT Gateways** solve a key networking challenge: letting private servers reach the 
  internet for updates while remaining invisible to inbound traffic.
- **Security Groups chain together** — the Webserver SG only trusts the ALB SG, so even 
  if someone finds a server's IP, they can't reach it directly.
- **EFS enables true horizontal scaling** — without shared storage, every new Auto 
  Scaling instance would start with an empty `/var/www/html` and no WordPress installation.
- **Auto Scaling + Load Balancing work as a team**: the load balancer spreads traffic, 
  and the ASG keeps the right number of healthy servers running — no manual intervention.
- **SSL termination at the load balancer** requires explicitly telling WordPress (via 
  the `HTTP_X_FORWARDED_PROTO` header) that the original request was HTTPS, since the 
  ALB-to-EC2 connection itself is plain HTTP internally.
- Setting up HTTPS is non-negotiable for production — AWS Certificate Manager makes it 
  free and straightforward with DNS validation.

---

## 🧹 Cost Management

This project intentionally uses resources that can rack up hourly charges if left 
running (NAT Gateways, ALB, RDS, EFS, and EC2 instances). After demoing or testing, all 
resources are torn down in dependency order — ASG → Launch Template → ALB/Target Group → 
RDS → EFS → NAT Gateways → Elastic IPs → Security Groups → VPC.
