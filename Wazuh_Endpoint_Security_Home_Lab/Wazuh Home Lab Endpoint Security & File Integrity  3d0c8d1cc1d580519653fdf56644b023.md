# Wazuh Home Lab: Endpoint Security & File Integrity Monitoring

---

This project documents the setup of a **Wazuh home lab** for endpoint security monitoring using a Wazuh server and a Kali Linux endpoint.

The lab covers:

- Wazuh server deployment
- Kali Linux agent deployment
- File Integrity Monitoring (FIM)
- Custom Wazuh detection rules
- Real-time file monitoring
- Linux service troubleshooting
- Wazuh API/startup troubleshooting
- SOC-style endpoint monitoring and investigation

---

# 1. Lab Architecture

The lab consists of two virtual machines:

```
                 ┌──────────────────────┐
                 │     Host Machine     │
                 │      VirtualBox      │
                 └──────────┬───────────┘
                            │
                     Bridged Network
                            │
              ┌─────────────┴─────────────┐
              │                           │
      ┌───────▼────────┐         ┌────────▼────────┐
      │  Wazuh Server  │         │   Kali Linux    │
      │                │         │                 │
      │ Wazuh Manager  │◄────────│ Wazuh Agent     │
      │ Wazuh Indexer  │         │                 │
      │ Wazuh Dashboard│         │ /etc/cron.d     │
      └────────────────┘         └─────────────────┘
              │
              ▼
        Security Alerts
              │
              ▼
       Wazuh Dashboard
```

For a home lab, a **Bridged Adapter** is convenient because the Wazuh VM and other lab machines can communicate over the same local network.

---

# 2. Wazuh VM Setup

Install the Wazuh Virtual Machine in VirtualBox and start it.

Log in using the credentials configured for the VM:

```
Username: wazuh-user
Password: wazuh
```

Switch to the root account:

```bash
su
```

## Enable Wazuh Services

Enable the Wazuh Manager, Indexer, and Dashboard to start automatically at boot:

```bash
systemctl enable wazuh-manager
systemctl enable wazuh-indexer
systemctl enable wazuh-dashboard
```

Verify that they are enabled:

```bash
systemctl is-enabled wazuh-manager
systemctl is-enabled wazuh-indexer
systemctl is-enabled wazuh-dashboard
```

Expected output for each:

```
enabled
```

Then reboot:

```bash
reboot
```

After the VM starts, you can verify the services with:

```bash
systemctl status wazuh-manager
systemctl status wazuh-indexer
systemctl status wazuh-dashboard
```

Wazuh's documentation similarly recommends enabling and starting the Manager service through systemd and verifying its status. ([Wazuh Documentation](https://documentation.wazuh.com/current/installation-guide/wazuh-server/step-by-step.html?utm_source=chatgpt.com))

---

# 3. Accessing the Wazuh Dashboard

After the Wazuh VM boots, find its IP address:

```bash
ip a
```

Identify the IP address assigned to the VM.

For example:

```
192.168.31.176
```

Open the Wazuh Dashboard from your host machine:

```
https://<WAZUH-IP>
```

For example:

```
https://192.168.31.176
```

The Wazuh Dashboard uses HTTPS on port **443**.

> Your IP address will be different depending on your network configuration.
> 

---

# 4. Troubleshooting Wazuh API and Startup Issues

One of the issues encountered during this home lab was a **Wazuh API error caused by the Wazuh Manager not successfully starting**.

When the dashboard reports API-related problems, first check the status of all major Wazuh components:

```bash
systemctl status wazuh-manager
systemctl status wazuh-indexer
systemctl status wazuh-dashboard
```

Pay particular attention to:

```bash
systemctl status wazuh-manager
```

A healthy service should show:

```
Active: active (running)
```

If it shows:

```
Active: failed
```

or fails to start correctly, investigate the Manager first.

---

## 4.1 Increasing the Wazuh Manager Startup Timeout

One solution used during this lab was to increase the amount of time systemd allows the Wazuh Manager to start.

Run:

```bash
systemctl edit wazuh-manager
```

Add:

```
[Service]
TimeoutStartSec=120
```

Save and exit.

Then reload systemd:

```bash
systemctl daemon-reload
```

Restart the Wazuh Manager:

```bash
systemctl restart wazuh-manager
```

Verify:

```bash
systemctl status wazuh-manager
```

### What `TimeoutStartSec` do?

`TimeoutStartSec` controls how long systemd waits for a service to complete its startup process before treating the startup as failed.

Increasing this value can help when the Wazuh Manager takes longer than the default timeout to initialize.

---

# 5. Making the Startup Timeout Explicitly Persistent

if after shutdown , and restarting the wazuh vm cause similar api issues then in that , chances are high that the value resets again to the dafault that we have setted to 120 sec in that case use this persistent method : 

Check whether the directory exists:

```bash
sudo ls -la /etc/systemd/system/wazuh-manager.service.d/
```

If it doesn't exist:

```bash
sudo mkdir -p /etc/systemd/system/wazuh-manager.service.d
```

Create the override file:

```bash
sudo nano /etc/systemd/system/wazuh-manager.service.d/override.conf
```

Add:

```
[Service]
TimeoutStartSec=300
```

Save and exit.

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Enable the service:

```bash
sudo systemctl enable wazuh-manager
```

Restart it:

```bash
sudo systemctl restart wazuh-manager
```

Verify the configured timeout:

```bash
systemctl show wazuh-manager -p TimeoutStartUSec
```

Expected output:

```
TimeoutStartUSec=5min
```

### Note

`systemctl edit wazuh-manager` normally creates a systemd drop-in override itself. Therefore, **you do not need to perform both methods**. The manual `override.conf` method is useful when you want to explicitly inspect or manage the drop-in configuration.

---

# 6. Adding Kali Linux as a Wazuh Agent

Once the Wazuh Dashboard is accessible:

1. Open **Endpoint Security**.
2. Go to **Agents**.
3. Select **Deploy/New Agent**.
4. Select the appropriate operating system.
5. Enter the Wazuh Manager's IP address.
6. Specify the desired agent name/group.

The dashboard provides an installation command based on your selections.

For this lab, the following command was used:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.7-1_amd64.deb && sudo WAZUH_MANAGER='192.168.31.176' WAZUH_AGENT_GROUP='default' WAZUH_AGENT_NAME='kali_attacker' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb
```

Replace:

```
192.168.31.176
```

with the actual IP address of your Wazuh Manager.

Wazuh currently lists the `wazuh-agent_4.14.7-1_amd64.deb` package for Debian/Ubuntu-based systems. ([Wazuh Documentation](https://documentation.wazuh.com/current/installation-guide/packages-list.html?utm_source=chatgpt.com))

---

## 6.1 Enable and Start the Agent

On Kali Linux:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Verify:

```bash
sudo systemctl status wazuh-agent
```

Expected status:

```
Active: active (running)
```

These systemd steps are also part of Wazuh's documented Linux agent deployment process. ([Wazuh Documentation](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html?utm_source=chatgpt.com))

---

# 7. Creating a Custom FIM Detection Rule

For this lab, the `/etc/cron.d/` directory is monitored.

This is useful from a security perspective because cron configuration can be abused for **persistence** on Linux systems.

On the Wazuh Manager, edit:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add the custom rules **inside a `<group>` element**.

For example:

```xml
<group name="syscheck,">

    <rule id="100101" level="10">
        <if_sid>550</if_sid>
        <field name="file">/etc/cron.d/</field>
        <description>FIM: File changed in /etc/cron.d</description>
    </rule>

    <rule id="100102" level="10">
        <if_sid>554</if_sid>
        <field name="file">/etc/cron.d/</field>
        <description>FIM: File added to /etc/cron.d</description>
    </rule>

</group>
```

Wazuh documents rule `550` for file modification events and rule `554` for file creation events in FIM-based detection examples. ([Wazuh Documentation](https://documentation.wazuh.com/current/proof-of-concept-guide/detect-remove-malware-virustotal.html?utm_source=chatgpt.com))

Wazuh also recommends using custom rule IDs in the `100000–120000` range to avoid conflicts with built-in rules. ([Wazuh Documentation](https://documentation.wazuh.com/current/user-manual/ruleset/rules/custom.html?utm_source=chatgpt.com))

---

# 8. Validate and Apply the Custom Rules

Before relying on a custom rule, it is good practice to test it using Wazuh's rule-testing utility:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

This tool allows you to test how Wazuh decodes events and which rules match them. ([Wazuh Documentation](https://documentation.wazuh.com/current/user-manual/reference/tools/wazuh-logtest.html?utm_source=chatgpt.com))

After saving your rule configuration, restart the **Wazuh Manager**:

```bash
sudo systemctl restart wazuh-manager
```

Then verify:

```bash
sudo systemctl status wazuh-manager
```

> **Important:** Changing `local_rules.xml` is a Wazuh Manager-side configuration change. Restarting the Kali agent alone does not apply the new Manager rules. ([Wazuh Documentation](https://documentation.wazuh.com/current/user-manual/ruleset/rules/custom.html?utm_source=chatgpt.com))
> 

---

# 9. Configuring Real-Time FIM on Kali

The FIM ( file integrity monitoring ) configuration itself is performed on the **endpoint being monitored**.

On Kali Linux, edit:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Locate the `<syscheck>` section.

A relevant configuration can look like:

```xml
<syscheck>
    <disabled>no</disabled>

    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>

    <directories realtime="yes" report_changes="yes">/etc/cron.d</directories>
</syscheck>
```

The important line is:

```xml
<directories realtime="yes" report_changes="yes">/etc/cron.d</directories>
```

### What do these options do?

`realtime="yes"` enables continuous monitoring of the directory on Linux using filesystem notification mechanisms.

`report_changes="yes"` enables reporting of changes to supported text files.

Wazuh documents both options for FIM configuration. ([Wazuh Documentation](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/syscheck.html?utm_source=chatgpt.com))

Restart the Kali agent:

```bash
sudo systemctl restart wazuh-agent
```

Verify:

```bash
sudo systemctl status wazuh-agent
```

---

# 10. Testing File Integrity Monitoring

Now test the configuration by creating a file inside `/etc/cron.d/`.

```bash
sudo touch /etc/cron.d/wazuh-fim-test
```

Then modify it:

```bash
echo "# Wazuh FIM test" | sudo tee -a /etc/cron.d/wazuh-fim-test
```

The command appends:

```
# Wazuh FIM test
```

to:

```
/etc/cron.d/wazuh-fim-test
```

Because `/etc/cron.d/` is configured for real-time FIM, Wazuh should detect the filesystem activity.

---

# 11. Viewing the Alert

Open the Wazuh Dashboard and navigate to the FIM events.

Depending on the dashboard version/layout, the event can be investigated through the endpoint/File Integrity Monitoring views.

The expected detection flow is:

```
File created/modified
        ↓
Wazuh Agent
        ↓
FIM / Syscheck
        ↓
Wazuh Manager
        ↓
Custom Rule
        ↓
Security Alert
        ↓
Wazuh Dashboard
```

---

# 12. Clipboard/Paste Issue in the Wazuh VM

During the lab, clipboard sharing between the host and the Wazuh VM may not work reliably.

This becomes especially inconvenient when entering long XML configurations.

One alternative is to connect to the Wazuh VM through SSH from the host machine.

From the host:

```bash
ssh wazuh-user@<WAZUH-IP>
```

For example:

```bash
ssh wazuh-user@192.168.31.176
```

Once connected, commands and configuration can be pasted through the host's terminal instead of relying on the VirtualBox console clipboard.

---

# 13. Troubleshooting Checklist

If the Wazuh Dashboard shows API errors, use the following sequence.

### Check the Wazuh Manager

```bash
systemctl status wazuh-manager
```

### Check the Indexer

```bash
systemctl status wazuh-indexer
```

### Check the Dashboard

```bash
systemctl status wazuh-dashboard
```

### Check whether services are enabled

```bash
systemctl is-enabled wazuh-manager
systemctl is-enabled wazuh-indexer
systemctl is-enabled wazuh-dashboard
```

### Check the Wazuh IP

```bash
ip a
```

### If Manager startup is timing out

Create/edit the systemd override:

```bash
sudo systemctl edit wazuh-manager
```

Add:

```
[Service]
TimeoutStartSec=300
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart wazuh-manager
```

Verify:

```bash
systemctl show wazuh-manager -p TimeoutStartUSec
```

Expected:

```
TimeoutStartUSec=5min
```

---

# 14. Security Scenario

The FIM configuration can be used to simulate a basic Linux persistence scenario.

For example:

```
        Attacker
           │
           ▼
Modifies /etc/cron.d
           │
           ▼
   Wazuh Agent detects
    filesystem change
           │
           ▼
      FIM / Syscheck
           │
           ▼
    Wazuh Manager
           │
           ▼
    Custom Rule Match
           │
           ▼
       Alert Generated
           │
           ▼
    SOC Analyst Reviews
        the Alert
```

This is a simple but realistic example of how an endpoint monitoring platform can help detect potentially suspicious changes to persistence-related configuration.

---

# 15. What This Home Lab Demonstrates

This lab provides practical experience with:

- **Wazuh SIEM/XDR capabilities**
- Endpoint agent deployment
- Linux endpoint monitoring
- File Integrity Monitoring
- Real-time filesystem monitoring
- Custom detection rules
- Rule severity configuration
- Monitoring security-sensitive directories
- Detection of file creation
- Detection of file modification
- Linux systemd service management
- Wazuh Manager troubleshooting
- API/startup troubleshooting
- Security alert investigation
- Basic Linux persistence detection

---

# 16. Key Takeaways

The main objective of this lab was not simply to install Wazuh, but to understand the complete detection workflow:

```
Endpoint
   ↓
Wazuh Agent
   ↓
FIM
   ↓
Wazuh Manager
   ↓
Detection Rule
   ↓
Alert
   ↓
SOC Investigation
```

By monitoring `/etc/cron.d/`, the lab demonstrates how a SOC analyst can detect potentially suspicious changes to Linux persistence mechanisms and investigate those events through a centralized security monitoring platform.

**Official Wazuh documentation:** [Wazuh Documentation](https://documentation.wazuh.com/current/?utm_source=chatgpt.com)