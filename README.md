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
- [ ] Phase 2: pfSense firewall deployment and rule configuration
- [ ] Phase 3: Wazuh SIEM + Suricata IDS/IPS
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

### Security Decisions & Reasoning

**Why manual VPC over the AWS wizard?**
The AWS "VPC and More" wizard auto-generates components silently and can
Create NAT Gateways that incur costs. Building manually ensures full
understanding of every routing decision and keeps the environment
within free tier limits.

**Why no internet route on private subnets?**
Workload and security subnets have zero direct internet exposure by
design. Any traffic in or out must pass through pfSense in the DMZ —
enforcing inspection, logging, and rule-based filtering at a single
controlled gateway.

**Why the same Availability Zone?**
Keeping all subnets in us-east-1a eliminates cross-AZ data transfer
charges, maintaining a strictly $0 lab environment.

### Outcome
A fully segmented AWS network with enforced routing boundaries, 
least-privilege security group rules, and a clear DMZ → private subnet 
architecture ready for firewall and SIEM deployment.
