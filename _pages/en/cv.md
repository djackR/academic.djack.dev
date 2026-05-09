---
layout: archive
title: "CV"
permalink: /en/cv/
author_profile: true
lang: en
---

{% include base_path %}

Experience
======

**V360 — SRE Tech Lead** (Jan 2026 – Present) *Remote*
* Leading SRE initiatives focused on reliability, observability, automation, and operational excellence for mission-critical systems
* Responsible for maintaining SLA/SLO targets, ensuring web latency below 500ms and queue processing below 30 seconds
* Managed production-grade EKS and ECS environments supporting critical services and distributed engineering teams
* Led infrastructure cost optimization by migrating workloads from AWS Lambda to Kubernetes, reducing monthly cloud costs by 83% (from USD 7k to USD 1.2k)
* Designed and maintained self-hosted GitHub Actions runners on AWS, reducing CI/CD pipeline execution time from hours to approximately 15 minutes
* Implemented observability solutions, operational dashboards, and alerting systems using CloudWatch and OpenSearch
* Led IAM Identity Center integration and Keycloak deployment on Kubernetes for centralized authentication and access governance
* Supported incident response, root cause analysis (RCA), and troubleshooting of high-availability distributed systems

**V360 — Cloud Engineer (Mid-level)** (Jan 2025 – May 2026) *Remote*
* Led infrastructure modernization by migrating workloads from AWS Elastic Beanstalk to ECS, including Docker containerization and cloud-native infrastructure design
* Optimized platform performance, reducing web application latency from approximately 1.5s to 600ms through architectural and infrastructure improvements
* Redesigned queue processing architecture by splitting monolithic workers into multiple dedicated services, significantly improving scalability and operational latency
* Executed PostgreSQL major version upgrades (v14 to v17) in production environments with minimal downtime and validated disaster recovery readiness
* Performed PostgreSQL backup recovery validation and disaster recovery testing to ensure operational reliability
* Implemented automation and CI/CD improvements using GitHub Actions and self-hosted runners on AWS

**V360 — Cloud Engineer (Junior) & IT Infrastructure Intern** (Nov 2022 – Jan 2025) *Remote*
* Supported AWS production operations, deployment automation, infrastructure monitoring, and cloud environment maintenance
* Assisted with CloudWatch monitoring setup, operational support, and infrastructure reliability initiatives

Education
======
* **M.Sc. in Computer Science** — Federal University of Pelotas (UFPel), 2022 – 2025
* **B.Eng. in Energy Engineering** — Federal University of Santa Catarina (UFSC), 2013 – 2021

Technical Skills
======
* **Cloud & Orchestration:** AWS (EKS, ECS, EC2, Lambda, RDS, OpenSearch, Route53, WAF, IAM Identity Center, VPC, CloudWatch), Kubernetes, Docker
* **Monitoring & Reliability:** CloudWatch, OpenSearch, Observability, Incident Response, SLA/SLO, PostgreSQL, Redis
* **CI/CD & Automation:** GitHub Actions, Self-hosted Runners, Infrastructure Automation
* **Programming & Scripting:** Bash, Ruby, Python, SQL
* **Systems & Security:** Linux, IAM Identity Center, Keycloak, Secrets Management
* **Concepts:** SRE, Platform Engineering, High Availability, Distributed Systems, Operational Excellence

Certifications
======
* AWS Educate – Introduction to Cloud
* Google Cloud Computing Foundations
* Practical Data Science

Languages
======
* **Portuguese:** Native
* **English:** Professional Working Proficiency
* **Spanish:** Limited Working Proficiency

Publications
======
  <ul>{% for post in site.publications reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
