# Enterprise Telemetry Integration Reference (11 Tools)

This document specifies the exact API contracts, query syntax, and investigative signals for each of the 11 security platforms integrated into **AgenticSOC**.

---

## 1. Centralized Audit Logs & Edge Security

### 1.1 Sumo Logic
- **Primary Use**: Search cross-cloud logs (AWS, Azure, GCP, on-prem firewalls, VPNs) and **ingested Nightfall DLP logs**.
- **API Endpoint**: `POST https://api.{endpoint}.sumologic.com/api/v1/search/jobs`
- **Query Method**: Search Job API with polling for results (`/api/v1/search/jobs/{id}/messages`).
- **Standard Cross-Platform Investigative Query**:
  ```sql
  _sourceCategory=* (user="jane@acme.com" OR "203.0.113.42")
  | parse "action=* " as action
  | count by _sourceCategory, action, status
  ```
- **Nightfall DLP Ingestion Query (Centralized SIEM)**:
  ```sql
  _sourceCategory=nightfall/* (actor="jane@acme.com" OR "CRITICAL" OR "HIGH")
  | json "event.detectorName", "event.actor", "event.riskScore", "event.fileLocation" as detector, actor, risk, target
  | count by detector, actor, risk, target
  ```
- **Investigation Output**: Chronological event list, user actions across disparate systems, failed/successful authorizations, and correlated DLP violations.

### 1.2 Google Cloud Audit Logs
- **Primary Use**: Track IAM privilege modifications, service account key creations, resource deletions, and sensitive BigQuery/Cloud Storage reads.
- **API Endpoint**: `POST https://logging.googleapis.com/v2/entries:list`
- **Filter Syntax**:
  ```
  logName:("logs/cloudaudit.googleapis.com%2Factivity" OR "logs/cloudaudit.googleapis.com%2Fdata_access")
  protoPayload.authenticationInfo.principalEmail="jane@acme.com"
  timestamp >= "2026-09-10T18:00:00Z"
  ```
- **High-Risk Methods Checked**:
  - `google.iam.admin.v1.CreateServiceAccountKey`
  - `google.iam.v1.IAMPolicy.SetIamPolicy`
  - `google.cloud.resourcemanager.v3.Projects.SetIamPolicy`
  - `google.storage.objects.get`

### 1.3 Google Cloud Armor
- **Primary Use**: Edge WAF/DDoS security policy logs, blocked requests, and Adaptive Protection alerts.
- **API Endpoint**: Cloud Logging API filter:
  ```
  resource.type="http_load_balancer"
  jsonPayload.@type="type.googleapis.com/google.cloud.loadbalancing.type.LoadBalancerLogEntry"
  jsonPayload.enforcedSecurityPolicy.name="prod-edge-armor-policy"
  ```
- **Investigative Signals**:
  - `enforcedSecurityPolicy.outcome`: `ACCEPT` or `DENY`
  - `enforcedSecurityPolicy.priority`: Triggered rule priority
  - `enforcedSecurityPolicy.configuredAction`: Action configured in policy
  - `remoteIp`: Client source IP
  - `ruleEvaluations.matchedRule`: Evaluated OWASP ModSecurity rule (e.g. `sqli-v33-stable`, `rce-v33-stable`)

---

## 2. Primary Identity Provider (IdP) & SaaS

### 2.1 Okta (Primary IdP)
- **Primary Use**: System Log audit trail, authentication method verification, session risk, and impossible travel detection.
- **API Endpoint**: `GET https://{OKTA_DOMAIN}.okta.com/api/v1/logs`
- **Filter Syntax**:
  ```
  target.id eq "jane@acme.com" or actor.alternateId eq "jane@acme.com"
  since: 2026-09-10T18:00:00Z
  ```
- **Investigative Signals**:
  - `eventType`: `user.authentication.sso`, `user.session.start`, `policy.evaluate_sign_on`, `security.threat.detected`
  - `authenticationContext.credentialType`: `PASSWORD`, `OKTA_VERIFY`, `FASTPASS`, `FIDO2_WEBAUTHN`, `SMS`
  - `securityContext.threatSuspected`: Boolean flag from Okta ThreatInsight
  - `debugContext.debugData.risk`: Assessed session risk score (`LOW`, `MEDIUM`, `HIGH`)
  - `client.geographicalContext`: Latitude, longitude, city, country (evaluates velocity)

### 2.2 Google Workspace
- **Primary Use**: SaaS user login audits, 2FA challenges, Google Drive external file shares, and OAuth app grants.
- **API Endpoint**: `GET https://admin.googleapis.com/admin/reports/v1/activity/users/{userKey}/applications/{applicationName}`
- **Applications Queried**:
  - `login`: Check for anomalous logins, failed 2FA challenges, or suspicious login types (`login_challenge_failed`, `login_verification_method`)
  - `token`: Check for new OAuth third-party app authorizations (`authorize`)
  - `drive`: Check for bulk downloads, public file link sharing (`change_user_access`, `edit_document_permissions`)

---

## 3. Data Loss Prevention (DLP) & Exfiltration

### 3.1 Nightfall.AI (Developer API & MCP)
- **Primary Use**: Cloud DLP violation detection, sensitive data leakage (credentials, API keys, PII, PHI), and data exfiltration across SaaS and endpoints.
- **Ingestion & SIEM Architecture**:
  - Nightfall event and violation logs are ingested into **Sumo Logic** (`_sourceCategory=nightfall/*`).
  - Cross-source SIEM queries, timeline assembly, and correlation with network/cloud logs are executed via Sumo Logic search jobs.
  - The **Nightfall Developer API & MCP server** are queried during active investigation for granular finding details (`get_violation_findings`), raw detector matches, real-time actor activity profiling, and local endpoint AI inventory.
- **Endpoint**: `https://api.nightfall.ai/v3` or Nightfall MCP server protocol.
- **Tool Safety Annotation**:
  - **26 Read-Only Tools** (`readOnlyHint: true`):
    - `search_violations`: Structured search syntax: `user_email:jane@acme.com state:ACTIVE risk_label:HIGH`
    - `get_violation_findings`: Unpacks exact detector matches (e.g. `AWS_CREDENTIALS`, `STRIPE_KEY`, `US_SSN`, `PRIVATE_SSH_KEY`)
    - `search_exfiltration_events`: Filters by `event_type:file_download`, `actor_email:jane@acme.com`, `endpoint.device_id`
    - `get_actor_activity`: Full cross-SaaS DLP timeline for the investigated employee
    - `list_ai_inventory` & `list_mcp_servers`: Discovers unauthorized AI extensions, plugins, or MCP servers running on endpoints
  - **7 Destructive Tools** (`destructiveHint: true`):
    - `take_action_on_violations`, `take_action_on_exfiltration_events`, `update_policy_*`
    - *Policy*: Agent NEVER invokes destructive tools autonomously. Any suggested remediation is staged as a question for the analyst.

---

## 4. EDR & Endpoint Security Telemetry

### 4.1 CrowdStrike Falcon
- **Primary Use**: EDR host inventory, real-time sensor detections, host network isolation status, and process execution trees.
- **API Endpoint**: `https://api.crowdstrike.com`
- **Methods Queried**:
  - `/devices/queries/devices/v1`: Search host by hostname, MAC, or IP
  - `/devices/entities/devices/v1`: Retrieve OS version, agent version, containment status
  - `/detects/queries/detects/v1`: Retrieve active detection IDs for the host
  - `/detects/entities/summaries/GET/v1`: Details of malicious process, parent process, command line, and MITRE tactics

### 4.2 Jamf Security Cloud
- **Primary Use**: Mobile Threat Defense (MTD), network threat prevention, zero-trust network access (ZTNA) risk scores.
- **API Endpoint**: `https://api.wandera.com/v1` (Jamf Security Cloud Radar API)
- **Investigative Signals**:
  - Device risk level (`SECURE`, `LOW`, `MEDIUM`, `HIGH`)
  - Compromised OS signals: Jailbreak / root detection, unauthorized OS tampering
  - Network threats: Blocked C2 outbound domains, phishing URLs accessed, man-in-the-middle Wi-Fi connections
  - Policy enforcement: Insecure Wi-Fi profiles, risky sideloaded configuration profiles

---

## 5. MDM & Device Compliance

### 5.1 Jamf Pro (Apple Fleet Management)
- **Primary Use**: macOS/iOS device management, FileVault 2 disk encryption verification, compliance profile status.
- **API Endpoint**: `https://{tenant}.jamfcloud.com/api/v1`
- **Queries**:
  - `/computers-inventory?filter=general.name=="{hostname}"`
  - Check `hardware.fileVault2Status`: `All Partitions Encrypted`
  - Check `security.sipStatus`: `Enabled`
  - Check `general.managed`: `true`

### 5.2 Microsoft Intune (Cross-Platform MDM)
- **Primary Use**: Windows and multi-OS device management, Azure AD registration, BitLocker encryption, compliance evaluation.
- **API Endpoint**: `https://graph.microsoft.com/v1.0/deviceManagement/managedDevices`
- **Filter Syntax**:
  - `$filter=userPrincipalName eq '{email}' or deviceName eq '{hostname}'`
- **Investigative Signals**:
  - `complianceState`: `compliant` vs `noncompliant`
  - `isEncrypted`: BitLocker / FileVault encryption status
  - `operatingSystem`: OS build and patch level
  - `ownership`: `company` (corporate-owned) vs `personal` (BYOD)

---

## 6. Email Security & Account Takeover (ATO)

### 6.1 Abnormal.AI (Abnormal Security)
- **Primary Use**: Phishing threat logs, business email compromise (BEC), compromised account cases, and mailbox forwarding rules.
- **API Endpoint**: `https://api.abnormalsecurity.com/v1`
- **Queries**:
  - `/threats?filter=recipientEmail eq '{email}'`: Check for credential harvesting or malware links delivered near the alert time
  - `/cases?filter=userEmail eq '{email}'`: Check for open Account Takeover (ATO) cases
  - `/account-takeover/abuse-mailbox`: Check for user-reported suspicious emails
- **Investigative Signals**:
  - Attack type: `Credential Phishing`, `Extortion`, `Invoice Fraud`, `Executive Impersonation`
  - Mailbox forwarding rules: Hidden inbox rules created to forward mail externally

---

## 7. External Threat Intelligence

### 7.1 CISA KEV (Known Exploited Vulnerabilities)
- **Feed**: `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`
- **Usage**: Automatically ingested during Phase 1 & 2 of the Threat Hunting agent. Filtered by active technologies discovered in the environment.

### 7.2 VirusTotal, ThreatFox, AlienVault OTX
- **VirusTotal**: Reputation score for IP/Domain/File (`/api/v3/ip_addresses/{ip}`, `/api/v3/files/{hash}`).
- **ThreatFox**: Active botnet C2 IPs and malware IOCs.
- **AlienVault OTX**: Threat pulses and community correlation.
