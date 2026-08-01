# AWS Security Specialty: The Long Road

![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)
![Focus](https://img.shields.io/badge/Focus-AWS%20Cloud%20Security-blue)
![Certification](https://img.shields.io/badge/Certification-AWS%20SCS--C03-8A2BE2)
![SDK](https://img.shields.io/badge/SDK-boto3-orange)
![Method](https://img.shields.io/badge/Method-Hands--On-4479A1)

This repository is the record of working from zero AWS knowledge toward the AWS Certified Security Specialty (SCS-C03), and toward the job the certification points at. Everything here was built by hand: a full subdomain-by-subdomain lab pass over the exam blueprint, three boto3 security tools, and a Python and SQL foundation underneath both.

---

## What Is In Here

**[AWS Journeys Edge](https://github.com/Snowblind019/AWS-Journeys-Edge)**, the hands-on lab series. Every domain and subdomain of the SCS-C03 blueprint, built out in AWS and captured as a CCNA-style lab: diagrams, CLI walkthroughs, the gotchas that only surface on deployment, and checkpoint questions for recall. Complete across all six domains.

**[AWS Security Projects](https://github.com/Snowblind019/AWS-Security-Projects)**, three capstone tools:

| Project | Stack | What it does |
|---------|-------|--------------|
| S3 Security Auditor | boto3 | Scans buckets for public access, missing encryption, and disabled versioning or logging, then sends SNS alerts grouped by severity |
| Security Group Auditor | boto3 | Flags overly permissive rules (0.0.0.0/0) and unused groups, alerting on each finding |
| CloudTrail Log Analyzer | boto3, SQL | Runs Athena queries against CloudTrail logs for root usage, failed auth, IAM changes, and other suspicious activity |

**Supporting work** that made the above possible: Python fundamentals into the boto3 SDK, Lambda and event-driven automation (EC2 backups, S3 validation, VPC cleanup, plus six security and cost functions), an SNS and SQS pipeline that alerts and retries on failure, a Glue and EMR job processing billing data at scale with PySpark, and SQL picked up for the CloudTrail work then rounded out through fundamentals.

---

## The Method: One Subdomain at a Time

The CCNA was a work requirement, and passing it did more than check a box. It gave me the method I now point at everything.

Studying for the CCNA I went objective by objective: take a single subdomain, build it out by hand, break it down, drill it, and only then move on. My first CCNA attempt skipped the hands-on labs, and it cost me. The granular, build-it-yourself pass is what carried me over the line the second time.

I pointed the same method at the SCS:

- **Domain by domain, then subdomain by subdomain.** No studying broad and hoping it sticks.
- **Built out in AWS by hand** for each subdomain, rather than read about.
- **Captured as a CCNA-style lab:** diagrams, CLI walkthroughs, deployment gotchas, checkpoint questions.
- **Reasoning documented in my own words,** so I own why one approach wins when the exam offers several that all look right.

That pass is finished. What remains before attempt four is drilling practice tests to pressure-test it.

---

## From Studying to Building

The turn in this project came when I stopped studying for an exam and started training for the job.

I began with Python fundamentals, then moved into boto3. Early on I made it harder than it needed to be, trying to memorize every line of syntax while also understanding the reasoning behind it. That burned me out fast. Once I accepted that syntax sticks on its own through daily use, and that the real skill is knowing what the commands do and being comfortable in the documentation, the whole thing opened up.

From there the work built on itself: guided builds for S3, EC2, RDS, and VPC typed out by hand; mini-projects tying those services together in a single script, first two at a time, then all four, to see what integration actually looks like; then Lambda, event-driven automation, and resilient systems.

Memorizing syntax was never the goal. Understanding how services fit together, what event-driven systems look like, and how to build something that survives failure, that was the goal. Syntax is the tool you reach for, and the docs are always there.

---

## Where This Stands

| Track | Status |
|-------|--------|
| Python and boto3 module | **Complete** |
| SQL fundamentals | **Complete** |
| Capstone security projects (x3) | **Complete** |
| CCNA (work requirement) | **Passed** |
| Subdomain hands-on pass (Journeys Edge) | **Complete** |
| Practice tests for attempt 4 | **In progress** |
| SCS-C03 exam | **Attempt 4 ahead** |

---

## Why I'm Still Here

Getting here took three failed attempts and a full rebuild of how I study. Each one exposed a real gap, and each fix is visible in the repos above: theory replaced by first principles, first principles replaced by building, narrow building replaced by depth across every subdomain. The full breakdown of what each attempt taught me is in [RETROSPECTIVE.md](Retrospective.md).

The goal was never the badge. It is to become a Cloud Security Engineer, and this is the depth you cannot fake through theory, only earn by building and breaking things.

> *"With men this is impossible, but with God all things are possible."*
> Matthew 19:26

---

## Companion Repos

- [AWS Journeys Edge](https://github.com/Snowblind019/AWS-Journeys-Edge), subdomain-by-subdomain hands-on labs
- [AWS Security Projects](https://github.com/Snowblind019/AWS-Security-Projects), the three capstone auditors and analyzer

## Connect With Me

- **LinkedIn:** [emilp-profile](https://www.linkedin.com/in/emilp-profile/)