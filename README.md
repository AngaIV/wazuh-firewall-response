# Active Defense SIEM System
**Automated Threat Detection and Incident Response using Wazuh and iptables**

---

## Project Overview

The goal of this project was to move past passive monitoring and build something that could actually respond to a threat on its own. By combining **Wazuh SIEM/XDR** with `iptables`, the system detects SSH brute-force attacks and automatically blocks the source IP without requiring manual intervention.

![Project Overview](assets/screenshots/01-project-overview.png)

---

## Core Components

| Component | Role |
|---|---|
| **Wazuh Manager** | Central hub that ingests logs, evaluates rules, and triggers active responses when thresholds are met. |
| **Victim VM** | Runs the Wazuh Agent and serves as the target machine, with `auth.log` continuously monitored. |
| **Kali Linux VM** | Acts as the threat actor in the simulation, generating SSH brute-force traffic against the victim. |
| **iptables** | Enforces host-level firewall rules, dropping packets from any IP address flagged by Wazuh. |

---

## Lab Network Setup

The environment runs three virtual machines on a shared host-only network, which keeps the attack traffic isolated while still allowing the machines to communicate with each other as they would in a real scenario.

![Network Architecture Diagram](assets/screenshots/02-network-architecture.png)

---

## Configuration and Implementation

### 1. Log Monitoring

The Wazuh Agent is configured to monitor the system authentication log on the victim machine:

```
/var/log/auth.log
```

Each SSH failure gets picked up as it happens. When enough failures occur within a defined window, the relevant detection rules fire.

![Wazuh Agent Log Monitoring](assets/screenshots/03-wazuh-agent-log-monitoring.png)

### 2. Active Response Configuration

Inside `ossec.conf` on the Wazuh Manager, two blocks handle the automated response — one defines the action to take, the other defines when to take it.

**Command definition:**

```xml
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>
```

**Response trigger:**

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5710,5711,5712,5716</rules_id>
  <timeout>600</timeout>
</active-response>
```

The rule IDs referenced here correspond to SSH authentication failures in Wazuh's built-in ruleset. A `timeout` of `600` seconds means the block is automatically lifted after **10 minutes**, which keeps the response proportionate while still disrupting an active attack.

![ossec.conf Active Response Configuration](assets/screenshots/04-ossec-conf-config.png)

### 3. Attack Simulation

A simple loop from the Kali machine was enough to trigger the detection rules:

```bash
# Controlled brute-force simulation
for i in {1..10}; do ssh ghost@192.168.56.103; done
```

The intent was to replicate the kind of repeated authentication failures that Wazuh's SSH rules are designed to catch.

![Kali Linux Brute Force Simulation](assets/screenshots/05-kali-brute-force.png)

---

## Results

### Detection

The Wazuh Manager registered the spike in failed login attempts almost immediately and generated alerts against the relevant rule IDs.

![Wazuh Security Alerts](assets/screenshots/06-wazuh-alerts.png)

### Automated Block

Once the threshold was crossed, Wazuh instructed the Agent to add a `DROP` rule in `iptables` for the attacking IP. The following command confirmed it was in place:

```bash
sudo iptables -L -n
```

![iptables Rule Verification](assets/screenshots/07-iptables-verification.png)

### Audit Trail

The `active-responses.log` file captured a timestamped record of when the block was applied and what triggered it, which is useful for reviewing the response after the fact.

![Active Response Log](assets/screenshots/08-active-response-log.png)

---

## Key Learnings

| Area | What I Took Away |
|---|---|
| **XDR/SIEM Operations** | Gained practical experience configuring Wazuh end-to-end, from agent deployment to manager-side rule tuning. |
| **Network Defense** | Understood how detection events can be chained directly into firewall enforcement using `iptables`. |
| **Incident Response** | Reinforced why automating the containment phase matters — manual response simply cannot match the speed of an automated attacker. |
| **Threat Simulation** | Running the attack myself gave useful perspective on what the logs look like from the defender's side. |
| **Troubleshooting** | Worked through XML syntax issues and rule ID mismatches during setup, which ended up being some of the most instructive parts of the process. |

---

## Skills Applied

- Wazuh SIEM/XDR configuration and management
- SSH brute-force detection
- Linux log monitoring (`auth.log`)
- Active response automation
- `iptables` firewall rules
- Host-only VM networking
- SOC workflow design
- Incident detection and containment
- Controlled adversarial testing with Kali Linux

---

## Summary

A SIEM that only generates alerts is only doing half the job. This project was an exercise in closing the loop — taking a detection and turning it into an action. The implementation is straightforward, but the underlying pattern is the same one used in production environments: detect, decide, respond. Replacing `iptables` with a cloud security group or an EDR integration would follow the same logic at a larger scale.

![Full Lab Overview](assets/screenshots/09-full-lab-overview.png)

> To replicate this setup, you will need Wazuh 4.x, a Linux victim VM with the Wazuh Agent installed, and a Kali Linux machine on the same network. All configuration snippets used in this project are included above.

![Setup Reference](assets/screenshots/10-setup-complete.png)
