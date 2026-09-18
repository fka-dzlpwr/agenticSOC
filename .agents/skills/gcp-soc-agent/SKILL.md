---
name: gcp-soc-agent
description: >-
  Builds, deploys, and operates an autonomous AI SOC Agent system on Google Cloud Platform (GCP).
  Use this skill whenever the user asks to implement, debug, test, or deploy the AI SOC triage
  and threat hunting agents, configure GCP infrastructure (Cloud Run, Pub/Sub, Cloud Scheduler),
  or integrate enterprise telemetry (Sumo Logic, Okta, GCP Audit Logs, Cloud Armor, Google Workspace,
  Nightfall.AI, CrowdStrike Falcon, Jamf, Microsoft Intune, Abnormal.AI, CISA KEV).
---

# GCP AI SOC Agent Development & Deployment Skill

This skill provides step-by-step procedures and runbooks for building, operating, and extending the **AgenticSOC** system on Google Cloud Platform.

## 1. Quick Architecture Reference

- **Triage Agent**: Event-driven container running on **Google Cloud Run**, subscribed to a **Cloud Pub/Sub** push topic (`soc-alerts`) with a dead-letter queue (`soc-alerts-dlq`).
- **Threat Hunter**: Scheduled batch container running on **Google Cloud Run Jobs**, invoked every 6 hours by **Cloud Scheduler**.
- **LLM Engine**: **Vertex AI** (`gemini-3.0-pro` / `gemini-3.8-flash` or `claude-3-7-sonnet` on Vertex) or Anthropic direct API.
- **Investigation Framework**: 4-phase hypothesis-driven investigation with a mandatory double self-critique loop.
- **Staging Boundaries**: Read-only tools for autonomous context gathering; outputs end in **"questions, not instructions"** for human review in Slack.

For complete deep-dive documentation, consult:
- [GCP Architecture Reference](./references/architecture.md)
- [Enterprise Telemetry Reference](./references/telemetry.md)
- [Prompts & Self-Critique Reference](./references/prompts.md)
- [Master Project Plan](../../../PROJECT_PLAN.md)

---

## 2. Common Workflows & Runbooks

### Runbook A: Running Local Mock Triage

To verify agent reasoning without connecting to live enterprise APIs:

```bash
# Test alert scenario 1: Suspicious IAM Role Assumption + Okta + Jamf compliance
python3 -m agenticsoc.cli triage --scenario privilege_escalation --mock

# Test alert scenario 2: Phishing (Abnormal) + Session Hijack (Okta) + EDR (CrowdStrike)
python3 -m agenticsoc.cli triage --scenario credential_theft --mock

# Test alert scenario 3: Cloud Armor Blocked Exploit + GCP Audit Logs + Nightfall DLP
python3 -m agenticsoc.cli triage --scenario cloud_armor_dlp --mock
```

**Validation Checks**:
1. Confirm the output includes all 4 phases: Initial Assessment, Evidence Collection, Classification, and Self-Critique.
2. Verify the structured output adheres to `TriageReport` JSON schema.
3. Confirm that the report ends with specific **questions for the analyst** rather than destructive mitigation commands.

---

### Runbook B: Running Scheduled Threat Hunt Locally

To test proactive historical log hunting:

```bash
# Hunt across the last 14 days using pre-cached CISA KEV and IOC feeds
python3 -m agenticsoc.cli hunt --lookback-days 14 --mock
```

**Validation Checks**:
1. Phase 1 (Environment Discovery): Confirms available log sources and active cloud projects.
2. Phase 2 (Threat Intel): Filters CISA KEV for vulnerabilities relevant to discovered technologies.
3. Phase 4 (Log Sweep): Executes broad IOC sweep across Sumo Logic and GCP Audit Logs.
4. Phase 6 (Report): Posts structured hunt summary with confirmed negative/positive hits and identified telemetry visibility gaps.

---

### Runbook C: Testing the Cloud Run HTTP / Pub/Sub Push Handler

Cloud Run receives alerts wrapped in a Pub/Sub push message format:

```bash
# Start local server
python3 -m uvicorn agenticsoc.server.app:app --host 0.0.0.0 --port 8080 &

# Send simulated Pub/Sub push event
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
      "attributes": {"source": "sumo_logic"},
      "data": "eydhbGVydF9pZCc6ICdhbGVydC0xMjM0JywgJ3VzZXInOiAnamFuZUBhY21lLmNvbScsICdzZXZlcml0eSc6ICdISUdIJ30=",
      "messageId": "msg-98765"
    },
    "subscription": "projects/my-project/subscriptions/soc-alerts-push"
  }'
```

---

### Runbook D: Deploying Infrastructure via Terraform

```bash
cd terraform
terraform init
terraform plan -var="project_id=YOUR_GCP_PROJECT_ID" -var="region=us-central1"
terraform apply -var="project_id=YOUR_GCP_PROJECT_ID" -var="region=us-central1"
```

Resources provisioned:
- Artifact Registry repository (`soc-agent-repo`)
- Pub/Sub Topic (`soc-alerts`) and Dead Letter Topic (`soc-alerts-dlq`)
- Pub/Sub Push Subscription linked to Cloud Run service
- Cloud Run Service (`soc-triage-agent`)
- Cloud Run Job (`soc-threat-hunter`)
- Cloud Scheduler Job (cron trigger every 6 hours)
- Google Secret Manager secrets for all 11 security tool API tokens
- Least-privilege IAM service accounts for Cloud Run and Cloud Scheduler

---

## 3. Tool Development Guidelines

When adding or modifying an enterprise telemetry tool in `src/agenticsoc/tools/`:
1. Inherit from `BaseSecurityTool` in `src/agenticsoc/tools/base.py`.
2. Implement `execute(query_params: dict) -> ToolResult`.
3. Provide a deterministic `mock_execute(query_params: dict) -> ToolResult` to ensure hermetic testing works out-of-the-box.
4. Keep the tool strictly **read-only** (`read_only=True`). Any remediation or modification action must be returned as a recommendation in `TriageReport.suggested_actions` for human analyst confirmation.
