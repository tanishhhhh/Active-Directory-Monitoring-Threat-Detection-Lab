

-----

# Active Directory Monitoring & Alerting Lab

  

## 📖 Project Overview

This project implements a comprehensive monitoring solution for an **Active Directory (AD)** environment using open-source tools. It provides real-time visibility into system performance, security events, and user activity, with automated alerting for suspicious behavior (e.g., brute force attacks).

### 🎯 Key Features

  * **Real-time Monitoring:** CPU, Memory, Disk, and Network traffic visualization.
  * **Security Auditing:** Detection of successful and failed user logins.
  * **Automated Alerting:** Email alerts triggered by high CPU usage or frequent login attempts.
  * **Dashboards:** Rich visualization of metrics using Grafana.

-----

## 🏗️ Architecture

The lab consists of the following components running on **Windows Server 2022**:

1.  **Windows Server 2022:** The Domain Controller (Target).
2.  **Windows Exporter:** Collects metrics from the OS and Active Directory.
3.  **Prometheus:** The time-series database that scrapes metrics from the exporter.
4.  **Alertmanager:** Handles alerts sent by Prometheus (grouping, inhibition, notification).
5.  **Grafana:** Connects to Prometheus to visualize data in dashboards.

-----

## ⚙️ Prerequisites

  * Virtualization Software (VMware Workstation or VirtualBox)
  * Windows Server 2022 ISO (Evaluation Edition)
  * Basic knowledge of PowerShell and YAML syntax.

-----

## 🚀 Installation & Configuration

### 1\. Windows Server Setup

  * Install **Windows Server 2022 Standard (Desktop Experience)**.
  * Install the **Active Directory Domain Services (AD DS)** role.
  * Promote the server to a **Domain Controller**.
  * **Crucial:** Enable "Audit Logon Events" in Group Policy (`secpol.msc` -\> Local Policies -\> Audit Policy) for both Success and Failure.

### 2\. Windows Exporter

Download the latest release of `windows_exporter`. Run the following command in **Administrator PowerShell** to enable the required collectors (AD and Logon):

```powershell
# Note: Use quotes to prevent parsing errors with the comma-separated list
.\windows_exporter-0.31.3-amd64.exe --collectors.enabled "ad,cpu,memory,net,os,service,system,logon"
```

  * *Port:* `9182`

### 3\. Prometheus Configuration (`prometheus.yml`)

Configure Prometheus to scrape data from the local exporter.

```yaml
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]

rule_files:
  - "alert.rules"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  
  - job_name: "windows_ad"
    static_configs:
      - targets: ["localhost:9182"]
```

### 4\. Alert Rules (`alert.rules`)

Create a file named `alert.rules` in the Prometheus directory to define security triggers.

```yaml
groups:
  - name: AD_Logon_Events
    rules:
      - alert: HighLogonEvents
        # Trigger if more than 2 interactive logins occur in 2 minutes
        expr: increase(windows_logon_logon_type_count{logon_type="2"}[2m]) > 2
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High number of logon events"
          description: "Multiple interactive logins detected."
```

### 5\. Alertmanager Setup (`alertmanager.yml`)

Configure the notification receiver (Email).

```yaml
global:
  resolve_timeout: 5m
route:
  receiver: 'email-alert'
receivers:
  - name: 'email-alert'
    email_configs:
      - to: 'admin@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'password'
```

-----

## 📊 Visualizing in Grafana

1.  Install Grafana and login (`admin`/`admin`).
2.  Add **Prometheus** as a Data Source (`http://localhost:9090`).
3.  Import a Dashboard (Recommended ID: `14510` for Windows Exporter) or create custom panels.

-----

## 🧪 Testing the Alerts

To verify the **HighLogonEvents** alert:

1.  Ensure all services (Exporter, Prometheus, Alertmanager) are running.
2.  Lock the Windows VM screen (`Windows Key + L`).
3.  Unlock and log in.
4.  **Repeat this process 5 times rapidly.**
5.  Wait approx. 60 seconds.
6.  Check `http://localhost:9090/alerts` to see the alert switch from **PENDING** to **FIRING** (Red).

-----

## 🛠️ Troubleshooting

  * **Exporter Crashing?** Ensure there are **no spaces** in the collector list flag. Use quotes around the list: `--collectors.enabled "..."`.
  * **Alerts not Firing?** Verify you are monitoring `windows_logon_logon_type_count` instead of raw Event IDs, as the standard exporter simplifies these metrics.
  * **No Data in Grafana?** Check that the Prometheus target status is **UP** at `http://localhost:9090/targets`.

-----

## 📜 License

This project is for educational purposes.

-----

