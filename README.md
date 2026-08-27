# Home Lab Setup — Enterprise-Style SOC Environment

**Personal Project | Cybersecurity Home Lab**

## Overview

This project documents the build of a personal home lab designed to look, feel, and behave like a small enterprise security environment. Rather than practicing security concepts in isolation — reading about a SIEM here, a firewall there — I wanted one connected environment where I could deploy the tools a real Security Operations Center (SOC) relies on, generate real traffic and real logs, and then go hunt through them myself.

The lab is built entirely on free, industry-standard tools running as virtual machines on a single physical host. It gives me a safe, isolated space to practice vulnerability management, threat detection, incident response, and some penetration testing, without touching a production network or paying for enterprise licensing.

## Why I Built This

Reading about firewalls, SIEMs, and EDR tools only goes so far. I wanted hands-on proof that I could deploy, configure, and actually use this stack the way an analyst would on the job. The primary goals driving this build were:

1. **Reinforce core cybersecurity concepts** by applying them instead of just studying them.
2. **Apply network architecture, access control, and threat mitigation theory** in an environment that behaves like a real one.
3. **Master enterprise-grade security tooling** — the same categories of tools (SIEM, EDR, firewall, vulnerability scanner) that show up in real SOC job descriptions.
4. **Gain hands-on experience deploying, configuring, and querying security tools**, not just reading their documentation.
5. **Understand network telemetry and attack visibility** by capturing and analyzing both north-south (in/out of the network) and east-west (between internal hosts) traffic.
6. **Simulate real attack-and-defense workflows** by running controlled adversary simulations — TTPs mapped to the MITRE ATT&CK framework — against a Windows Active Directory environment and Linux hosts, then hunting for them myself.
7. **Sharpen SOC analyst triage and documentation habits**: building efficient investigation routines, cutting down false positives, and writing clear, professional incident reports with remediation recommendations.

## Core Topology

![Home Lab Network Topology](screenshots/01-topology/homelab-topology-diagram.png)

The lab is built around a central **pfSense** router/firewall that strictly segments traffic across dedicated virtual network interfaces — the same zero-trust principle a real enterprise network uses to keep a compromised host in one zone from freely reaching another.

Four functional zones sit behind that firewall:

- **Attacker Zone** — a Kali Linux VM used as the offensive platform for running scans and simulated attacks against the victim network.
- **Victim / Enterprise Zone** — a Windows Server 2019 Active Directory Domain Controller, Windows 10 workstations, and a handful of intentionally vulnerable Linux boxes (Ubuntu, CentOS, DVWA, VulnHub images) that represent everyday enterprise targets.
- **Network Security Monitoring Zone** — a Security Onion sensor fed by a SPAN port, so it sees a mirrored copy of traffic crossing the firewall without sitting inline.
- **SIEM / Logging Zone** — a Splunk Enterprise server that receives endpoint logs forwarded directly from the victim machines via the Splunk Universal Forwarder.

All of it runs as virtual machines under VMware Workstation Pro on one physical host, with pfSense's multiple virtual network adapters (below) doing the actual work of keeping each zone on its own segment.

![pfSense VM with six isolated virtual network adapters](screenshots/02-vmware-workstation/05-pfsense-vm-network-adapters.png)

## Tool Roles and Why Each One Matters

| Tool | Role in the Lab | Why It Matters |
|---|---|---|
| **pfSense** | Central security gateway — routing and access control between every subnet and the outside network. | This is the enforcement point for the whole zero-trust design. If pfSense isn't configured correctly, every other segmentation decision in the lab is meaningless. It's also where I practice writing and tuning firewall rules, the same skill used to lock down real network perimeters. |
| **Kali Linux** | Offensive platform for penetration testing and simulated attacks against the victim network. | You can't practice detection and response without something generating real attacks to detect. Kali lets me run reconnaissance, exploitation, and post-exploitation techniques mapped to MITRE ATT&CK so the "attacks" I'm hunting for in Splunk and Security Onion are realistic. |
| **Active Directory & Server Stack** (Windows Server 2019, Windows 10 workstations, vulnerable Linux servers) | Represents enterprise targets, user/identity management, and vulnerable workloads. | The vast majority of real-world enterprise environments run on Active Directory, and most attacks eventually touch identity in some way. Standing up and administering a domain controller is core to understanding how lateral movement, privilege escalation, and credential attacks actually work. |
| **Security Onion** | Performs real-time Network Security Monitoring (NSM), full packet inspection, and intrusion detection on a mirrored copy of network traffic from the SPAN port. | This is the network's independent set of eyes — it sees traffic regardless of what's happening (or not being logged) on the endpoint itself. It's how I practice deep packet analysis and threat hunting at the network layer instead of relying purely on host logs. |
| **Splunk Enterprise** | Centralized SIEM that ingests endpoint activity logs forwarded from every victim machine via the Splunk Universal Forwarder. | Splunk is the backbone of the "detect, investigate, respond" workflow that any SOC analyst role is built around. Learning to build searches, dashboards, and correlation logic in Splunk directly transfers to real analyst work. |

## Enterprise Realism — Design Choices That Matter

A few deliberate design decisions push this lab closer to how a real enterprise environment is actually built, instead of a flat, everything-can-talk-to-everything network:

- **Zero-Trust Subnetting** — every zone routes through pfSense's stateful firewall, so nothing moves laterally between zones by default. Any access has to be explicitly allowed.
- **Dual Telemetry (Host + Network)** — Splunk handles host-level log collection while Security Onion handles deep packet inspection at the network level, mirroring the layered visibility a real SOC depends on. An attacker who evades host logging can still be caught on the wire, and vice versa.
- **Out-of-Band Logging** — the monitoring and SIEM infrastructure sits on its own dedicated network segment, separate from production/victim traffic, so log collection never competes with or interferes with the traffic it's analyzing.

The entire environment is built as virtual machines under **VMware Workstation Pro**.

---

## Build Log

### Part 1 — Downloading and Installing VMware Workstation Pro

**Download**

I downloaded VMware Workstation for my OS from [VMware's product page](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion).

![VMware Fusion and Workstation download page](screenshots/02-vmware-workstation/01-vmware-download-page.png)

Clicking Download took me to my account's download page (note: an account is required even though the software itself is free). From there I selected **"free software downloads available here"**, which listed every free download available, and I picked **VMware Workstation Pro 17.6.4**.

Before installing, I verified the downloaded file's integrity by checking its SHA256 hash against the value published by VMware — a habit worth keeping for any installer, since it confirms the file wasn't corrupted or tampered with in transit.

![VMware Workstation Pro setup wizard welcome screen](screenshots/02-vmware-workstation/02-vmware-setup-wizard-welcome.png)

**Installation**

I double-clicked the downloaded installer and went with the default settings to keep things simple. Partway through, the wizard flagged that my host had Hyper-V / Device Credential Guard enabled and offered to install the Windows Hypervisor Platform (WHP) automatically to stay compatible.

![VMware compatibility step for Hyper-V / Credential Guard hosts](screenshots/02-vmware-workstation/03-vmware-hyperv-compatibility-step.png)

I then ran into an installation error: a required file (`vmnetbridge.dll`) could not be found, which halted the install.

![VMware installer "Files Needed" error for a missing DLL](screenshots/02-vmware-workstation/04-vmware-install-missing-file-error.png)

I canceled the install, restarted my computer, and re-ran the installer as Administrator — that combination fixed it. My reasoning: restarting clears out temp files and resets any locked processes or leftover state from the failed attempt, while running as Administrator sidesteps any permission-related blockage that a regular user install can hit when writing system-level network drivers.

With VMware Workstation Pro installed, I built out the VM inventory for the lab (pfSense, Security Onion, the Windows Server 2019 domain controller, a Windows 10 workstation, Kali Linux, and the Splunk server) and set up multiple custom virtual network adapters (VMnet2 through VMnet6) so each functional zone in the topology above could live on its own isolated segment.

---

### Part 2 — pfSense: Firewall, Routing, and Segmentation

pfSense is the piece that turns a pile of separate VMs into an actual segmented network. After deploying it as a VM with six network adapters (one per zone/subnet, shown above), I configured it through its web interface.

![pfSense system status page](screenshots/03-pfsense-firewall/01-pfsense-system-status.png)

I built out the firewall ruleset on the LAN interface to follow least-privilege by default: explicit allow rules for DNS, HTTP, HTTPS, and NTP outbound, and an explicit **deny-all** rule at the bottom to catch everything else. This is the same default-deny posture used in production firewalls — allow only what you know you need, then block the rest.

![Firewall rules table: DNS/HTTP/HTTPS/NTP allowed, default deny-all at the bottom](screenshots/03-pfsense-firewall/02-firewall-rules-lan-outbound.png)

I also set up **pfBlockerNG** for DNS-based blocklisting (DNSBL), which filters outbound requests to known-malicious domains at the DNS layer before they ever reach a host. Getting the DNSBL wizard running required first configuring a Virtual IP for pfSense's localhost interface — I hit a validation error the first time through ("no VIP configured") before going back and setting that up correctly, which was a good reminder that pfBlockerNG has prerequisite steps that aren't obvious from the wizard alone.

![pfBlockerNG DNSBL configuration wizard, with a VIP validation error](screenshots/03-pfsense-firewall/04-pfblockerng-dnsbl-wizard.png)

![pfBlockerNG rule inserted at the top of the firewall rule set](screenshots/03-pfsense-firewall/03-pfblockerng-firewall-rule.png)

---

### Part 3 — Active Directory: Building the Victim/Enterprise Zone

The Windows Server 2019 VM was promoted to a Domain Controller and configured with the core roles an enterprise identity environment depends on: **AD Certificate Services (AD CS)**, **AD Domain Services (AD DS)**, **DNS**, and **File and Storage Services**.

![Windows Server 2019 Server Manager dashboard showing AD CS, AD DS, DNS, and File and Storage Services roles](screenshots/06-active-directory/01-windows-server-2019-ad-roles-dashboard.png)

Windows 10 machines were joined to this domain to act as user workstations — the everyday endpoints that generate the bulk of "normal" activity in the lab, and the machines that later get instrumented with the Splunk Universal Forwarder so their logs flow into the SIEM.

![Windows 10 workstation desktop, joined to the domain and running the Splunk Universal Forwarder](screenshots/07-endpoints/01-windows10-workstation-desktop.png)

---

### Part 4 — Security Onion: Network Security Monitoring

Security Onion is deployed as its own VM with two network roles: a **management interface** (for accessing the web console and administering the box) and one or more **monitor interfaces** connected to pfSense's SPAN port (for passively capturing a mirrored copy of network traffic). The setup wizard walks through both roles step by step:

**1. Internet connectivity mode** — I chose **Direct**, so Security Onion's own update/package traffic (git, docker, curl, yum, etc.) goes straight out rather than through a proxy.

![Security Onion setup — Direct vs. Proxy internet connectivity](screenshots/04-security-onion/01-setup-network-mode.png)

**2. Installation type** — I selected **Standalone**, a single-box deployment appropriate for a home lab, rather than Import, Evaluation, Distributed, or Desktop mode.

![Security Onion setup — choosing Standalone installation](screenshots/04-security-onion/02-setup-installation-type.png)

**3. Install vs. Configure Network** — running the full standard installation rather than a network-only reconfiguration.

![Security Onion setup — Install vs. Configure Network only](screenshots/04-security-onion/03-setup-install-or-configure-network.png)

**4. Node connectivity** — set to **Standard** (this node has internet access), as opposed to an air-gapped install.

![Security Onion setup — Standard vs. Airgap node](screenshots/04-security-onion/04-setup-node-connectivity.png)

Partway through this process, the wizard detected a previous install attempt and warned that continuing would remove that prior configuration — a good reminder that re-running setup on Security Onion is a destructive action, not something to click through casually.

![Security Onion setup — "previous install detected, this is destructive" warning](screenshots/04-security-onion/05-setup-previous-install-warning.png)

**5. Management NIC** — selected the interface used to reach the web console and administer the box.

![Security Onion setup — selecting the management NIC](screenshots/04-security-onion/06-setup-management-nic.png)

**6–8. Management network details** — set a **static IP** (recommended over DHCP for infrastructure like this), then assigned the IPv4 address/CIDR, DNS search domain, and hostname for the box.

![Security Onion setup — static vs. DHCP for the management interface](screenshots/04-security-onion/07-setup-management-ip-method.png)

![Security Onion setup — assigning the static IPv4 address](screenshots/04-security-onion/08-setup-static-ip-assignment.png)

![Security Onion setup — DNS search domain](screenshots/04-security-onion/09-setup-dns-search-domain.png)

![Security Onion setup — hostname](screenshots/04-security-onion/10-setup-hostname.png)

**9. Monitor interface(s)** — dedicated separately from the management NIC, and connected to pfSense's SPAN port so Security Onion passively receives a mirrored copy of network traffic instead of sitting inline.

![Security Onion setup — assigning NICs to the monitor interface](screenshots/04-security-onion/11-setup-monitor-interfaces.png)

Once setup finished, the Security Onion web console came up, giving access to Alerts, Dashboards, Hunt, Cases, PCAP, and the underlying tools it bundles (Kibana, Grafana, CyberChef, and more) — this is where the actual network threat-hunting work happens.

![Security Onion web console — Overview page after a successful install](screenshots/04-security-onion/12-web-console-overview.png)

---

### Part 5 — Splunk Enterprise: Centralized SIEM

Splunk Enterprise runs on its own dedicated Linux VM and receives forwarded logs from every endpoint in the victim zone via the **Splunk Universal Forwarder**.

Getting there wasn't entirely smooth. Early on, the Splunk server VM had a network interface that wouldn't come up correctly (`ens33` stuck "connecting" and failing to obtain an IP), which I had to troubleshoot at the OS level before Splunk itself could be reached over the network.

![Troubleshooting a stuck network interface on the Splunk server VM](screenshots/05-splunk-siem/02-splunk-server-network-troubleshooting.png)

During the Splunk install itself, the installer ran through its configuration checks, validated installed files, and started the `splunkd` daemon — but the web UI wasn't reachable yet while it finished spinning up:

![Splunk install terminal — preliminary checks and splunkd startup](screenshots/05-splunk-siem/01-splunk-install-terminal-checks.png)

![Splunk web UI unreachable immediately after starting the daemon](screenshots/05-splunk-siem/03-splunk-web-ui-first-login.png)

Once the web server finished initializing, Splunk's home page came up normally, and I could start building searches against the forwarded endpoint data:

![Splunk home page after a successful first login](screenshots/05-splunk-siem/04-splunk-search-no-results.png)

One thing worth documenting honestly: an early test search (`index=* host="192.168.2.10"`) returned zero events, which is a normal and expected troubleshooting step right after standing up a new forwarder — it's the point where you go back and confirm the forwarder is actually pointed at the indexer and that the host's data is landing in the index you expect, rather than assuming the pipe is broken.

---

## How This Project Connects to My Other Work

This home lab isn't a standalone exercise — it's the environment where the rest of my cybersecurity study and project work actually gets tested:

- **CompTIA Security+ and CySA+ study logs** — concepts like network segmentation, access control, log analysis, and incident response show up in these certifications as multiple-choice questions first. This lab is where I turn that theory into muscle memory: writing the actual firewall rule, running the actual Splunk search, reading the actual Security Onion alert.
- **Cyber Range Mentorship Program** — the guided exercises and mentorship from that program shaped a lot of the design decisions here, particularly around zero-trust subnetting and dual telemetry (host + network visibility). This lab is where I extend those lessons into a persistent environment I control end-to-end.
- **Automation (PowerShell) scripts** — as the Active Directory and Windows endpoint pieces mature, this is the natural place to apply and test PowerShell automation for tasks like user provisioning, log collection, and routine administration, instead of writing scripts in a vacuum with nothing real to run them against.
- **Projects-Completed / Detection & Response exercises** — this lab is the infrastructure underneath future detection engineering and incident-response writeups: MITRE ATT&CK-mapped attack simulations run from the Kali VM, detected and triaged through Security Onion and Splunk, and documented as standalone incident reports.

In short: the study logs teach the concepts, the mentorship program shapes the approach, and this home lab is the shared enterprise-style environment where all of it gets applied, tested, and turned into evidence of hands-on capability — for penetration testing, threat detection, and incident response alike.

## Tech Stack Summary

- **Hypervisor:** VMware Workstation Pro 17.6.4
- **Firewall / Router:** pfSense (with pfBlockerNG)
- **Offensive Platform:** Kali Linux
- **Enterprise / Victim Zone:** Windows Server 2019 (Active Directory Domain Services, AD CS, DNS, File & Storage Services), Windows 10, Ubuntu, CentOS, DVWA, VulnHub images
- **Network Security Monitoring:** Security Onion (Kibana, Grafana, CyberChef, and its native Alerts/Hunt/Cases tooling)
- **SIEM:** Splunk Enterprise with the Splunk Universal Forwarder

## Status

This is an actively evolving personal lab. Planned next steps include running MITRE ATT&CK-mapped attack simulations from the Kali VM and documenting full detection-to-response walkthroughs using Security Onion and Splunk together.

---

*This project is part of my personal cybersecurity portfolio, documenting hands-on work toward SOC Analyst, Vulnerability Management, and Incident Response roles.*
