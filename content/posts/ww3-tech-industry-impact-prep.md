---
title: "World War 3 and the Tech Industry: Impacts and What to Prepare For"
date: 2026-03-22T11:59:00-06:00
draft: true
---

> **Note (scope + uncertainty):** “World War 3” is often used as shorthand for a broad range of escalation scenarios (regional war expanding, major-power conflict, or sustained multi-theater confrontation). The exact impacts on tech depend heavily on **where** conflict occurs, **who** is involved, and **how long** it lasts. This post focuses on practical, non-alarmist preparation for businesses and individuals in tech.

## How a major war would ripple through tech

### 1) Supply chains: chips, components, and logistics get stressed fast
Tech is unusually dependent on a global, tightly-coupled supply chain:

- **Semiconductors & advanced packaging:** Any disruption to foundries, specialty materials (photoresists, gases), or advanced packaging capacity can create long lead times.
- **Electronics components:** Power management ICs, passives, connectors, PCBs—small shortages can stall whole products.
- **Shipping and insurance:** Conflict can raise freight costs, reroute shipping lanes, and increase cargo insurance. Lead times become less predictable.

**What it looks like in practice:** delayed product launches, higher COGS, last-minute redesigns (BOM substitutions), and inventory swings.

### 2) Cyber escalation: critical infrastructure and enterprises become targets
In major geopolitical conflict, cyber operations often intensify:

- **Ransomware and disruption campaigns** become more frequent and more destructive.
- **Targeting shifts** from “pay us” to “cause maximum downtime.”
- **Supply-chain attacks** (dependencies, CI/CD, vendor compromise) increase.

**Impact on tech companies:** larger security spend, stricter controls, increased incident response workload, and reputational risk.

### 3) Cloud and connectivity: stability matters more than speed
A large conflict can stress:

- **Undersea cables and regional peering** (physical sabotage risk, outages, reroutes).
- **Cross-border data flows** (policy restrictions, sanctions compliance, data localization).
- **Platform concentration risk** (dependency on a small number of hyperscalers).

**Outcome:** resilience becomes a competitive advantage; “multi-region” and “multi-cloud” discussions become real.

### 4) Regulation, sanctions, and export controls reshape the market
War can trigger rapid changes in:

- **Export controls** (advanced GPUs, lithography, encryption tools)
- **Sanctions** (customer screening, payment rails, vendor restrictions)
- **Talent mobility** (visa regimes, travel restrictions)

**Impact:** some markets become inaccessible; compliance and legal review becomes part of product and sales operations.

### 5) Capital allocation and hiring shift
Depending on severity:

- Risk capital can retreat; **valuations compress**.
- Budgets shift toward **defense, energy, and security** priorities.
- Some segments (cyber, defense tech, resilience tooling) grow faster.

## What different parts of tech should prepare for

### Hardware / semiconductor / embedded teams
- **Second-source critical parts** (or redesign for alternates).
- Maintain a **parts risk register**: single-source components, long-lead items, geopolitical pinch points.
- Consider **inventory strategy** for truly critical components (but avoid panic stocking).
- Build test and qualification capability to validate substitutions quickly.

### Cloud / SaaS teams
- Map your **critical dependencies** (identity, payments, DNS, email, CI/CD, observability, CDNs).
- Improve **graceful degradation**: what still works when a dependency is down?
- Strengthen **regional resilience**: multi-region failover runbooks and game days.
- Tighten **security posture**: least privilege, MFA everywhere, secrets hygiene, SBOMs.

### Security teams (and everyone adjacent)
- Prepare for **high-intensity incident response**:
  - rehearsed playbooks
  - clear escalation paths
  - offline backups + restore drills
  - tabletop exercises with execs
- Watch vendor risk: **access logs, SSO configuration, third-party app approvals**.

### Leadership / operations
- Update business continuity:
  - key suppliers and alternatives
  - critical roles and succession
  - remote work contingencies
  - communications plan (outages, misinformation)
- Align on a **sanctions/export controls** response plan with counsel.

## A practical readiness checklist (non-alarmist)

### Company-level checklist
1. **Dependency map:** top 20 external systems/services you can’t operate without.
2. **Resilience targets:** define RTO/RPO for your critical services; test them.
3. **Backups:** immutable + offline copy; practice restores.
4. **Security baseline:** MFA, device posture, least privilege, patch SLAs.
5. **Supply chain risk:** SBOM, signed builds, hardened CI/CD, vendor security review.
6. **Compliance readiness:** sanctions screening process; export classification awareness.
7. **Crisis comms:** internal + external messaging templates.

### Individual career checklist
- Build skills in **security, reliability, systems thinking**, and **risk management**.
- Keep an updated resume + portfolio; maintain financial runway if possible.
- Avoid single points of failure in your own workflow (2FA backups, password manager, device backups).

## What to watch (early signals)
- Restrictions on advanced compute exports or cloud services.
- Sudden spikes in cyber incidents across sectors.
- Shipping route changes, insurance spikes, persistent lead time increases.
- Large-scale sanctions expansion and payment rail disruptions.

## Closing thought
The best “WW3 preparation” for tech is mostly the same discipline that makes companies strong in normal times: **reduce single points of failure**, **practice recovery**, **know your dependencies**, and **design for disruption**.

If you want, I can tailor this post to your audience (verification / hardware / ASIC) and add a section on how geopolitical risk affects **chip verification workloads, EDA access, and compute availability**.
