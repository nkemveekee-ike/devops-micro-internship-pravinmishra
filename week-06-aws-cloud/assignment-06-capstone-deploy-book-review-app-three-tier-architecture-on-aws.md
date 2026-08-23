# Assignment 6 — Capstone Assignment — Deploy Book Review App (Three-Tier Architecture) on AWS

Part of the DevOps Micro Internship (DMI) Cohort 3 with Agentic AI

---

## Purpose

This is the most important assignment of the course. You will deploy the Book Review App in a fully production-style three-tier architecture on AWS: a Next.js Web Tier behind Nginx and a public ALB, a private Node.js/Express App Tier behind an internal ALB, and a private Multi-AZ MySQL RDS database with a read replica. You are expected to design, deploy, isolate, debug, and document the result independently.

---

# Task 1 — Architecture Diagram

## Goal

Create an architecture diagram showing the custom VPC (10.0.0.0/16), the six subnets across two Availability Zones (two public Web Tier, two private App Tier, two private Database Tier), the public ALB, Web Tier EC2/Nginx, internal ALB, private App Tier EC2, private Multi-AZ RDS with its read replica, and the permitted traffic flow.

### Evidence

#### Diagram image or link

Add your diagram image or link here.

---

# Task 2 — AWS Region & Services Used

## Goal

Record the AWS Region used and list every AWS service used across networking, compute, load balancing, security, and the database.

### Notes

**Region:**

Region: eu-north-1 (Stockholm)

---

**Services:**

Category	Services
Networking	VPC, 6 Subnets (2 public / 4 private), Internet Gateway, NAT Gateway, Route Tables
Compute	EC2 — Web Tier (t3.small, Ubuntu 24.04), App Tier (t4g.micro, Ubuntu 24.04 arm64)
Load Balancing	2× Application Load Balancer — 1 public (Web Tier), 1 internal (App Tier)
Security	3× Security Groups (Web, App, DB tiers), least-privilege chained rules
Database	Amazon RDS for MySQL — primary instance + read replica
Process Management	pm2 (Node.js process manager, both tiers), systemd service registration
Web Server	Nginx (reverse proxy on Web Tier)

---

# Task 3 — Public Entry Point

## Goal

Confirm the Book Review App loads through the public ALB DNS name.

### Evidence

#### Public ALB DNS

Paste your public ALB DNS name here:

`Add your URL here`

---

# Task 4 — Evidence Screenshots

## Goal

Capture visual proof of every tier and load balancer.

### Evidence

#### Web EC2

Add your screenshot here.

---

#### App EC2

Add your screenshot here.

---

#### Public ALB

Add your screenshot here.

---

#### Internal ALB

Add your screenshot here.

---

#### RDS + Replica

Add your screenshot here.

---

#### App UI proof

Add your screenshot here.

---

# Task 5 — Summary

## Goal

Summarize what worked in the final deployment, the issues encountered and how each was fixed, and the tools or sources used to research and debug.

### Notes

**What worked:**

What worked:

The full three-tier architecture is live and serving real traffic end-to-end: a user hitting the public ALB is routed to an Nginx-fronted Next.js frontend in a public subnet, which reverse-proxies API calls through an internal ALB to a private Node.js/Express backend, which reads and writes to a MySQL RDS instance with a read replica — all isolated behind least-privilege security groups with no Elastic IPs anywhere in the design (the NAT Gateway's EIP is the only exception, and it's attached to the NAT Gateway itself, not to any instance). Both application tiers run under pm2, registered as systemd services so they survive reboots and are automatically restarted on failure. Both EC2 target groups report healthy in their respective ALBs.

---

**Issues + fixes:**

AWS "Free Plan" account restriction blocked Multi-AZ RDS. AWS's July 2025 account-tier change locks Multi-AZ and other higher-cost features behind a "Paid plan" upgrade requiring a verified payment method, which wasn't available. Deployed RDS as Single-AZ instead, and created a read replica separately (which the Free Plan did allow) to satisfy the read-scaling requirement. This is a genuine platform constraint, not a design shortcut — the architecture is written so that switching to Multi-AZ later is a single console setting once the account is upgraded.

Hit the Free Plan's RDS instance-count limit when creating bookreview-db, because a stopped RDS instance from an earlier, completed assignment was still counted against the quota. Deleted the unused instance to free the slot.

EC2 capacity errors across multiple instance types and both AZs (t3.micro, t3.small, t4g.micro, t4g.small all intermittently unavailable in eu-north-1a/eu-north-1b). Worked around by retrying different instance type/AZ combinations until each tier found available capacity; ended up on t3.small (x86) for the Web Tier and t4g.micro (arm64) for the App Tier.

Console defaulted to the wrong VPC and subnet repeatedly during EC2 launch (defaulting to an existing default VPC instead of bookreview-vpc, and to "No preference" instead of the intended subnet). Caught and corrected each time by verifying the Network settings section before every launch, rather than trusting the form's defaults.

Internal ALB was accidentally created in the wrong subnets (a web subnet and a DB subnet instead of the two App Tier subnets). Fixed by editing the ALB's subnet mappings after creation.
Security groups were missing rules that only became apparent when testing connectivity: the Web Tier's security group had zero inbound rules right after creation (blocking both SSH and HTTP), and the App Tier target group's health checks failed with "Request timed out" because the App Tier's security group only permitted traffic from the Web Tier's security group, not from the internal ALB's own security group (which the ALB and the App EC2 instance shared). Fixed by adding the missing HTTP/SSH rules and a self-referencing rule allowing the App Tier security group to talk to itself.

RDS connection failures from copy-paste template placeholders left in the backend's .env file (YOUR_RDS_ENDPOINT_ and YOUR_MASTER_USERNAME_ prefixes accidentally concatenated onto the real values). Diagnosed via the exact Sequelize/mysql2 error messages (ENOTFOUND, ER_ACCESS_DENIED_ERROR) and corrected the .env values.

ER_TOO_MANY_KEYS MySQL error after repeated backend restarts during troubleshooting: Sequelize's sync({ alter: true }) re-added a unique index on the Users.email column on every restart, eventually exceeding MySQL's 64-key-per-table limit. Fixed by dropping and recreating the 
database (safe, since only seed data existed) and avoiding unnecessary restarts afterward.

Nginx served stale DNS for the internal ALB after a config reload, causing intermittent 504 Gateway Timeout errors on /api/ even though the internal ALB itself was healthy and reachable. A full systemctl restart nginx (rather than reload) forced fresh DNS resolution and resolved it.

A double /api/api/ prefix in outgoing frontend requests caused persistent 404s on the homepage's book list, even though the equivalent calls in the app's shared api.js service module were correct. Traced to src/app/page.js hardcoding its own ${NEXT_PUBLIC_API_URL}/api/books call instead of reusing the api.js convention (which already expected NEXT_PUBLIC_API_URL to include /api). Fixed by aligning page.js with the rest of the app and rebuilding.

SSH access to the private App Tier instance was handled via SSH agent forwarding through the public Web Tier instance (ssh -A), so the private key never had to be copied onto either EC2 instance — keeping the App Tier genuinely unreachable from outside the VPC while still allowing administration.


---

**Tools/sources used:**

AWS Console (EC2, VPC, RDS, ALB/Target Groups, Security Groups), SSH (via Git Bash on Windows, with agent forwarding for the private App Tier), pm2 for process management, nano for remote file editing, the mysql CLI client for direct database troubleshooting, curl for isolating exactly which layer of the stack (Nginx, internal ALB, backend, RDS) a failure was happening at, browser DevTools Network/Console tabs for diagnosing the frontend request bug, and Claude for step-by-step guidance, error-message interpretation, and catching several configuration mistakes (wrong VPC/subnet defaults, wrong ALB subnets, missing security group rules) before they caused harder-to-diagnose problems later.

---

# LinkedIn Post (Required)

## Goal

Publish a LinkedIn post sharing the capstone deployment, including the public ALB DNS (or a redacted screenshot), three to five lines on what you built and why it is production-style, and one proof screenshot.

## Evidence

#### LinkedIn Post URL

Paste your LinkedIn post URL here:

`Add your URL here`

---

#### Screenshot of LinkedIn post

Add your screenshot here.

---

# Submission Instructions

- Add all required screenshots and links in your submission
- Do not expose passwords, RDS credentials, connection strings, private keys, or account IDs

---

# Completion Checklist

- [ ] Task 1: Architecture diagram completed
- [ ] Task 2: AWS Region and services documented
- [ ] Task 3: Public ALB DNS confirmed working
- [ ] Task 4: All six evidence screenshots captured (Web Tier, App Tier, both ALBs, RDS + replica, app UI)
- [ ] Task 5: Deployment summary completed (what worked, issues/fixes, tools/sources)
- [ ] LinkedIn post published and URL submitted
- [ ] App Tier and Database Tier confirmed not publicly accessible
- [ ] No sensitive data exposed

---

## 📌 About DMI & CloudAdvisory

DevOps Micro Internship (DMI) is a project-based DevOps program run by Pravin Mishra (The CloudAdvisory) focused on real-world execution, systems thinking, and career readiness.

It helps learners build strong DevOps foundations with hands-on experience.

---

## 📌 Resources

- 🌐 DMI Official Website: https://dmi.pravinmishra.com?utm_source=github&utm_medium=readme  
- 🎓 University: https://university.pravinmishra.com?utm_source=github&utm_medium=readme  
- 💬 Discord Community: https://discord.pravinmishra.com?utm_source=github&utm_medium=readme  
- 📝 Blog: https://dmi.pravinmishra.com/blog?utm_source=github&utm_medium=readme  
- ▶️ YouTube Playlist: https://www.youtube.com/playlist?list=PLFeSNDtI4Cho  
- 🔗 Pravin Mishra (LinkedIn): https://www.linkedin.com/in/pravin-mishra-aws-trainer/  
- 🏢 CloudAdvisory (LinkedIn): https://www.linkedin.com/company/thecloudadvisory/

---

*This submission is part of DevOps Micro Internship (DMI) Cohort 3 — Agentic AI Track.*