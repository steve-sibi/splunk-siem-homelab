# Splunk SIEM Homelab - A full overview
NOTE: I'm working on a cloud implementation (because my laptop has been through enough torture) and will make a new read-me when that's complete for anyone interested.

# Overview
This was a personal project of mine I wanted to take on to build experience in architecting and using Splunk in a simulated environment. This guide walks through setting up a SIEM+SOAR homelab using Splunk Enterprise 9.4.1 and Splunk SOAR 6.4.0 on a local VMware environment.  I will cover the lab requirements, installation steps, log forwarding configuration, data ingestion, building dashboards, writing correlation searches (SPL queries), and designing simple SOAR playbooks. Screenshots will be added later for clarity.

# System Requirements and Lab Architecture

Before diving in, these are the system specs I tried to meet to run Splunk and SOAR VMs (I had to nerf the specs because I only have so much compute :( ) :
- Splunk Enterprise Server (Ubuntu 20.04/22.04): ~2 vCPU, 4 GB RAM, 50 GB disk (for a small lab). Splunk can run on modest hardware for testing, though production deployments recommend more (e.g. 12 CPU, 12GB+ RAM)​. The server will run the Splunk indexing and search head role.
- Splunk SOAR Server (Ubuntu 20.04 or RHEL 8 base): ~2 vCPU, 4 GB RAM, 50 GB disk. Splunk SOAR (formerly Phantom) is resource-intensive; Splunk recommends 16GB+ RAM for production use​, but you can use less in a low-volume lab. This VM will handle security automation playbooks.
- Windows 10 Client: 2 vCPU, 4 GB RAM, 30 GB disk. Used as a workstation endpoint with the Splunk Universal Forwarder and Sysmon installed for detailed Windows logs.
- Ubuntu Workstation (Ubuntu 20.04 Desktop): 2 vCPU, 4 GB RAM, 20 GB disk. Used as a Linux endpoint with Universal Forwarder, hosting an Apache web server (to generate access logs) and possibly UFW/iptables for firewall logging.
- Kali Linux Attacker: 2 vCPU, 2 GB RAM, 20 GB disk. Used to simulate attacks (port scans, brute-force attempts, web attacks) against the other machines. This machine typically does not run a forwarder since it’s an adversary simulator, but it shares the host-only network to interact with the targets.

# Network Layout
All VMs reside on a private virtual network (VMware host-only or NAT network) so they can communicate. The Splunk Enterprise server listens on port 9997 for incoming forwarded logs and on port 8000 for the Splunk Web interface. Splunk SOAR listens on its web port (default HTTPS port 9999). Ensure the VMs are on the same subnet and that any host firewalls allow the necessary ports (e.g., TCP 8000, 8089, 9997 on Splunk Enterprise; TCP 9999 on SOAR).

# Lab Architecture Summary
- Splunk Enterprise (SIEM) on Ubuntu Server – receives and indexes logs from all endpoints.
- Splunk SOAR on a separate server – ingests alerts from Splunk and executes automated response playbooks.
- Endpoints (Win10, Ubuntu workstation) – running Universal Forwarders to send logs (Windows Event Logs/Sysmon, Linux system logs, Apache logs, firewall logs) to Splunk.
- Attacker (Kali) – generates malicious activity (scans, attacks) that will be reflected in the endpoints’ logs and detected by Splunk.

Logs collected in this homelab include Windows Sysmon events, OS security events (for login attempts), firewall logs (Windows Firewall or UFW), and Apache web server access logs. These diverse data sources will showcase end-to-end detection and response: from data ingestion in Splunk to automated actions in SOAR.

# Installing Splunk Enterprise 9.4.1 on Ubuntu

1. Download Splunk: Sign in to Splunk’s download portal and get the Linux installer for Splunk Enterprise 9.4.1 (e.g., a .deb package for Ubuntu or a tar.gz). For example, you might download a file named splunk-9.4.1-<build>-Linux-x86_64.deb. (Note: You can use the free license for a lab, which allows indexing up to 500 MB of data per day. I was able to get the developer license in this case, so I can ingest up to 10GB of data daily)
   
2. Install the package: Transfer the installer to your Ubuntu server VM (via wget or via the host). Then run the installation. For a Debian-based system:

```
# Run as root or with sudo:
sudo dpkg -i splunk-9.4.1-*.deb
# If using a tar.gz, extract to /opt and then run the splunk binary from /opt/splunk
```
This will install Splunk into /opt/splunk. After installation, create an admin user and accept the license.

3. Start Splunk for the first time:

```
sudo /opt/splunk/bin/splunk start --accept-license
# You will be prompted to create an admin username & password
```

Splunk should now be running. Enable it to start on boot (optional but useful in a homelab such as mine):

```
sudo /opt/splunk/bin/splunk enable boot-start
```



