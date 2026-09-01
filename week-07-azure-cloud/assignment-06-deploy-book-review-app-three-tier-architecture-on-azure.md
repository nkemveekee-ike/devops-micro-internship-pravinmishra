# Assignment 6 — Capstone: Deploy Book Review App (Three-Tier Architecture) on Azure

Part of the DevOps Micro Internship (DMI) Cohort 3 with Agentic AI

---

## Purpose

This is the most important assignment of the course. You will deploy the Book Review App in a production-ready, best-practice-compliant three-tier architecture on Azure: separated presentation, application, and database tiers, least-privilege network access, a controlled public entry point, protected secrets, and availability/monitoring evidence.

---

# Task 1 — Design the Azure Three-Tier Architecture

## Goal

Create an architecture diagram and implementation plan identifying the presentation, application, and database components, the chosen Azure services, the public entry point, and the internal traffic paths.

### Evidence

#### Screenshot 1 — Architecture diagram showing the public entry point, three tiers, network boundaries, and traffic flow

![screenshots](screenshots/A6-T1-S1.png)

---

#### Screenshot 2 — Written architecture assumptions and selected Azure services

![screenshots](screenshots/A6-T1-S2.png)

---

# Task 2 — Create the Azure Network Foundation

## Goal

Create a dedicated Resource Group and VNet with separate subnets for the web, application, and database tiers, keeping the application and database tiers without direct public access.

### Evidence

#### Screenshot 3 — Resource Group overview showing the assignment resources

![screenshots](screenshots/A6-T2-S1.png)

---

#### Screenshot 4 — VNet overview showing the address space and all required subnets

![screenshots](screenshots/A6-T2-S2.png)

---

#### Screenshot 5 — Route-table or Private DNS evidence where applicable

![screenshots](screenshots/A6-T2)

---

# Task 3 — Configure Security and Secret Management

## Goal

Apply least-privilege NSG rules so traffic flows Internet → public entry point → web tier → application tier → database tier, and store credentials in Azure Key Vault or another approved secure mechanism.

### Evidence

#### Screenshot 6 — NSG rules proving least-privilege access between the tiers

![screenshots](screenshots/A6-T3-S1.png)

---

#### Screenshot 7 — Key Vault or approved secret-management configuration (without displaying secret values)

![screenshots](screenshots/A6-T3-S2.png)

---

# Task 4 — Deploy the Presentation (Web) Tier

## Goal

Deploy the Book Review App presentation layer on the approved web-tier compute service, configured to route requests to the internal application-tier endpoint, and not directly exposed except through the public entry service.

### Evidence

#### Screenshot 8 — Web-tier compute overview showing subnet and availability configuration

![screenshots](screenshots/A6-T4-S1.png)

---

#### Screenshot 9 — Terminal or service output proving the presentation layer is running

![screenshots](screenshots/A6-T4-S22.png)

---

# Task 5 — Deploy the Business (Application) Tier

## Goal

Deploy the Book Review App backend privately in the application subnet, configured to use the private database endpoint and secured environment values, reachable only through its internal endpoint.

### Evidence

#### Screenshot 10 — Application-tier compute overview showing private subnet placement

![screenshots](screenshots/A6-T5-S1.png)

---

#### Screenshot 11 — Backend process, service, or listening-port evidence

![screenshots](screenshots/A6-T5-S2.png)

---

#### Screenshot 12 — Internal health-check or API response (without exposing secrets)

![screenshots](screenshots/A6-T5-S3.png)

---

# Task 6 — Deploy the Managed Database Tier

## Goal

Create a private Azure managed database (public access disabled), with availability/backup/retention settings, the Book Review App schema imported, and access restricted to the application tier only.

### Evidence

#### Screenshot 13 — Database overview showing private connectivity and public access disabled

![screenshots](screenshots/A6-T6-S1.png)

---

#### Screenshot 14 — Availability, backup, and retention configuration

![screenshots](screenshots/A6-T6-S2.png)

---

#### Screenshot 15 — Successful schema or connectivity verification (without exposing credentials)

![screenshots](screenshots/A6-T6-S3.png)

---

# Task 7 — Configure Traffic Management, Availability, and Monitoring

## Goal

Configure the approved public entry service with health probes and backend pools, internal routing for the application tier where required, and enable Azure Monitor/diagnostics/logs/alerts for the key resources.

### Evidence

#### Screenshot 16 — Public entry service showing listener, frontend endpoint, and healthy web targets

![screenshots](screenshots/A6-T7-S1.png)

---

#### Screenshot 17 — Internal application-tier load-balancing or routing configuration where applicable

![screenshots](screenshots/A6-T7-S2.png)

---

#### Screenshot 18 — Azure Monitor, diagnostic settings, logs, metrics, or alert evidence

![screenshots](screenshots/A6-T7-S3.png)

---

# Task 8 — Validate the Production-Style Deployment

## Goal

Confirm the Book Review App works end to end through the public endpoint, with at least one database read and one write, confirm private tiers are not internet-reachable, and complete a safe availability test.

### Evidence

#### Screenshot 19 — Browser showing the Book Review App through the public endpoint

![screenshots](screenshots/A6-T8-S1.png)

---

#### Screenshot 20 — Proof of successful database-backed read and write operations

![screenshots](screenshots/A6-T8-S2.png)

---

#### Screenshot 21 — Evidence that private tiers are not publicly accessible

![screenshots](screenshots/A6-T8-S3.png)

---

#### Screenshot 22 — Availability-test and healthy-target evidence

![screenshots](screenshots/A6-T8-S4.png)

---

#### Public Endpoint

Paste your public endpoint URL here:

http://20.164.96.156

---

### Notes

Summarize what worked, issues encountered and how they were fixed, and the availability/security/secrets/monitoring/backup choices made.

The app's up and running as a proper three-tier setup. Web tier (VMSS + Nginx) sits in a public subnet and proxies to the app tier (VMSS + Node/Express), which is fully private and only reachable through an internal load balancer. Database is a private MySQL Flexible Server that only the app tier can talk to. All public traffic comes in through an Application Gateway — nothing else is exposed.

Issues I ran into:

Kept hitting a 4-vCPU regional quota limit that blocked VM creation — had to juggle sizes and free up quota a few times.
App subnet had no internet access at first, so it couldn't install anything. Added a NAT Gateway temporarily, removed it once installs were done.
Azure MySQL forces SSL by default, and my app kept failing to connect — turned out to be one line in models/index.js not passing the SSL config through to Sequelize. Fixed it.
Internal load balancer and App Gateway both showed empty backend pools even after adding my VMSS — had to manually upgrade the VMSS instances for it to actually take effect. Tripped me up twice.
Left the Node app running in the foreground over SSH and it died when I disconnected. Restarted with nohup so it stays up.
Ran out of public IPs (limit of 3) creating the App Gateway's IP — deleted an old unused one to free up room.

Availability/security/secrets/monitoring/backup:
App Gateway autoscales 1-2 instances. Web and app tiers are VMSS but stuck at 1 instance each due to the quota — a real constraint, not a design choice. NSGs only allow traffic where needed (web→app on the app port, app→db on 3306); neither app tier nor database has a public IP; only SSH path is through Bastion. DB connection string is an env var, not hardcoded; managed identity is set up on the VMSS for future Key Vault use. Everything logs to one Log Analytics workspace — App Gateway logs/metrics, VM Insights on both tiers, full MySQL diagnostics. MySQL backups are 7-day retention with local + geo-redundant.

---

# Submission Instructions

- Add all required screenshots and links in your submission
- Do not expose passwords, keys, connection strings, or subscription IDs

---

# Completion Checklist

- [X] Task 1: Architecture diagram and assumptions documented (Screenshots 1–2)
- [X] Task 2: Network foundation created with isolated tiers (Screenshots 3–5)
- [X] Task 3: Least-privilege security and secret management configured (Screenshots 6–7)
- [X] Task 4: Presentation tier deployed (Screenshots 8–9)
- [X] Task 5: Application tier deployed privately (Screenshots 10–12)
- [X] Task 6: Managed database tier deployed privately (Screenshots 13–15)
- [X] Task 7: Public entry, internal routing, and monitoring configured (Screenshots 16–18)
- [X] Task 8: End-to-end validation and availability test completed (Screenshots 19–22, Public Endpoint, Notes)
- [X] No sensitive data exposed

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
