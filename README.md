# ServiceNow Quick Discovery

A practical walkthrough of **Quick Discovery** against a small home/lab network using two MID Servers:

- **Windows MID** (`mid.server.win11-mid`) — running on Windows 11 (with workarounds)
- **Linux MID** (`gx10-mid`) — running on an ASUS GX10 / Ubuntu (with workarounds)

Quick Discovery targets a single known IP (or a small list). Full Discovery can scan IP ranges or entire subnets.

---

## Table of Contents

- [What we ran](#what-we-ran)
- [Discovery Log (2026-11-02)](#discovery-log-2026-11-02)
- [Clicking “To Check Pattern Log Press Here”](#clicking-to-check-pattern-log-press-here)
- [Linux Horizontal Discovery](#linux-horizontal-discovery-gx10-c9b9--192168252)
  - [DNS Names related list](#dns-names-related-list)
  - [Related lists on the Linux Server CI](#related-lists-on-the-linux-server-ci)
- [Final asset data (Windows desktop)](#final-asset-data-windows-desktop)
  - [Classic form view](#classic-form-view)
  - [CMDB Workspace — Summary](#cmdb-workspace--summary)
  - [CMDB Workspace — Attributes](#cmdb-workspace--attributes-all)
  - [CMDB Workspace — File Systems](#cmdb-workspace--file-systems)
  - [CMDB Workspace — Running Processes](#cmdb-workspace--running-processes)
  - [CMDB Workspace — Network Adapters](#cmdb-workspace--network-adapters)
- [Discovery Status, Devices, and ECC Queue](#discovery-status-devices-and-ecc-queue)
- [Windows Classify probe](#windows-classify-probe)
- [MID Server capabilities — Windows vs Linux vs Mac](#mid-server-capabilities--windows-vs-linux-vs-mac)
- [Can you discover many machines without Quick Discovery one-by-one?](#can-you-discover-many-machines-without-quick-discovery-one-by-one)
- [What can be discovered without being “on” the device?](#what-can-be-discovered-without-being-on-the-device)
- [Reporting on Discovery per MID Server](#reporting-on-discovery-per-mid-server)
- [What determines which pattern / probe runs?](#what-determines-which-pattern--probe-runs)
- [Screenshots](#screenshots)
- [Related](#related)

---


## What we ran

| MID Server | Host | What it can normally discover | What we actually ran |
|---|---|---|---|
| `mid.server.win11-mid` | Windows 11 | Windows + Linux | Quick Discovery of a Windows desktop (`192.168.2.32`) |
| `gx10-mid` | Linux (ASUS GX10) | Linux only | Quick Discovery of Linux targets |

**Result for the Windows run:** one Computer CI created/updated (`desktop-081g9oh`).

---

## Discovery Log (2026-11-02)

Sample entries from the Discovery Log filtered to “Created on Today”:

| Time | Level | Short Message | Device |
|---|---|---|---|
| 22:47:18 | Information | Discovery started | — |
| 22:50:51 | Information | Discovery completed | — |
| 22:48:04 | Warning | Other TCP connection states found: ["TIME_WAIT"] | 192.168.2.32 |
| 22:47:41 | Information | Active, now Classifying | desktop-081g9oh |
| 22:52:32 | Information | Exploring CI Pattern, Pattern name: **Windows OS - Desktops** | desktop-081g9oh |
| 22:48:33 | Information | Exploring CI Pattern, Pattern name: **Linux Server** | gx10-c9b9 |
| 22:52:52 | Information | Exploring CI Pattern, Pattern name: **Windows OS - Desktops** | desktop-081g9oh |
| 22:47:30 | Information | Classified, now Identifying | 192.168.2.32 |
| 22:48:40 | Information | Launching Pattern for CI type cmdb_ci_docker_engine, skipping creation of Shell CI | gx10-c9b9 |

![Discovery Log from 2026-11-02 showing Windows OS - Desktops and Linux Server patterns](images/discovery-log-today.png)

Key observations:
- The Windows desktop (`desktop-081g9oh` / `192.168.2.32`) was classified via WMI and then explored with the **Windows OS - Desktops** pattern.
- The Linux MID host (`gx10-c9b9` / `192.168.2.52`) triggered the **Linux Server** pattern.
- A warning about TCP `TIME_WAIT` connections appeared during the scan of the Windows desktop (common and usually harmless).

---

## Clicking “To Check Pattern Log Press Here”

From the Discovery Log (the list shown earlier), each pattern exploration line contains a link:

> Exploring CI Pattern, Pattern name: **Linux Server**. To Check Pattern Log Press Here

Clicking that link opens the **Horizontal Discovery Log** for the specific IP/host (in this case `192.168.2.52` / `gx10-c9b9`).

This is the detailed step-by-step trace of what the pattern actually did on the MID Server.

---

## Linux Horizontal Discovery (gx10-c9b9 / 192.168.2.52)

When you click the “To Check Pattern Log Press Here” link on a Linux Server pattern entry, ServiceNow opens the **Horizontal Discovery Log** for that IP.

The log shows the pattern steps that ran on the Linux MID (`mid.server.gx10-mid`):

- Pre Pattern Execution
- Selecting Pattern for Execution
- Pattern checks (UNIX Cluster, ORACLE Clusterware, etc.)
- File Tracking, Target Cleanup
- Creating CI / Creating Relations / Creating References
- Payload Processing (including Delete Handler)

Behind the scenes the MID executed:

- **HorizontalDiscoveryProbe** — Pattern Launcher: Linux Server (input payload contains the compressed result XML)
- **SSHCommand** — `bash ${file:pkg_munger}` (output from the instance telling the MID what to run over SSH)
- **MultiProbe** — Unix - Application Dependency Mapping

The resulting CI is a **Linux Server**:

- **Name:** `gx10-c9b9`
- **Manufacturer:** ASUSTeK COMPUTER INC.
- **Serial:** `TAMSAG013604ANJ`
- **OS:** Linux Ubuntu 24.04.5
- **CPU:** ARM (20 cores)
- **RAM:** 124,544 MB
- **Disk:** 936.21 GB
- **IP:** `192.168.2.52`
- **Description:** Linux gx10-c9b9 … Ubuntu SMP … aarch64 GNU/Linux

Direct link to the CI:  
https://dev296154.service-now.com/now/nav/ui/classic/params/target/cmdb_ci_linux_server.do%3Fsys_id%3D9adeb4c459d053107b64ee18086f95a3

### DNS Names related list

Under **Related Items → Owns::Owned by (parent) - DNS Names** you see multiple DNS Name records owned by this Linux Server.

**What is happening:**

During horizontal discovery the pattern collects DNS information from the host (via `/etc/hosts`, `hostname`, reverse lookups, or the DNS Names sensor). Each name it finds is created as a **DNS Name** CI (`cmdb_ci_dns_name`) and linked to the server with an **Owns / Owned by** relationship.

In this case the host has several names (including Tailscale MagicDNS names ending in `.tail11125f.ts.net`). Discovery created a separate DNS Name record for each and attached them to `gx10-c9b9`. This is normal and useful — it lets you see every name that resolves to the same server without treating them as separate CIs.

### Related lists on the Linux Server CI

**Network Adapters (5)**

| Name | IP Address | Netmask | Notes |
|---|---|---|---|
| docker0 | 172.17.0.1 | 255.255.0.0 | Docker bridge |
| enP7s7 | 192.168.2.52 | 255.255.255.0 | Physical / main interface |
| tailscale0 | 100.125.17.75 | 255.255.255.255 | Tailscale |
| veth… | fe80::… | 64 | Virtual Ethernet (IPv6 link-local) |

**Storage Devices (2)**

| Name | Type | Interface | Size |
|---|---|---|---|
| /dev/nvme0 | Disk | NVMe | 931.5 GB |
| /dev/nvme0n1 | Logical | NVMe | 931.5 GB |

**File Systems (2)**

| Name | Mount | Type | Size | Free |
|---|---|---|---|---|
| /dev/nvme0n1p1 | /boot/efi | VFAT | 511.0 MB | 503.6 MB |
| /dev/nvme0n1p2 | / | Ext4 | 915.3 GB | 43.8 GB |

**Software Installed** — 2,964 packages discovered (sample: 7zip, accountsservice, acl, adcli, adduser, etc.).

**Running Processes** — 251 processes (sample includes cupsd notifier, a user process, dbus-daemon, etc.).

**CI IPs** — Multiple IPv4 and IPv6 addresses attached to the various interfaces (docker0, enP7s7, tailscale0, veth…).

**Memory Modules (1)**

| Name | Capacity (MB) | Form Factor | Type |
|---|---|---|---|
| DIMM0 | 131,076 | Chip | Synchronous |

These related lists are populated automatically by the Linux Server horizontal pattern and the associated probes (SSH commands, package inventory, process list, network interfaces, etc.).

---

## Final asset data (Windows desktop)

After Quick Discovery finished, the CI looked like this:

- **Name:** `desktop-081g9oh`
- **Manufacturer:** Asus
- **Serial number:** `250352463701574`
- **Operating System:** Windows 11 Pro
- **OS Version:** `10.0.26300`
- **OS Domain:** WORKGROUP
- **RAM:** 65,536 MB
- **CPU:** GenuineIntel, 3,200 MHz, 1 CPU, 24 cores
- **Disk space:** 7,899.2 GB
- **IP Address:** `192.168.2.32`
- **FQDN:** `DESKTOP-081G9OH`
- **Discovery source:** ServiceNow
- **Most recent discovery:** 2026-11-02 22:52:52

### Classic form view

![Classic Computer form showing OS, manufacturer, serial, RAM, CPU, and disk](images/computer-ci-form.png)

### CMDB Workspace — Summary

![CMDB Workspace Summary for desktop-081g9oh showing operational status, discovery source, IP, serial, RAM, and CPU](images/cmdb-summary.png)

### CMDB Workspace — Attributes (All)

![CMDB Workspace Attributes tab showing disk space, manufacturer, model, OS, serial, and other fields](images/cmdb-attributes.png)

### CMDB Workspace — File Systems

Discovered drives on the desktop:

| Drive | Available (MB) | Capacity (MB) | Mount |
|---|---|---|---|
| C | 402,183 | 1,906,311 | C:\ |
| D | 425,427 | 457,844 | D:\ |
| E | 509,865 | 1,906,310 | E:\ |
| F | 397,351 | 1,907,600 | F:\ |
| G | 523,076 | 1,907,728 | G:\ |

![CMDB Workspace File Systems related list](images/cmdb-filesystems.png)

### CMDB Workspace — Running Processes

Discovery captured 332 running processes (sample shown below).

![CMDB Workspace Running Processes list](images/cmdb-running-processes.png)

### CMDB Workspace — Network Adapters

- **Name:** Ethernet 3
- **IP Address:** `192.168.2.32`
- **Netmask:** `255.255.255.0`
- **DHCP Enabled:** true
- **MAC Address:** `74:19:F8:16:12:55`

![CMDB Workspace Network Adapters](images/cmdb-network-adapters.png)

---

## Discovery Status, Devices, and ECC Queue

On the Discovery Status record you see three related lists:

- **Discovery Log** — high-level messages
- **Devices** — CIs that were created or updated
- **ECC Queue** — the actual probe traffic (input/output) between the instance and the MID

![Devices related list showing the discovered desktop](images/devices-tab.png)

![ECC Queue showing probes executed by the Windows MID](images/ecc-queue.png)

The ECC Queue for this run shows `mid.server.win11-mid` executing:

- `HorizontalDiscoveryProbe` / Pattern Launcher: Windows OS - Desktops
- `Shazzam` (port scan)
- `WMIRunner` / WMI: Classify
- `WMIRunner` / WMI: Installed Software
- MultiProbe for Application Dependency Mapping

---

## Windows Classify probe

The classification step that identified the machine as Windows used the **Windows - Classify** WMI Probe.

It reads a set of WMI paths and registry keys (ComputerSystem, Win32_OperatingSystem, etc.) to decide the class and OS.

![Windows Classify WMI Probe definition and WMI fields](images/windows-classify-probe.png)

---

## MID Server capabilities — Windows vs Linux vs Mac

| MID Server OS | Can discover Windows | Can discover Linux | Can discover macOS | Notes |
|---|---|---|---|---|
| **Windows** | Yes (WMI) | Yes (SSH) | Limited / not out-of-box | Best general-purpose MID for mixed environments |
| **Linux** | No | Yes (SSH) | Limited | Cannot run WMI probes |
| **macOS** | No | Yes (SSH) | Yes (with credentials) | Rare as a MID host |

**In practice:**
- A Windows MID is the most flexible for a mixed lab.
- A Linux MID is sufficient (and often preferred) for pure Linux estates.
- Discovering a Mac normally requires SSH credentials and a Linux-capable MID (or a Windows MID that can also speak SSH). There is no native WMI-style probe for macOS.

---

## Can you discover many machines without Quick Discovery one-by-one?

**Yes.** Quick Discovery is intentionally limited to known IPs. For a whole subnet (or a long list of devices) you use a **Discovery Schedule** with an IP range.

### Example home/lab network

Imagine a typical setup behind a Bell modem and an ASUS router:

| Device | Type | Typical OS / role | What Discovery can usually see |
|---|---|---|---|
| Bell modem | Network device | Proprietary / embedded | Often only via SNMP if enabled; otherwise just an IP with open ports |
| ASUS router | Network device | Proprietary (Linux-based) | SNMP or basic port scan; rarely a full CI without credentials |
| Windows gaming laptop | Computer | Windows 11 | Full CI if WMI credentials work (OS, hardware, software, processes) |
| Chromebook | Computer | ChromeOS / Linux | Limited — usually port scan only unless SSH is enabled |
| MacBook | Computer | macOS | Possible with SSH credentials; no native WMI-style probe |
| Linux desktop / server | Computer | Ubuntu / etc. | Full CI with SSH credentials |
| iPad | Mobile | iOS | Port scan / basic identification only; no deep inventory |
| Amazon Fire tablet | Mobile | Fire OS (Android) | Port scan only |
| Meta Quest | VR headset | Android-based | Port scan only |
| Smart TV | Consumer device | Proprietary | Port scan or SNMP if available; limited CI |
| Phone (iOS / Android) | Mobile | iOS / Android | Port scan only |

**How you would scan this subnet**

1. Go to **Discovery → Discovery Schedules → New**
2. Set the MID Server (or leave blank to let the system choose)
3. Under **IP Ranges**, add something like `192.168.2.0/24` (or a list of known IPs)
4. Activate the schedule and click **Discover now** (or wait for it to run)

**What the result looks like**

- One Discovery Status record
- Multiple rows in the **Devices** related list (one per responsive host)
- ECC Queue entries for every probe that ran against each IP
- Rich CIs only for devices that responded to WMI (Windows) or SSH (Linux/Mac) with valid credentials
- Everything else appears as a basic IP or network device (or nothing if it is firewalled / powered off)

You do **not** need to run Quick Discovery once per device.

### Practical notes for a mixed home network

- **Windows MID** is the most useful here because it can do WMI for the gaming laptop and SSH for the Linux box and (with credentials) the Mac.
- A **Linux MID** can handle the Linux machines and anything that speaks SSH, but it cannot run WMI probes.
- Most consumer devices (TV, iPad, Fire tablet, Quest, phones) will only show up as IPs with open ports unless they expose SNMP or you have a custom pattern.
- The Bell modem and ASUS router are usually discovered as network devices if SNMP is turned on; otherwise they are just IPs.


## What can be discovered without being “on” the device?

Discovery does not install an agent on the target. It uses:

- **Port scans** (Shazzam) — sees open ports (22, 135, 161, etc.)
- **WMI** (Windows MID only) — pulls OS, hardware, software if credentials are valid
- **SSH** (Linux/Mac) — runs commands or scripts if credentials are valid
- **SNMP** — for network devices
- **Pattern-based horizontal discovery** — follows relationships once a CI is identified

If the host is off, firewalled, or has no valid credentials, you may still see it as an IP with open ports, but you will not get a rich CI (OS, serial, installed software, etc.).

---

## Reporting on Discovery per MID Server

Useful places:

- **Discovery Status** list — filter by `Mid server` or look at the agent that appears in the ECC Queue
- **ECC Queue** — filter where `Agent` = `mid.server.win11-mid` (or your Linux MID)
- **Discovery Log** on a status record
- Reports on `discovery_status` or `ecc_queue` grouped by agent

Example filter on ECC Queue:
```
agent=mid.server.win11-mid^state=processed
```

---

## What determines which pattern / probe runs?

It is **not** simply “the type of MID Server.”

The decision is driven by:

1. **Open ports** found by the Shazzam (port) scan
2. **Credentials** available to the MID for that target
3. **Classification probes** that succeed (e.g. Windows - Classify via WMI, or Linux classification via SSH)
4. **Patterns** that match the classified CI class

A Windows MID can run WMI probes. A Linux MID cannot. But even a Windows MID will only run the Windows classification pattern if WMI responds and credentials work. If only SSH is open and credentials exist, it can still discover Linux boxes.

---

## Screenshots

All screenshots referenced above live in the `images/` folder of this repository. To display them:

1. Create an `images/` folder in the repo (if it does not exist).
2. Upload the PNG files with the exact names used in the markdown above (including `discovery-log-today.png`).
3. Commit and push.

Alternatively, drag the images into the README editor on GitHub and it will generate `user-attachments` URLs automatically.

---

## Related

- MID Server setup (including the Windows 11 and GX10 workarounds): [ServiceNowMidServer](https://github.com/eltonaguiar/ServiceNowMidServer)
- Portfolio overview: [servicenow_portfolio](https://github.com/eltonaguiar/servicenow_portfolio)
