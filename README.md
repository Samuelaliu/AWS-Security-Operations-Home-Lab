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
