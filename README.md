# Mini-SOC Lab — pfSense, Suricata, Wazuh

## Overview

This project is a small Security Operations Center (SOC) built in a virtual lab, as a final-year project (PFE) for a master's program interview in cybersecurity. The goal is simple: build a network that can detect attacks and show them in a clear dashboard.

Three tools work together:

- **pfSense** — the firewall. It splits the network into safe zones and controls which machine can talk to which.
- **Suricata** — the intrusion detection system (IDS). It watches the network traffic and raises an alert when it sees something suspicious.
- **Wazuh** — the SIEM (Security Information and Event Management). It collects the alerts, understands them, and shows them in a dashboard.

## Architecture

The lab is built with VirtualBox and has three network zones on pfSense:

- **WAN** — represents the outside world.
- **LAN** — the "trusted" internal zone. The attacker machine (Kali Linux) is placed here, to simulate an attacker who is already inside the network.
- **OPT1 / DMZ** — a separate, more controlled zone, where the sensitive machines live.

Machines and their IP addresses:

| Machine | IP | Role |
| --- | --- | --- |
| Kali Linux | LAN | Attacker, launches the test attacks |
| Ubuntu Victim | 192.168.2.100 | Target of the attacks, runs the Wazuh agent |
| Ubuntu SIEM | 192.168.2.101 | Simple client used only to open the Wazuh dashboard in a browser |
| Wazuh Manager | 192.168.2.102 | The real SIEM engine — receives logs, analyzes them, runs the dashboard |

Putting the attacker (Kali) and the victim in different zones is intentional: it forces all attack traffic to pass through the firewall, which is exactly where Suricata is listening.

## Suricata — Intrusion Detection

Suricata runs directly on pfSense, in IDS mode (it detects attacks, it does not block them). It is active on both the LAN and OPT1 interfaces, so traffic is inspected in both directions.

Besides the standard ETOpen ruleset, three custom rules were written to detect specific attacks:

```
# Custom Suricata rules - applied on LAN and OPT1 interfaces
alert tcp any any -> $HOME_NET 4444 (msg:"Possible Reverse Shell - Port 4444"; flags:S,CE; sid:1000001; rev:1;)
alert tcp $HOME_NET any -> any any (msg:"Possible Bind Shell Activity"; flow:to_client,established; content:"/bin/bash"; sid:1000002; rev:1;)
alert tcp any any -> $HOME_NET 22 (msg:"Possible SSH Brute Force"; flow:to_server; flags:S; threshold: type threshold, track by_src, count 5, seconds 60; sid:1000003; rev:1;)
```

- **SID 1000001** — catches traffic on port 4444, a port commonly used for reverse shells (Netcat, Metasploit).
- **SID 1000002** — looks for the text `/bin/bash` in the traffic, a sign that a shell was opened and used.
- **SID 1000003** — counts SSH connection attempts; if the same source IP tries 5 times in 60 seconds, it is flagged as a brute-force attempt.

## Wazuh — Collecting and Understanding the Alerts

Suricata alerts are sent from pfSense to the Wazuh Manager over syslog. On the Wazuh side, two custom files turn the raw text log into a clear, structured alert:

- **Decoder** (`local_decoder.xml`) — reads the raw Suricata log line and pulls out the useful fields: source IP, destination IP, ports, protocol, and the Suricata rule ID.
- **Rules** (`local_rules.xml`) — look at those extracted fields and decide if an alert should be raised, with a severity level and a clear description.

Once this is in place, an alert that used to look like plain, unreadable text becomes a clean entry in the dashboard, with the attacker's IP, the victim's IP, and a human-readable description such as "Possible Reverse Shell detected".

## Detection Pipeline

This is the full path an attack takes, from the moment it happens to the moment it is visible on screen:

1. Kali attacks Ubuntu Victim (for example, a reverse shell on port 4444).
2. The traffic crosses pfSense, between the LAN and OPT1 zones.
3. Suricata inspects the traffic and matches one of the custom rules.
4. pfSense sends the alert to the Wazuh Manager over syslog.
5. The Wazuh decoder extracts the fields from the raw alert.
6. The Wazuh rule checks those fields and raises a structured alert.
7. The alert appears in the Wazuh dashboard, with full details.

In parallel, the Wazuh agent installed on Ubuntu Victim also reports its own host-level events (for example, Apparmor blocking a suspicious action), adding a second layer of visibility on top of the network detection.

## Results

All three custom attacks were tested end to end — reverse shell, bind shell, and SSH brute force — and confirmed working all the way to the dashboard. The Wazuh dashboard shows the alerts with correctly extracted source and destination IPs, not just raw text.

## What I Learned

Building this lab meant debugging real, practical problems: fixing XML syntax errors in Wazuh's configuration files, freeing up disk space taken by Wazuh's vulnerability database, finding a missing checkbox in pfSense's logging settings, and understanding exactly how Suricata and Wazuh talk to each other. These problems — and solving them — are as much a part of this project as the final working result.
