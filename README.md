# AWS-Security-Operations-Home-Lab
An end-to-end cloud security operations environment built on AWS free tier,
covering network segmentation, firewall controls, SIEM/EDR monitoring,
IDS/IPS, WAF, and automated threat detection across a multi-subnet VPC.

**Tools & Technologies:**
pfSense · Wazuh · Suricata · AWS WAF · OpenVAS · Python · AWS Free Tier

**Skills Demonstrated:**
Network segmentation · Firewall rule management · SIEM ownership ·
IDS/IPS detection · Vulnerability management · Threat detection &
automated response


## Progress
- [x] Phase 1: VPC architecture, subnets, routing, security groups ✅
- [x] Phase 2: pfSense firewall deployment and rule configuration ✅
- [x] Phase 3: Wazuh SIEM + Suricata IDS/IPS ✅
- [ ] Phase 4: AWS WAF + web application target
- [ ] Phase 5: Attack simulation + detection + remediation
- [ ] Phase 6: Automated response + documentation


**Architecture**
<img width="5586" height="952" alt="image" src="https://github.com/user-attachments/assets/22b8ffd9-39f0-4e92-9bf7-da8f5b1bfaaa" />

- **VPC:** 10.0.0.0/16 across 3 segmented subnets
- **DMZ Subnet** (10.0.10.0/24) — pfSense firewall gateway layer
- **Workload Subnet** (10.0.20.0/24) — simulated target/vulnerable host
- **Security Subnet** (10.0.30.0/24) — Wazuh SIEM server

---

## Phase 1: VPC Architecture & Network Segmentation

### Objective
Design and deploy a secure, segmented AWS network architecture that
isolates the workload and security infrastructure from direct internet
exposure — enforcing traffic control through a dedicated firewall layer.

### What I Built

**VPC**
Created a custom VPC (`security-ops-lab-vpc`) with a /16 CIDR block
(10.0.0.0/16), providing 65,536 IP addresses across the environment.
Chosen over the AWS default VPC to maintain full control over routing,
segmentation, and security boundaries.

<img width="1920" height="995" alt="Screenshot 2026-05-26 at 11 12 18 pm" src="https://github.com/user-attachments/assets/7f926d1e-df98-40d6-a64d-3c7d43fcc904" />

<img width="1920" height="993" alt="Screenshot 2026-05-27 at 3 23 50 am" src="https://github.com/user-attachments/assets/f2ba6423-3bf4-4095-aff0-8c4a0b91d86b" />

---

**Subnets**
Three subnets were created within the same Availability Zone (us-east-1a)
to eliminate cross-AZ data transfer costs while maintaining logical
network separation:

| Subnet | CIDR | Purpose |
|---|---|---|
| dmz-subnet | 10.0.10.0/24 | Public-facing firewall layer |
| workload-subnet | 10.0.20.0/24 | Isolated target host |
| security-subnet | 10.0.30.0/24 | Wazuh SIEM server |

<img width="1918" height="978" alt="Screenshot 2026-05-27 at 1 50 45 am" src="https://github.com/user-attachments/assets/31b783e1-c1d2-4d0b-a375-a459307e4f14" />


---

**Internet Gateway & Route Tables**
An Internet Gateway (`security-ops-igw`) was created and attached to the
VPC. Two route tables enforce strict traffic separation:

- **public-rt** — attached to `dmz-subnet` only. Contains a
  `0.0.0.0/0 → IGW` route, allowing inbound/outbound internet traffic
  exclusively through the firewall layer.
- **private-rt** — attached to `workload-subnet` and `security-subnet`.
  Contains no internet route, ensuring these subnets cannot communicate
  directly with the internet. All traffic must pass through pfSense.

<img width="1920" height="998" alt="Screenshot 2026-05-27 at 1 50 16 am" src="https://github.com/user-attachments/assets/63ccf7ae-3b12-483d-84ac-c3507f243399" />
<img width="1920" height="976" alt="Screenshot 2026-05-27 at 1 52 12 am" src="https://github.com/user-attachments/assets/19ca25cf-0b09-44d4-99af-83df55562cb6" />


---

**Security Groups**
Three security groups enforce least-privilege access controls at the
instance level, adding a second layer of defence beyond subnet routing:

**pfsense-sg** — pfSense firewall
- SSH (port 22) restricted to admin IP only
- All internal VPC traffic (10.0.0.0/16) permitted for inter-subnet routing
- HTTP/HTTPS (80/443) open to 0.0.0.0/0 for traffic inspection

**workload-sg** — Target host
- SSH and all traffic restricted to DMZ subnet (10.0.10.0/24) only
- No direct internet access permitted

**wazuh-sg** — Wazuh SIEM server
- SSH restricted to admin IP only
- Ports 1514/1515 (Wazuh agent communication) open to the VPC range
- Port 55000 (Wazuh API) open to the VPC range
- HTTPS (443) restricted to admin IP for dashboard access

<img width="1916" height="995" alt="Screenshot 2026-05-27 at 3 30 05 am" src="https://github.com/user-attachments/assets/428e3a5d-8780-4c8f-a80f-267aa2c218d9" />

---

### My Security Decisions & Reasoning

**Why manual VPC over the AWS wizard?**
The AWS "VPC and More" wizard auto-generates components silently and can
Create NAT Gateways. I'm using the VPC only to build manually to ensure full control of every routing decision.

**Why is there no internet route on private subnets?**
Workload and security subnets have zero direct internet exposure by
design. Any traffic in or out must pass through pfSense in the DMZ
enforcing inspection, logging, and rule-based filtering at a single
controlled gateway.

**Why the same Availability Zone?**
Keeping all subnets in us-east-1a eliminates cross-AZ data transfer
charges, maintaining a strictly $0 lab environment.

### Outcome
A fully segmented AWS network with enforced routing boundaries, 
least-privilege security group rules, and a clear DMZ → private subnet 
architecture ready for firewall and SIEM deployment.

## Phase 2: Firewall Deployment & Network Traffic Control

### My Objective
Deploy and configure a Linux-based firewall gateway in the DMZ subnet 
to control, inspect, and NAT traffic between the Internet and the 
isolated private subnets — enforcing the network segmentation designed 
in Phase 1.

### Instance Details
| Property | Value |
|---|---|
| Name | ubuntu-firewall |
| AMI | Ubuntu Server 26.04 LTS |
| Instance Type | t3.micro |
| Subnet | dmz-subnet (10.0.10.0/24) |
| Private IP | 10.0.10.157 |
| Security Group | pfsense-sg |
| Key Pair | security-ops-lab-key |

<img width="1918" height="939" alt="Screenshot 2026-05-29 at 3 09 18 am" src="https://github.com/user-attachments/assets/c47b3551-931c-4bbb-a391-6b2e4fa14ce5" />
<img width="1920" height="970" alt="Screenshot 2026-05-29 at 3 10 24 am" src="https://github.com/user-attachments/assets/35485a73-c9d6-4403-b729-0561dfb470ac" />
<img width="1920" height="962" alt="Screenshot 2026-05-29 at 3 07 58 am" src="https://github.com/user-attachments/assets/208e688d-6e0e-4ac5-bfb7-7f254ffcf16b" />

### What Was Built

**System Preparation**

Updated all system packages to ensure the instance was fully patched 
before any configuration:

```Command prompt
sudo apt update && sudo apt upgrade -y
```

---

**IP Forwarding**

Enabled IP forwarding at the kernel level — this is what transforms 
the Ubuntu instance from a regular host into a network router/firewall 
capable of forwarding packets between subnets:

```Command on terminal
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
Output confirmed: net.ipv4.ip_forward = 1

Without this setting, the instance would accept packets destined for 
itself but silently discards any packets meant for other hosts, making 
inter-subnet routing impossible.

---
**nftables Firewall Configuration**

Installed and configured nftables as the firewall engine. nftables is 
the modern Linux packet filtering framework, replacing iptables in 
current Ubuntu distributions.

Identified the correct network interface:

```command prompt
ip link show
# Interface: ens5
```
I wrote a full firewall ruleset to `/etc/nftables.conf` covering three 
enforcement layers:

**1. Input Chain — Host Protection (Default: DROP)**
Controls traffic destined for the firewall instance itself:
- Allows established/related connections (stateful inspection)
- Allows loopback interface traffic
- Allows SSH on port 22 (access controlled at security group level)
- Allows ICMP for network diagnostics
- Allows all traffic from within the VPC range (10.0.0.0/16)
- Drops everything else by default

**2. Forward Chain — Inter-Subnet Traffic Control (Default: DROP)**
Controls traffic passing through the firewall between subnets:
- Allows established/related forwarded connections
- Permits workload subnet (10.0.20.0/24) to reach security subnet 
  (10.0.30.0/24) for Wazuh agent communication
- Permits the security subnet to reach the workload subnet for monitoring 
  and response
- Logs and drops all other forwarded traffic with the prefix 
  "nftables-drop:" for SIEM ingestion

**3. NAT Postrouting — Internet Access for Private Subnets**
Masquerades outbound traffic from private subnets through the 
firewall's public interface (ens5):
- Workload subnet (10.0.20.0/24) → NAT → internet
- Security subnet (10.0.30.0/24) → NAT → internet

Full ruleset applied and verified:

```Command prompt
sudo nft list ruleset
```

<img width="1192" height="963" alt="Screenshot 2026-05-29 at 4 01 44 am" src="https://github.com/user-attachments/assets/56fc8d81-7ff6-4441-873b-371be6fd5c95" />

---

**Persistence Configuration**

I enabled nftables as a systemd service to ensure rules survive reboots:

```command prompt
sudo systemctl enable nftables
sudo systemctl restart nftables
sudo systemctl status nftables
```

Status confirmed: `active (exited) — status=0/SUCCESS`

<img width="1279" height="913" alt="Screenshot 2026-05-29 at 3 34 03 am" src="https://github.com/user-attachments/assets/c1765e7f-b735-4c9d-9212-ceef794e13bc" />


---

### My Security Decisions & Reasoning

**Why Ubuntu + nftables instead of pfSense?**
pfSense on AWS Marketplace charges $0.12/hr in software fees 
approximately $87/month on top of EC2 costs. nftables on Ubuntu 
achieves identical firewall outcomes (stateful inspection, NAT, 
inter-subnet routing, packet logging) at zero cost, and requires A 
deeper hands-on understanding of firewall rule construction since 
there is no GUI abstraction.

**Why default DROP policy on input and forward chains?**
Deny-by-default is a core security principle only explicitly 
permitted traffic passes through. Any misconfiguration results in 
traffic would be blocked rather than accidentally permitted, which is 
the safer failure mode in a security environment.

**Why log dropped forward packets?**
The `log prefix "nftables-drop:"` rule on the forward chain means 
every blocked inter-subnet packet is logged to the system journal. 
These logs will be ingested by Wazuh, enabling SIEM 
visibility into blocked traffic patterns and potential lateral 
movement attempts.

**Why NAT from private subnets through the firewall?**
Private subnets have no internet gateway route by design. 
NAT masquerading through the firewall means outbound internet access 
for the Wazuh server and workload host is possible, but all traffic 
is routed and inspectable through a single controlled egress point.

---

### My Outcome
A fully configured Linux firewall gateway running in the DMZ subnet 
with stateful packet inspection, explicit inter-subnet routing rules, 
NAT for private subnet internet access, dropped packet logging for 
SIEM ingestion and persistent configuration across reboots. The 
network is now ready for SIEM and IDS/IPS deployment.

## Phase 3: Wazuh SIEM Deployment & Endpoint Monitoring

### Objective
Deploy a fully functional Security Information and Event Management 
(SIEM) platform in the isolated security subnet, connect it to the 
workload host as a monitored endpoint, and establish real-time security 
event visibility across the environment.

### Instance Details — Wazuh Server
| Property | Value |
|---|---|
| Name | wazuh-server |
| AMI | Ubuntu Server 26.04 LTS |
| Instance Type | t3.small (2GB RAM) |
| Subnet | security-subnet (10.0.30.0/24) |
| Private IP | 10.0.30.42 |
| Security Group | wazuh-sg |
| Access Method | Bastion/jump host via Ubuntu firewall |

### Instance Details — Workload Host
| Property | Value |
|---|---|
| Name | workload-host |
| AMI | Ubuntu Server 26.04 LTS |
| Instance Type | t3.micro |
| Subnet | workload-subnet (10.0.20.0/24) |
| Private IP | 10.0.20.143 |
| Security Group | workload-sg |
| Access Method | Bastion/jump host via Ubuntu firewall |
<img width="1919" height="1000" alt="Screenshot 2026-05-30 at 6 41 01 am" src="https://github.com/user-attachments/assets/47bb69f1-cfab-4512-8b15-0cb80f2834a0" />

### What Was Built

**Wazuh All-in-One Installation**

Installed Wazuh 4.14.5 on the security subnet server using the official 
all-in-one installer, deploying three core components:

- **Wazuh Manager** — the central security event processing engine
- **Wazuh Indexer** — OpenSearch-based log storage and indexing
- **Wazuh Dashboard** — web-based SIEM interface for alert management

```command prompt
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a -i
```

---

**Infrastructure Challenges Resolved**

Several real-world infrastructure problems were encountered and resolved 
during deployment — each one a genuine debugging exercise:

**Challenge 1: Private subnet internet isolation**
The security subnet had no internet gateway route by design (Phase 1). 
Temporarily associated the subnet with the public route table during 
installation, then restored private isolation after completion. This 
mirrors real production patterns where private instances are temporarily 
exposed for patching, then re-isolated.

**Challenge 2: Insufficient RAM on t3.micro**
The Wazuh Indexer (OpenSearch/Java) requires more than 1GB RAM. Resolved 
by adding 2GB swap space and upgrading the instance to t3.small (2GB RAM) 
to ensure the stable operation of all three Wazuh components simultaneously.

```command prompt
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```
**Challenge 3: Version mismatch between manager and agent**
Initial agent installation pulled version 4.14.5 against a 4.7.5 manager. 
Resolved by upgrading all Wazuh components to 4.14.5 for consistency.

**Challenge 4: Missing SSL certificates after upgrade**
Post-upgrade, the dashboard config referenced certificate paths that 
differed from the actual installed filenames. Resolved by updating the 
dashboard configuration to match the correct certificate paths:

```command prompt
sudo sed -i 's|certs/dashboard-key.pem|certs/wazuh-dashboard-key.pem|' \
    /etc/wazuh-dashboard/opensearch_dashboards.yml
sudo sed -i 's|certs/dashboard.pem|certs/wazuh-dashboard.pem|' \
    /etc/wazuh-dashboard/opensearch_dashboards.yml
```

**Challenge 5 — Bastion/jump host architecture**
The security subnet has no direct internet route, making direct SSH 
impossible. Implemented a jump host pattern through the firewall instance:

```command prompt
# Access Wazuh server through firewall bastion
ssh -i ~/.ssh/security-ops-lab-key.pem \
    -J ubuntu@ \
    ubuntu@10.0.30.42
```
---

**Wazuh Agent Deployment on Workload Host**

Installed the Wazuh agent on the workload host to establish endpoint 
monitoring and connect it to the SIEM manager:

```ommand prompt
# Add Wazuh repository
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
    sudo gpg --no-default-keyring \
    --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
    --import

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
    https://packages.wazuh.com/4.x/apt/ stable main" | \
    sudo tee /etc/apt/sources.list.d/wazuh.list


# Install and configure the agent
sudo apt update && sudo apt install wazuh-agent -y
sudo sed -i 's/MANAGER_IP/10.0.30.42/' /var/ossec/etc/ossec.conf
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

**Services Verified Running**

All three Wazuh server components confirmed active:

wazuh-manager:    active (running) ✅

wazuh-indexer:    active (running) ✅

wazuh-dashboard:  active (running) ✅

wazuh-agent:      active (running) on workload host ✅

<img width="1215" height="444" alt="Screenshot 2026-05-30 at 6 57 37 am" src="https://github.com/user-attachments/assets/4d08e051-9425-42f8-a42c-e1a7c014a403" />
<img width="1220" height="520" alt="Screenshot 2026-05-30 at 6 58 40 am" src="https://github.com/user-attachments/assets/80dcece3-6d74-483a-a09a-447e14cb39bb" />
<img width="1222" height="365" alt="Screenshot 2026-05-30 at 6 59 32 am" src="https://github.com/user-attachments/assets/a3d8a1d7-862d-4fbc-82dc-04fde1d7267e" />
<img width="1226" height="354" alt="Screenshot 2026-05-30 at 7 00 01 am" src="https://github.com/user-attachments/assets/ede1abee-d0e1-490d-bd7f-320e89f228b5" />

---

**Dashboard Access — SSH Tunnel**

Since the Wazuh dashboard runs on a private subnet instance, access is 
established via SSH tunnel through the firewall bastion:

```command prompt
ssh -i ~/.ssh/security-ops-lab-key.pem \
    -L 8443:10.0.30.42:443 \
    -N ubuntu@<firewall-public-ip>
```

Browser access: `https://localhost:8443`

<img width="1920" height="993" alt="Screenshot 2026-05-30 at 3 00 57 am" src="https://github.com/user-attachments/assets/a850ccb5-65f9-4165-9a6c-e0ec24e67085" />

I did this to make the tunneling pattern keep the dashboard completely off the public 
internet while remaining accessible for the administration of a standard 
security operations practice.

<img width="1913" height="988" alt="Screenshot 2026-05-30 at 3 01 30 am" src="https://github.com/user-attachments/assets/6ad22a76-fa51-484b-9da3-23874e5cfb8b" />

---

### Results

**Agent successfully connected and monitoring:**

Active agents:    1  (workload-host — 10.0.20.143)

Disconnected:     0

**Live alerts generated within minutes of agent connection:**

Critical severity:  0

High severity:      0

Medium severity:    153

Low severity:       716

Alerts include file integrity monitoring, system configuration checks, 
log collection events, and policy compliance findings — all generated 
automatically by Wazuh's built-in detection rules.

<img width="1917" height="1042" alt="Screenshot 2026-05-30 at 7 44 26 am" src="https://github.com/user-attachments/assets/9bbe056c-f469-4c4f-9d72-af7b9f90e929" />
<img width="1919" height="1038" alt="Screenshot 2026-05-30 at 7 24 43 am" src="https://github.com/user-attachments/assets/1006f618-377c-4175-bd09-f38439296b32" />
<img width="1917" height="1041" alt="Screenshot 2026-05-30 at 6 33 15 am" src="https://github.com/user-attachments/assets/dc1c16d6-58d5-43be-8c02-0995a6bc44c4" />

---

### My Security Decisions & Reasoning

**Why t3.small instead of t3.micro for Wazuh?**
The Wazuh Indexer is a Java/OpenSearch application with a minimum 
practical RAM requirement of 1.5GB during startup. Running it on t3.micro 
caused consistent startup timeouts. t3.small is the minimum 
viable size for a single-node Wazuh deployment.

**Why did I isolate the SIEM server in a private subnet?**
A SIEM server is a high-value target it holds logs of every security 
event across the environment. Direct internet exposure would make it 
vulnerable to the same attacks it's designed to detect. Private subnet 
isolation with bastion-only access follows defence-in-depth principles.

**Why i use a jump host instead of direct access?**
The bastion/jump host pattern creates a single, auditable access point 
into the private network. Every SSH session into the SIEM or workload 
host passes through the firewall instance, which logs the connection. 

**Why Wazuh over commercial SIEM alternatives?**
Wazuh is open-source, free, and used in production by organisations 
globally. It provides SIEM, EDR, file integrity monitoring, vulnerability 
detection, and compliance reporting in a single platform making it 
ideal for demonstrating real security operations capability without 
licensing costs.

---

### Outcome
A fully operational SIEM environment with the Wazuh manager, indexer, 
and dashboard running in an isolated security subnet, one monitored 
endpoint generating real security alerts, and a bastion controlled 
access architecture that mirrors production security operations 
infrastructure.

## Phase 4: AWS WAF + Web Application Deployment

### Objective
Deploy a web application on the workload host, expose it through an 
Application Load Balancer, and protect it with AWS WAF — demonstrating 
real-world web application firewall configuration, rule management, and 
attack detection and blocking.

### Architecture

<img width="4884" height="4404" alt="image" src="https://github.com/user-attachments/assets/0dae6af6-59ef-4351-9a25-9fa28870f404" />

---

### What Was Built

**Web Application — Nginx on Workload Host**

Deployed nginx as a web server on the workload host to serve as the 
WAF target application:

```command prompt
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Created a custom landing page identifying the application:

```html
<h1>AWS Security Operations Lab</h1>
<p>This web application is protected by AWS WAF.</p>
<p>Server: workload-host | Private IP: 10.0.20.143</p>
```
<img width="1431" height="862" alt="Screenshot 2026-06-01 at 4 22 06 am" src="https://github.com/user-attachments/assets/b7a64b98-2fa0-46a8-82bc-c0be42f76d4e" />

---

**Application Load Balancer**

Deployed an internet-facing Application Load Balancer to sit between 
the WAF and the workload host:

| Property | Value |
|---|---|
| Name | security-ops-alb |
| Scheme | Internet-facing |
| Subnets | dmz-subnet (us-east-1a), dmz-subnet-2 (us-east-1b) |
| Security Group | alb-sg (HTTP/HTTPS from 0.0.0.0/0) |
| Listener | HTTP:80 → Forward to security-ops-tg |
| DNS | security-ops-alb-1367897050.us-east-1.elb.amazonaws.com |

<img width="1920" height="993" alt="Screenshot 2026-06-01 at 3 25 19 am" src="https://github.com/user-attachments/assets/cb2f0852-22ca-4976-96f1-cbf711ab4bd3" />

A dedicated target group (`security-ops-tg`) was created pointing to 
the workload host on port 80, with HTTP health checks against `/` to 
monitor nginx availability.

<img width="1920" height="989" alt="Screenshot 2026-06-01 at 2 33 22 am" src="https://github.com/user-attachments/assets/6e18667a-a4be-44a4-8478-663772dbe0a9" />

<img width="1920" height="995" alt="Screenshot 2026-06-01 at 2 33 54 am" src="https://github.com/user-attachments/assets/e1ea89b2-280f-4611-8b4f-655e20a7805c" />

<img width="1920" height="982" alt="Screenshot 2026-06-01 at 2 37 05 am" src="https://github.com/user-attachments/assets/3a35923c-e171-4bb3-a328-474a7e75687f" />


The workload-sg was updated to allow HTTP traffic from alb-sg only
ensuring the workload host is never directly accessible from the 
internet, only through the ALB.

<img width="1684" height="689" alt="Screenshot 2026-06-01 at 3 47 02 am" src="https://github.com/user-attachments/assets/40630913-b57c-40cf-acb1-fdfc7a1915e0" />


---

**AWS WAF Configuration**

Created a Web ACL (`security-ops-waf`) attached to the ALB with three 
AWS managed rule groups:

| Rule Group | WCUs | Protection |
|---|---|---|
| Core Rule Set (CRS) | 700 | XSS, path traversal, malformed requests |
| Known Bad Inputs | 200 | Log4j exploits, SSRF, known attack signatures |
| SQL Injection (SQLi) | 200 | SQL injection pattern detection and blocking |
| **Total** | **1100/5000** | |


