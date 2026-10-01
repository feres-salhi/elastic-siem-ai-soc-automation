# SIEM + AI Automation Lab — Lab Notes

## Goal
Build a mini SOC: Windows server on AWS sends logs to Elastic SIEM,
suspicious activity creates an alert, Tines sends it to an AI for a summary,
and an analyst gets an email.

## Setup
- Start date: 2026-09-28
- Cloud: AWS (Free plan), region Frankfurt (eu-central-1)
- Server: Windows Server 2025, c7i-flex.large (2 vCPU, 4 GB RAM)
- SIEM: Elastic Security Serverless (14-day trial, started 2026-09-29, ends ~13 Oct)
- Automation: Tines Stories (free), story "Elastic Alert AI Triage"

## Log
- Created project folder and notes.
- Created AWS account (Free plan), region set to Frankfurt (eu-central-1).
- Enabled MFA on root account (authenticator app).
- Created zero-spend budget alert (email notification).
- Launched Windows Server 2025 (c7i-flex.large) in Frankfurt (eu-central-1b).
- Security group siem-victim-sg: RDP (3389) allowed only from my IP (/32).
- Connected to the server via RDP as Administrator (password decrypted with key pair).
- Created Elastic Security serverless project in AWS Frankfurt (eu-central-1), Security Complete tier.
- Added Elastic Defend integration "defend-windows" (Complete EDR) with agent policy
  "windows-victim-policy" (includes System integration for Windows event logs).
- Installed Elastic Agent 9.5.4 on the server (PowerShell as Administrator, Fleet enrollment token). Enrollment successful.
- Fleet shows the agent on EC2AMAZ-91TNAJP as Healthy (policy windows-victim-policy).
- Verified in Discover: ~1,000 events in 15 min from host ec2amaz-91tnajp (Endpoint process events), spike at agent install time.
- Created custom detection rule "Whoami Execution - User Discovery"
  (KQL: process.name : "whoami.exe", severity Medium, runs every 1 min, MITRE T1033). Status: Succeeded.
- Ran whoami on the server (21:59 UTC on 30 Sep). Rule fired: 2 Medium alerts.
- Finding: 1 execution = 2 alerts, because Elastic Defend logs both process start and end events.
- Built Tines story "Elastic Alert AI Triage": Webhook → AI Agent (Task, no tools,
  system instructions separated from alert data) → Send Email. Moved story to team so the webhook works 24/7.
- Tested with a fake alert via PowerShell (Invoke-RestMethod). Email with AI summary arrived in ~1 min;
  AI mapped T1033 and flagged the test alert as likely benign (because of the "TEST" note field).
- Added Webhook connector "Tines - Elastic Alert AI Triage" (POST, Content-Type: application/json)
  as rule action: summary of alerts, per rule run. Body sends rule name, severity, alert count,
  host/user/process per alert and a link back to Elastic.
- END-TO-END SUCCESS (01 Oct, 00:07 UTC): whoami on server → Elastic rule → webhook → Tines AI → email,
  fully automatic. AI identified host, user, time, T1033 and suggested checking parent process and command line.

## Problems & fixes
- MFA setup failed with "Authentication code for device is not valid".
  Cause: old QR entries in the authenticator app / codes are time-based.
  Fix: deleted old entries, restarted setup, scanned the new QR once,
  entered two consecutive codes quickly.
- Instance was first launched in us-east-1 (N. Virginia) by mistake:
  the console switched region when opening EC2.
  Fix: terminated it, created a new key pair and relaunched in Frankfurt.
  Lesson: resources, key pairs and security groups are per region;
  always check the region before creating anything.
- RDP failed the next day ("Remote Desktop can't connect").
  Cause: my home IP changed, so the security group blocked me.
  Fix: updated the inbound RDP rule to "My IP" again.
- RDP failed after restarting the server.
  Cause: a stopped/started instance gets a new public address; I used an old address.
  Fix: downloaded a fresh RDP file / used the current Public IPv4 DNS.
- Alerts first not visible: time filter was "Today" and the alert was from before midnight.
  Fix: changed the time range to "Last 24 hours".

## Improvements (ideas)
- Match only process start: process.name : "whoami.exe" and event.type : "start"
- Include the Elastic alert link directly in the email
- Ask the AI for plain text (no Markdown)
- Test the triage AI against prompt injection via alert fields
