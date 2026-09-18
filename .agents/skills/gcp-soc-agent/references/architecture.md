# Architecture & GCP Infrastructure Reference

## 1. Google Cloud Platform Architecture

```
                                  +-----------------------+
                                  |    Alert Producers    |
                                  |  - Sumo Logic Webhook |
                                  |  - Cloud Armor Logs   |
                                  |  - Nightfall Webhook  |
                                  |  - SCC Findings       |
                                  +-----------+-----------+
                                              |
                                              v
                               +-----------------------------+
                               |  Cloud Pub/Sub: soc-alerts  |
                               +--------------+--------------+
                                              |
                     +------------------------+------------------------+
                     | Push Subscription                                | Dead Letter (max delivery = 3)
                     v                                                  v
     +-------------------------------+                       +----------------------+
     |   Cloud Run: soc-triage-agent |                       |  Pub/Sub:            |
     |   (Serverless, 15m timeout)   |                       |  soc-alerts-dlq      |
     +---------------+---------------+                       +----------------------+
                     |
       +-------------+-------------+
       |                           |
       v                           v
+--------------+           +------------------+
|  Vertex AI   |           |  Google Secret   |
|  Gemini 3    |           |  Manager         |
|  / Claude    |           |  (API Tokens)    |
+--------------+           +------------------+
       |
       +---------------------------+
       |                           |
       v                           v
+------------------+       +------------------+
| Structured JSON  |       |  Slack / Staging |
| to Cloud Logging |       |  (Questions for  |
+------------------+       |   Analyst)       |
                           +------------------+

--------------------------------------------------------------------------------

+----------------------+
|   Cloud Scheduler    |
|   (cron: 0 */6 * * *)|
+----------+-----------+
           |
           v
+-------------------------------+
| Cloud Run Job:                |
| soc-threat-hunter             |
| (Runs to completion, max 24h) |
+----------+--------------------+
           |
           v
+-------------------------------+
| CISA KEV + ThreatFox Feeds    |
| Sweep against Sumo Logic &    |
| GCP Cloud Audit Logs          |
+----------+--------------------+
           |
           v
+-------------------------------+
| Slack Threat Hunt Report      |
+-------------------------------+
```

---

## 2. Infrastructure Component Specifications

### 2.1 Cloud Pub/Sub
- **Topic**: `projects/{PROJECT_ID}/topics/soc-alerts`
- **Dead Letter Topic**: `projects/{PROJECT_ID}/topics/soc-alerts-dlq`
- **Push Subscription**:
  - Push endpoint: `https://{CLOUD_RUN_SERVICE_URL}/`
  - `ack_deadline_seconds`: `600` (10 minutes)
  - `dead_letter_policy`:
    - `dead_letter_topic`: `projects/{PROJECT_ID}/topics/soc-alerts-dlq`
    - `max_delivery_attempts`: `3`
  - `retry_policy`: Exponential backoff (`minimum_backoff = 10s`, `maximum_backoff = 600s`)

### 2.2 Cloud Run Service (`soc-triage-agent`)
- **Runtime**: Containerized Python 3.12 (using FastAPI/Uvicorn)
- **Concurrency**: `5` (to prevent hitting LLM rate limits simultaneously)
- **Timeout**: `900s` (15 minutes hard ceiling)
- **Memory**: `2Gi`
- **CPU**: `2`
- **Autoscaling**: `min_instances = 0`, `max_instances = 10`
- **Ingress**: Internal + Cloud Load Balancing (or restricted to Pub/Sub push identity token)
- **Service Account**: `soc-triage-sa@{PROJECT_ID}.iam.gserviceaccount.com`

### 2.3 Cloud Run Job (`soc-threat-hunter`)
- **Runtime**: Containerized Python 3.12 (CLI entrypoint)
- **Timeout**: `3600s` (1 hour)
- **Memory**: `4Gi`
- **CPU**: `2`
- **Task Count**: `1`
- **Trigger**: Cloud Scheduler invoking `run-job` via HTTP API with OAuth token

### 2.4 Google Secret Manager
All sensitive credentials are stored in Secret Manager and mounted as environment variables in Cloud Run:
- `SUMO_ACCESS_ID` & `SUMO_ACCESS_KEY`
- `OKTA_DOMAIN` & `OKTA_API_TOKEN`
- `NIGHTFALL_API_KEY`
- `CROWDSTRIKE_CLIENT_ID` & `CROWDSTRIKE_CLIENT_SECRET`
- `JAMF_URL`, `JAMF_USERNAME`, `JAMF_PASSWORD`
- `JAMF_SECURITY_API_KEY`
- `INTUNE_TENANT_ID`, `INTUNE_CLIENT_ID`, `INTUNE_CLIENT_SECRET`
- `WORKSPACE_SERVICE_ACCOUNT_KEY`
- `ABNORMAL_API_KEY`
- `VIRUSTOTAL_API_KEY`
- `SLACK_BOT_TOKEN` & `SLACK_CHANNEL_ID`

---

## 3. Least-Privilege IAM Matrix

| Service Account | Roles Granted | Justification |
| :--- | :--- | :--- |
| `soc-triage-sa` | `roles/secretmanager.secretAccessor` | Access API tokens at runtime |
| `soc-triage-sa` | `roles/logging.logWriter` | Emit structured JSON to Cloud Logging |
| `soc-triage-sa` | `roles/logging.viewer` | Query Cloud Audit & Cloud Armor logs |
| `soc-triage-sa` | `roles/aiplatform.user` | Invoke Vertex AI Gemini / Claude models |
| `pubsub-invoker-sa` | `roles/run.invoker` | Authorize Pub/Sub to invoke Cloud Run |
| `scheduler-invoker-sa` | `roles/run.developer` | Authorize Cloud Scheduler to trigger Cloud Run Job |
