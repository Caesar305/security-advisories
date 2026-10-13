# Security research case studies

This repository contains sanitized accounts of vulnerability research and remediation. Findings distinguish confirmed behavior from potential impact and describe fixes without publishing exploitation instructions.

## Case studies

| Case study | Finding | Remediation |
| --- | --- | --- |
| [Authenticated privilege escalation in a commercial SaaS platform](advisories/saas-self-service-privilege-escalation.md) | Self-service updates could modify administrative attributes | Dedicated input schema and explicit field allow-lists |

## Research context and disclosure

The case study describes internal research performed by a researcher affiliated with the affected platform's vendor. It is a first-party account, not an independent assessment. The vendor, product, researcher name, version numbers, dates, and internal references are intentionally withheld from this public-facing draft.

The write-up does not claim a CVE assignment, independent validation, or acceptance by an external security program. An anonymized narrative alone does not independently prove discovery or remediation.

Public content excludes endpoint paths, field names, exploitation payloads, credentials, customer data, sensitive source code, and production logs. New security concerns should be reported privately to the relevant vendor through an established support or security channel.
