# Project Context & Rules: GCP-Native Autonomous SOC Agent System

Welcome to **AgenticSOC**, an enterprise-grade autonomous Security Operations Center (SOC) agent system engineered for **Google Cloud Platform (GCP)**.

This system is inspired by the Scanner.dev AI SOC agent architecture ([Part 1: Foundations](https://scanner.dev/blog/building-your-first-ai-soc-agents-foundations-and-your-first-agent-part-1) and [Part 2: Deployment](https://scanner.dev/blog/building-your-first-ai-soc-agents-deploying-triage-and-threat-hunting-agents-in-aws-part-2)) and adapted to run natively on Google Cloud with multi-source enterprise security telemetry.

---

## 1. Core Operating Principles

Every agent implementation and workflow in this codebase MUST follow these four core principles:

1. **Hypothesis-Driven Investigation**:
   - Do NOT jump to conclusions. For every alert, generate 2-4 candidate hypotheses ranked by likelihood:
     - Benign explanation (legitimate administrative action, scheduled job, routine user travel)
     - Misconfiguration / Policy violation (improperly configured service, accidental secret exposure)
     - Actual malicious attack (credential compromise, lateral movement, data exfiltration)
     - Insider threat (authorized principal abusing elevated access)
   - Systematically gather targeted evidence to confirm or refute each hypothesis.

2. **Double Self-Critique Loop**:
   - Before finalizing any triage classification, the agent MUST run two explicit critique passes:
     - Pass 1: Identify blind spots (e.g., did we verify if the IP is an internal proxy or corporate ZTNA gateway?).
     - Pass 2: Challenge confidence calibration and check what missing evidence could overturn the verdict.

3. **"Questions, Not Instructions" (Staging Boundaries)**:
   - Follow the **read-only investigation, staging for response** paradigm.
   - Use **read-only tools** for gathering context.
   - Never execute irreversible or destructive actions autonomously (e.g. revoking credentials, deleting files, modifying firewall rules).
   - End triage reports with **targeted, clarifying questions for the human analyst** (e.g. *"Did Jane have an approved change ticket to access the customer database?"*) rather than prescriptive instructions (*"Disable this user"*).

4. **Structured JSON Audit Trail**:
   - Log every tool invocation, search query, and final classification as structured JSON to Google Cloud Logging (`console.log(JSON.stringify({...}))` / Python `logger.info(extra={...})`).

---

## 2. Telemetry Matrix (11 Enterprise Integrations)

The agent connects to and correlates across 11 security platforms:
- **Centralized Audit Logs & WAF**:
  - `Sumo Logic`: Historical SIEM search job queries
  - `Google Cloud Audit Logs`: GCP Admin Activity, Data Access, IAM permission changes
  - `Google Cloud Armor`: Edge WAF/DDoS security policy logs, blocked requests, Adaptive Protection
- **Primary IdP & SaaS**:
  - `Okta`: Primary IdP (System Log `/api/v1/logs`, MFA factor evaluation, ThreatInsight risk)
  - `Google Workspace`: Admin SDK Reports API (Logins, 2FA challenges, Drive sharing, OAuth tokens)
- **Data Loss Prevention (DLP) & Exfiltration**:
  - `Nightfall.AI`: Developer API & MCP server (26 read-only tools: `search_violations`, `get_violation_findings`, `search_exfiltration_events`, `get_actor_activity`)
- **EDR & Endpoint Security**:
  - `CrowdStrike Falcon`: Host status, active detections, sensor containment
  - `Jamf Security Cloud`: Mobile Threat Defense (MTD), network threat prevention, ZTNA risk
- **MDM & Device Compliance**:
  - `Jamf Pro`: macOS/iOS fleet management, FileVault encryption, compliance
  - `Microsoft Intune`: Windows/Mobile compliance, Azure AD device registration
- **Email Security & ATO**:
  - `Abnormal.AI`: Phishing campaigns, BEC, compromised mailbox signals, abuse mailbox
- **Threat Intelligence**:
  - `CISA KEV`, `VirusTotal`, `ThreatFox`, `AlienVault OTX`

---

## 3. Infrastructure & Deployment Model

- **Event-Driven Triage**:
  - Ingestion via **Cloud Pub/Sub** (`soc-alerts`) with a **Dead Letter Queue** (`soc-alerts-dlq`).
  - Executed on **Google Cloud Run** (containerized microservice with configurable timeout up to 60m).
- **Scheduled Threat Hunting**:
  - Triggered by **Google Cloud Scheduler** (cron: `0 */6 * * *`).
  - Executed on **Google Cloud Run Jobs** (batch container task searching across 30-365 days of historical logs).
- **AI / LLM Engine**:
  - **Vertex AI** (Gemini 3 Pro / Gemini 3.8 Flash / Claude 3.7 Sonnet on Vertex) with fallback to Anthropic API.
- **Secrets Management**:
  - **Google Secret Manager** for all enterprise API tokens and credentials.
- **Infrastructure as Code**:
  - Managed exclusively through **Terraform** in the `terraform/` directory.

---

## 4. Skills & Reference Documents

When working on this repository in Antigravity:
- Activate the skill: `.agents/skills/gcp-soc-agent/SKILL.md`
- Master Plan: [`PROJECT_PLAN.md`](./PROJECT_PLAN.md)
- Telemetry Architecture: [`.agents/skills/gcp-soc-agent/references/telemetry.md`](./.agents/skills/gcp-soc-agent/references/telemetry.md)
- GCP Infrastructure: [`.agents/skills/gcp-soc-agent/references/architecture.md`](./.agents/skills/gcp-soc-agent/references/architecture.md)
- Prompts & Schemas: [`.agents/skills/gcp-soc-agent/references/prompts.md`](./.agents/skills/gcp-soc-agent/references/prompts.md)
