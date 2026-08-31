# siem-ddos-soc-lab
🛡️ Automated SOC home lab simulating distributed TCP SYN floods via Kali Linux, routing through pfSense, and engineering real-time threat detection dashboards inside Splunk.
# SOC Lab: Building, Simulating, and Detecting a Distributed DoS Attack

## 📌 Project Overview
This project details the creation of an end-to-end Security Operations Center (SOC) home lab designed to simulate a Distributed Denial of Service (DDoS) attack. The engineering objective was to construct a full security pipeline: routing high-volume multi-threaded attack traffic across segmented network zones, processing the data through a pfSense firewall, and shipping the resulting telemetry to a centralized Splunk SIEM for threat detection and real-time visualization.

---

## 🏗️ Lab Architecture & Topology
The virtual lab environment is strictly isolated within a hypervisor sandbox using independent host-only subnets to ensure attack traffic never escapes into the production network.

*   **Attacker Network (`ATTACKER` Zone):** Kali Linux VM (`192.168.40.0/24`) acting as the threat vector.
*   **Perimeter Security:** pfSense Virtual Firewall acting as the gateway router and log generator.
*   **Target Network (`DMZ_NET` Zone):** Target Web Server node hosting accessible endpoints at `192.168.20.100:80`.
*   **SIEM Network (`SIEM_NET` Zone):** Centralized Splunk Enterprise instance monitoring infrastructure events.

---

## 🛠️ Configuration & Implementation Steps

### Phase 1: Firewall Rule Engineering (pfSense)
To accurately monitor incoming network floods without crashing the perimeter, a granular ingress policy was built on the **ATTACKER** interface:
*   **Action:** `Pass`
*   **Protocol:** `TCP`
*   **Source:** `Any` *(See Engineering Lessons Learned for details on this configuration)*
*   **Destination:** `Single host or alias` -> `192.168.20.100`
*   **Destination Port Range:** `HTTP (80)` to `HTTP (80)`
*   **Telemetry Generation:** Enabled **"Log packets that are handled by this rule"** to force active syslog generation.

### Phase 2: Remote Log Streaming Setup
1. Inside pfSense, navigated to **Status > System Logs > Settings**.
2. Enabled **Remote Logging Options** pointing directly to the Splunk collection node via custom port **UDP 1514**.
3. Restricted remote syslog streaming payload choices exclusively to **Firewall Events** to reduce indexing noise.

### Phase 3: Launching the Spoofed SYN Flood
From the Kali Linux terminal in the `ATTACKER` zone, a multi-threaded TCP SYN flood was executed to simulate an external botnet infrastructure targeting the web endpoint:

```bash
sudo hping3 -S --flood --rand-source -p 80 192.168.20.100
```
*   `-S`: Floods structured TCP SYN packets to hang the target's connection state tables.
*   `--flood`: Dispatches packets at the maximum raw physical capacity of the virtual link.
*   `--rand-source`: Mandates the packet crafter to randomize public source IP addresses (e.g., `222.36.160.105`), effectively transforming a single-host DoS into a highly distributed footprint.

---

## 📊 Splunk Ingestion & Threat Detection Logic

### Data Normalization
By default, the raw logging interface streams events into Splunk under a generic `sourcetype=syslog`. To enable deep analysis, the **Splunk Add-on for pfSense** was integrated, remapping the UDP 1514 port input data directly to structural `sourcetype=pfsense:filterlog` variables.

### 1. Real-Time Flood Volume Visualization
This tracking query monitors traffic thresholds directly traversing the network boundary:
```spl
index=* sourcetype=pfsense:filterlog dest_ip=192.168.20.100 dest_port=80
| timechart span=1s count by action
```
*   **SOC Analysis:** Normal operation displays minor fluctuations. Upon attack deployment, the dashboard displays a vertical, cliff-like trendline showcasing thousands of connection events passing into the zone.

### 2. High-Diversity Botnet Alert Logic
To trigger real-time defensive operations, this script separates legitimate bulk file requests from true distributed infrastructure floods by evaluating unique source variance ratios:
```spl
index=* sourcetype=pfsense:filterlog dest_ip=192.168.20.100 dest_port=80
| stats count distinct_count(src_ip) as unique_botnet_nodes by dest_ip
| where count > 5000 AND unique_botnet_nodes > 200
```

---

## 💡 Engineering Lessons Learned & Troubleshooting

During the development phase, initial attack validation loops showed that logs were generating inside Splunk, but the traffic action stayed hard-locked onto `match,block,in`, preventing packets from reaching the destination web host.

### The Bug: Source Masking vs. Tool Mechanics
The initial pfSense firewall pass rule limited the accepted traffic **Source** parameter specifically to the local network block `192.168.40.0/24`. However, because the execution layer relied on the `--rand-source` parameter within `hping3`, the outbound IP wrappers were spoofed into random public identities. As a result, pfSense correctly identified that the IP headers did not belong to the allowed internal subnet block and processed them through the default "Block All" clean-up rule.

### The Fix
The firewall configuration was refactored by toggling the accepted packet **Source** criteria to **`Any`**. Because this policy is applied exclusively to physical connections coming through the hardware bounds of the isolated **ATTACKER** interface, it remains secure while allowing the firewall to properly pass spoofed public headers into the DMZ layer.
