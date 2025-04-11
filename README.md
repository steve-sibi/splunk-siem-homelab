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

4. Configure Splunk to receive data from forwarders: In Splunk Web (navigate to `http://<splunk-server-ip>:8000` and log in as admin), go to Settings > Forwarding and Receiving > Configure Receiving. Add a new port `9997` (the default port that forwarders send data to). This opens port 9997 on the Splunk server to listen for incoming Universal Forwarder data. Ensure the Ubuntu firewall (ufw) allows this port if enabled.

5. Create indexes (optional but recommended): For organization, create separate indexes for different log types. In Splunk Web, go to Settings > Indexes and add indexes such as:
   
- `windows` (for general Windows Event logs),
- `sysmon` (for Sysmon events specifically),
- `linux` (for Linux system logs),
- `web` (for Apache/web server logs),
- `firewall` (for firewall logs).

If you prefer, you can use the default `main` index for everything, but separate indexes help manage and search data more efficiently. 

6. Verify Splunk is ready: You should be able to access the Splunk GUI on port 8000 and Splunk should be waiting for data on port 9997. At this point, Splunk Enterprise is set up as our central SIEM platform.

# Installing Splunk SOAR 6.4.0 on a VM

Splunk SOAR (formerly known as Phantom) will handle orchestration and automation. We install it on a separate Linux VM. 

1. Obtain Splunk SOAR: Download the Splunk SOAR 6.4.0 on-premises installer from Splunk (requires a Splunk account). Splunk SOAR is available as an RPM/tar package for Red Hat-based systems, and can also be installed on Ubuntu 20.04 with a tar installer. Alternatively, Splunk may provide a pre-built OVA for easy deployment.

2. Prepare the OS: Use a supported OS (RHEL/CentOS 8 or Ubuntu 20.04 LTS). Ensure the VM has at least a few GB of RAM (8 GB or more). Update the OS packages and set a hostname. If using Ubuntu, you may need to install dependencies like `docker` (Splunk SOAR uses Docker containers under the hood).

3. Install Splunk SOAR - Follow the official install steps:
- For RHEL/CentOS: create a `phantom` user, untar the Splunk SOAR installer to `/opt/phantom`, then run the installer script (`./soar-install`). For Ubuntu, a similar process is followed with an installer script.
- During installation, default ports will be set up (Splunk SOAR uses ports 9999 for HTTPS web UI by default, and others for services – make sure to allow them).
- The installer will prompt for some settings (you can accept defaults for a lab). Wait for the installation to complete (it can take 10-15 minutes as it sets up containers).

4. Initial Configuration - Once installed, start the Splunk SOAR service:

```
sudo /opt/phantom/bin/phantom start
```
Open a browser to `https://<soar-server-ip>:9999`. You should see the Splunk SOAR login page. Log in with the default credentials or the ones you set during installation (the default might be `admin` / `password` if not prompted to change).

5. Set admin account and networking - In the SOAR UI, create your admin user (if not already done) and adjust any settings (for example, SMTP for email if you plan to send email alerts, or integrating Slack API if you plan Slack alerts, etc.). In Administration, you can generate an Auth Token for API access (needed when Splunk Enterprise sends events to SOAR).

6. Integrate Splunk Enterprise with SOAR - To enable end-to-end alerting, configure Splunk and SOAR to talk:
- Install the Splunk App for SOAR Export on the Splunk Enterprise server (available on Splunkbase). This app allows Splunk to send notable events or search results to Splunk SOAR.
- In Splunk SOAR, add a new Asset for Splunk (under Administration > Integrations). This involves providing the Splunk server’s address, port (8089 for Splunk’s API), and credentials or token. Set the asset to ingest events (so that SOAR will fetch or receive alerts from Splunk)​.
- (Alternatively, you can configure Splunk SOAR to poll Splunk for alerts or use the app’s alert action to push events to SOAR. In our lab, we assume the app and asset method for simplicity.)

After this integration, any alerts triggered in Splunk (via saved searches or correlation searches) can be sent to Splunk SOAR for automated response. We will create example alerts and playbooks in later sections.




