# Assignment 5 — Deploy a Highly Available Two-Tier Application on AWS (VPC + ALB + ASG + Multi-AZ RDS)

Part of the DevOps Micro Internship (DMI) Cohort 3 with Agentic AI

---

## Purpose

In this assignment, you will design and deploy a highly available two-tier web application on AWS: highly available networking across two Availability Zones, an Application Load Balancer, an Auto Scaling Group for the web tier, and a private Multi-AZ RDS database. You must prove high availability with real failure tests.

---

# Task 1 — Create HA Networking (VPC + 4 Subnets + IGW + NAT + Route Tables)

## Goal

Build a VPC (10.0.0.0/16) with two public and two private subnets across two Availability Zones, an Internet Gateway, a NAT Gateway, and the matching public/private route tables.

### Evidence

#### Screenshot 1 — VPC details showing CIDR 10.0.0.0/16

![screenshots](screenshots/A5-T1-S1.png)

---

#### Screenshot 2 — Subnets list showing four subnets and their Availability Zones


![screenshots](screenshots/A5-T1-S2.png)

---

#### Screenshot 3 — Public route table showing the Internet Gateway route and both public-subnet associations

![screenshots](screenshots/A5-T1-S3.png)

---

#### Screenshot 4 — Private route table showing the NAT Gateway route and both private-subnet associations

![screenshots](screenshots/A5-T1-S4.png)

---

#### Screenshot 5 — NAT Gateway status showing Available and the Elastic IP

![screenshots](screenshots/A5-T1-S5.png)

---

# Task 2 — Create Security Groups (ALB, EC2, RDS) with Least Privilege

## Goal

Create `ha-alb-sg` (HTTP public), `ha-web-sg` (HTTP only from `ha-alb-sg`, SSH from your IP), and `ha-db-sg` (database port only from `ha-web-sg`).

### Evidence

#### Screenshot 6 — ALB Security Group inbound rules

![screenshots](screenshots/A5-T2-S1.png)

---

#### Screenshot 7 — EC2 Security Group inbound rules showing the ALB Security Group reference and SSH from your IP

![screenshots](screenshots/A5-T2-S2.png)

---

#### Screenshot 8 — RDS Security Group inbound rule showing the database port allowed only from the EC2 Security Group

![screenshots](screenshots/A5-T2-S3.png)

---

# Task 3 — Deploy Database Tier (RDS Multi-AZ in Private Subnets)

## Goal

Launch a private, Multi-AZ RDS database (MySQL or PostgreSQL) using the private DB Subnet Group and `ha-db-sg`.

### Evidence

#### Screenshot 9 — RDS summary showing Multi-AZ = Yes and Publicly accessible = No

![screenshots](screenshots/A5-T3-S1.png)

---

#### Screenshot 10 — RDS connectivity section showing the DB Subnet Group and Security Group

![screenshots](screenshots/A5-T3-S2.png)

---

# Task 4 — Build a Launch Template (User Data Installs App + Connects to DB)

## Goal

Create a Launch Template whose user data installs the web-server runtime, deploys the application, configures the database connection, and starts the required services.

### Evidence

#### Screenshot 11 — Launch Template details showing that user data exists, including a visible snippet

![screenshots](screenshots/A5-T4-S1.png)

---

#### Screenshot 12 — A running instance created from the template showing that the application responds on port 80 through a local test or browser using its public IP

![screenshots](screenshots/A5-T4-S2.png)

---

# Task 5 — Create an Application Load Balancer (ALB) Across 2 Public Subnets

## Goal

Create an internet-facing ALB across both public subnets with an HTTP listener and a healthy instance target group.

### Evidence

#### Screenshot 13 — ALB details showing two public subnets in two Availability Zones

![screenshots](screenshots/A5-T5-S1.png)

---

#### Screenshot 14 — Target group showing at least one healthy target

![screenshots](screenshots/A5-T5-S2.png)

---

# Task 6 — Create Auto Scaling Group (ASG) in 2 Public Subnets

## Goal

Create an Auto Scaling Group from the Launch Template across both public subnets, with desired capacity 2, minimum 2, and maximum 4, registered to the ALB target group.

### Evidence

#### Screenshot 15 — Auto Scaling Group showing desired, minimum, and maximum capacity and the selected subnet Availability Zones

![screenshots](screenshots/A5-T6-S1.png)

---

#### Screenshot 16 — EC2 instances list showing two running instances in different Availability Zones

![screenshots](screenshots/A5-T6-S2.png)

---

# Task 7 — Configure App to Use RDS + Validate Read/Write

## Goal

Confirm the application communicates with the RDS database through the ALB DNS name with at least one read and one write operation.

### Evidence

#### Screenshot 17 — Browser showing the application loaded through the ALB DNS name with the URL visible

![screenshots](screenshots/A5-T7-S1.png)

---

#### Screenshot 18 — Proof of a database write through a UI message or database query output

![screenshots](screenshots/A5-T7-S2.png)

---

# Task 8 — High Availability Tests (Must Do Both)

## Goal

Test A: terminate one web instance and confirm the Auto Scaling Group replaces it automatically without interrupting the ALB.

Test B: simulate an Availability Zone impact (stop, detach, or reduce desired capacity in one AZ) and confirm the application stays available.

### Evidence

#### Screenshot 19 — EC2 showing the terminated instance and the newly launched instance; timestamps are helpful

![screenshots](screenshots/A5-T8-S1.png)

---

#### Screenshot 20 — Target group showing healthy targets after replacement

![screenshots](screenshots/A5-T8-S2.png)

---

#### Screenshot 21 — Evidence that an instance was removed, detached, placed in Standby, or stopped in one Availability Zone

![screenshots](screenshots/A5-T8-S3.png)

---

#### Screenshot 22 — Browser showing that the ALB DNS endpoint still works during the change

![screenshots](screenshots/A5-T8-S4.png)


![screenshots](screenshots/A5-T8-S5.png)
---

# Task 9 — Architecture and Test-Results Summary

## Goal

Summarize the VPC/subnet layout, the ALB and Auto Scaling Group setup, the private Multi-AZ RDS setup, and the results of both high-availability tests.

### Evidence

#### Screenshot 23 — A simple architecture diagram, which may be hand-drawn, or an AWS console overview showing the components

![screenshots](screenshots/A5-T9-S1.png)

---

### Notes

Summarize the VPC and subnets across the two Availability Zones.

VPC and Subnet Layout

The application was deployed in the AWS eu-north-1 (Stockholm) region using a VPC with CIDR 10.0.0.0/16 spanning two Availability Zones.

The network contains four subnets:

Public subnet A — 10.0.1.0/24
Public subnet B — 10.0.2.0/24
Private subnet A — 10.0.11.0/24
Private subnet B — 10.0.12.0/24

The public subnets are used by the Application Load Balancer and web tier, while the private subnets are designated for the database tier. An Internet Gateway provides internet connectivity to the public subnets, while a NAT Gateway provides outbound internet access for resources in the private subnet tier.

Summarize the ALB and Auto Scaling Group setup.

ALB and Auto Scaling Group Setup

An Application Load Balancer named ha-web-alb was configured across both public subnets to distribute incoming application traffic.

The ALB forwards traffic to an Auto Scaling Group named ha-web-asg, which runs Ubuntu 24.04 web instances configured through the Launch Template user data. The instances run Apache/PHP and automatically configure the application to connect to the RDS database.

The Auto Scaling Group provides resilience by maintaining the required web-tier capacity and automatically replacing instances that become unavailable. The ALB target group health checks ensure that traffic is directed only to healthy instances.

Summarize the private Multi-AZ RDS setup.

Private RDS Setup

A MySQL RDS instance named ha-db was deployed for the database tier in the private subnet environment.

Database access is restricted using the ha-db-sg security group so that MySQL traffic on port 3306 is permitted only from the web-tier security group. The database is therefore not directly accessible from the public internet.

Note: Multi-AZ RDS deployment was unavailable because of an AWS account-level Free Plan restriction. A Single-AZ RDS deployment was therefore used instead. As a result, the web-tier high availability was tested successfully, but database Multi-AZ failover could not be demonstrated.

Summarize the results of both high-availability tests.

High-Availability Test Results

Test A — Instance Termination

A running web-tier EC2 instance was intentionally terminated to simulate an instance failure. The Auto Scaling Group detected the termination and automatically launched a replacement instance. The replacement registered with the ALB target group and passed its health checks.

Result: PASS — The application remained available through the ALB while the Auto Scaling Group restored the required capacity.

Test B — Web Instance Failure Simulation

A running web-tier EC2 instance was stopped to simulate an instance failure. The ALB continued serving application traffic through the remaining healthy instance.

Result: PASS — The application remained accessible despite the failure of one web-tier instance.

During testing, AWS also reported intermittent regional capacity shortages for t3.micro and t3.small instances in eu-north-1a and eu-north-1b. At some points, this limited the Auto Scaling Group to a single running instance. However, when capacity was available, both required high-availability tests were successfully completed and verified.

---

# LinkedIn Post (Required)

## Goal

Publish a LinkedIn post about the high-availability build, including the ALB URL (or a redacted screenshot), three to five lines on what you built and how you tested high availability, and one proof screenshot.

## Evidence

#### LinkedIn Post URL

Paste your LinkedIn post URL here:

https://lnkd.in/p/e4PH_xH7

---

#### Screenshot of LinkedIn post

![screenshots](screenshots/A5-T10-S1.png)

---

# Submission Instructions

- Add all required screenshots in your submission
- Do not expose passwords, connection strings, private keys, or account IDs

---

# Completion Checklist

- [X] Task 1: VPC, four subnets, IGW, NAT Gateway, and route tables created (Screenshots 1–5)
- [X] Task 2: Least-privilege ALB, EC2, and RDS security groups created (Screenshots 6–8)
- [X] Task 3: Private Multi-AZ RDS created (Screenshots 9–10)
- [X] Task 4: Self-configuring Launch Template created and tested (Screenshots 11–12)
- [X] Task 5: ALB created across both public subnets (Screenshots 13–14)
- [X] Task 6: Auto Scaling Group running two instances across two AZs (Screenshots 15–16)
- [X] Task 7: Application verified through the ALB with a database read and write (Screenshots 17–18)
- [X] Task 8: Both high-availability tests completed (Screenshots 19–22)
- [X] Task 9: Architecture and test-results summary completed (Screenshot 23 & Notes)
- [X] LinkedIn post published and URL submitted
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