# Building an Autonomous AI Agent for P&C Claims Processing in n8n

> © 2026 Srinivasan Sankar · @AgentisIQ · All Rights Reserved.  
> Unauthorized reproduction, redistribution, or modification of this document is strictly prohibited.

---

## PART 1 — Scoping the Agent

### What This Agent Does
The agent receives a new insurance claim (via webhook, email, or form), extracts structured data, classifies the claim, assesses damage/risk using AI, routes it to the right handler, and updates your system of record — all autonomously.

### Inputs
- Claim submission (JSON from portal, email, or web form)
- Supporting documents (PDFs, images of damage)
- Claimant profile (from CRM or policy database)
- Historical claims data (for fraud scoring)

### Outputs
- Structured claim record (saved to DB / Google Sheets / CRM)
- AI-generated claim summary + severity score
- Routing decision (auto-approve / adjuster review / fraud flag)
- Notification to claimant + internal team (email / Slack)
- Audit log entry

---

## PART 2 — High-Level Architecture

```
[TRIGGER]
   │
   ▼
[INTAKE & VALIDATION]
   │
   ▼
[DOCUMENT EXTRACTION] ──► (OCR / Vision API)
   │
   ▼
[AI CLAIM ANALYSIS] ──► (OpenAI GPT-4o)
   │
   ▼
[FRAUD SCORING] ──► (ML API or Rules Engine)
   │
   ▼
[ROUTING DECISION]
   │
   ├──► AUTO-APPROVE ──► [Pay & Notify]
   ├──► ADJUSTER QUEUE ──► [Assign + Notify]
   └──► FRAUD FLAG ──► [Alert + Freeze]
   │
   ▼
[STORE CLAIM RECORD]
   │
   ▼
[AUDIT LOG + NOTIFICATIONS]
```

---

## PART 3 — Node-by-Node Walkthrough

---

### NODE 1 — Webhook Trigger (Claim Intake)

**Purpose:** Receives the new claim submission from your portal or external system.

**Node Type:** `Webhook`

**Configuration:**
```
HTTP Method: POST
Path: /claims/new
Response Mode: "Last Node" (so we can return a confirmation)
Authentication: Header Auth (use a secret token)
```

**Expected incoming payload:**
```json
{
  "claim_id": "CLM-2024-00847",
  "claimant_name": "Jane Doe",
  "policy_number": "POL-99234",
  "incident_date": "2024-12-01",
  "incident_type": "auto_collision",
  "description": "Rear-end collision on Highway 101. Significant trunk damage.",
  "documents": ["https://storage.co/photos/CLM-2024-00847/photo1.jpg"],
  "estimated_damage": 8400
}
```

---

### NODE 2 — Set Node (Normalize & Enrich Claim Object)

**Purpose:** Standardize the incoming payload and add metadata fields before processing.

**Node Type:** `Set`

**Configuration:**
```
claim_id:         {{ $json.claim_id }}
received_at:      {{ new Date().toISOString() }}
status:           "processing"
incident_type:    {{ $json.incident_type }}
description:      {{ $json.description }}
claimant_name:    {{ $json.claimant_name }}
policy_number:    {{ $json.policy_number }}
documents:        {{ $json.documents }}
estimated_damage: {{ $json.estimated_damage }}
```

---

### NODE 3 — HTTP Request (Fetch Policy Details)

**Purpose:** Pull the claimant's policy data from your internal API or database to validate coverage.

**Node Type:** `HTTP Request`

**Configuration:**
```
Method: GET
URL: https://your-policy-api.com/policies/{{ $json.policy_number }}
Auth: Bearer Token (add in n8n Credentials)
Headers:
  Content-Type: application/json
```

**What to extract from response (via Set node after):**
```
coverage_limit:    {{ $json.coverage_limit }}
deductible:        {{ $json.deductible }}
policy_active:     {{ $json.is_active }}
covered_incidents: {{ $json.covered_types }}
```

---

### NODE 4 — IF Node (Policy Validation Check)

**Purpose:** Immediately reject invalid or inactive policies before wasting AI calls.

**Node Type:** `IF`

**Condition:**
```
{{ $json.policy_active }} equals true
AND
{{ $json.covered_incidents }} contains {{ $json.incident_type }}
```

- **True path →** Continue to AI Analysis  
- **False path →** Route to rejection handler (send email + log)

---

### NODE 5 — HTTP Request (Document OCR/Vision Extraction)

**Purpose:** Extract text and damage information from uploaded photos/PDFs.

**Node Type:** `HTTP Request` → calls OpenAI Vision or Google Document AI

**Using OpenAI Vision (GPT-4o):**

```
Method: POST
URL: https://api.openai.com/v1/chat/completions
Auth: Bearer (OpenAI API Key in n8n Credentials)
```

**Request Body (JSON):**
```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "You are an insurance claims assessor. Analyze this damage photo and extract: 1) Type of damage 2) Estimated severity (low/medium/high/total-loss) 3) Visible damage description 4) Any suspicious indicators. Return as JSON."
        },
        {
          "type": "image_url",
          "image_url": { "url": "{{ $json.documents[0] }}" }
        }
      ]
    }
  ],
  "max_tokens": 800
}
```

**Parse the response in a Code node immediately after:**
```javascript
const content = $input.first().json.choices[0].message.content;
// Strip markdown code fences if present
const cleaned = content.replace(/```json|```/g, '').trim();
const parsed = JSON.parse(cleaned);

return [{
  json: {
    ...$input.first().json,
    vision_damage_type: parsed.damage_type,
    vision_severity: parsed.severity,
    vision_description: parsed.description,
    vision_flags: parsed.suspicious_indicators
  }
}];
```

---

### NODE 6 — OpenAI Node (Core Claim Analysis)

**Purpose:** The brain of the agent — performs full AI analysis, generates a claim summary, severity score, and routing recommendation.

**Node Type:** `OpenAI` (built-in n8n node) or HTTP Request

**System Prompt:**
```
You are a senior P&C insurance claims analyst AI. Your job is to:
1. Analyze claim details and document findings
2. Generate a professional claim summary (2-3 sentences)
3. Assign a severity score 1-10 (10 = most severe)
4. Estimate a fair payout range based on the policy and damage
5. Identify any fraud indicators (score 0-100, 100 = highly suspicious)
6. Recommend routing: "AUTO_APPROVE", "ADJUSTER_REVIEW", or "FRAUD_INVESTIGATION"

Routing rules:
- AUTO_APPROVE: severity ≤ 4, fraud_score < 20, damage ≤ $5,000
- FRAUD_INVESTIGATION: fraud_score ≥ 60
- ADJUSTER_REVIEW: everything else

Always respond in valid JSON only. No extra text.
```

**User Message (dynamic):**
```
Claim ID: {{ $json.claim_id }}
Incident Type: {{ $json.incident_type }}
Claimant Description: {{ $json.description }}
Estimated Damage: ${{ $json.estimated_damage }}
Coverage Limit: ${{ $json.coverage_limit }}
Deductible: ${{ $json.deductible }}
Vision Analysis: {{ $json.vision_description }}
Severity from Vision: {{ $json.vision_severity }}
Suspicious Flags: {{ $json.vision_flags }}
Incident Date: {{ $json.incident_date }}
```

**Expected JSON response:**
```json
{
  "summary": "Claimant reports rear-end collision causing significant trunk damage...",
  "severity_score": 6,
  "fraud_score": 12,
  "estimated_payout_min": 7200,
  "estimated_payout_max": 8800,
  "routing": "ADJUSTER_REVIEW",
  "notes": "Damage aligns with described incident. No major fraud indicators."
}
```

**Parse in Code node after:**
```javascript
const raw = $input.first().json.message?.content || 
            $input.first().json.choices?.[0]?.message?.content;
const cleaned = raw.replace(/```json|```/g, '').trim();
const ai = JSON.parse(cleaned);

return [{
  json: {
    ...$input.first().json,
    ai_summary: ai.summary,
    severity_score: ai.severity_score,
    fraud_score: ai.fraud_score,
    payout_min: ai.estimated_payout_min,
    payout_max: ai.estimated_payout_max,
    routing: ai.routing,
    ai_notes: ai.notes
  }
}];
```

---

### NODE 7 — Switch Node (Routing Decision)

**Purpose:** Branch the workflow based on the AI's routing recommendation.

**Node Type:** `Switch`

**Configuration:**
```
Value to switch on: {{ $json.routing }}

Route 1: AUTO_APPROVE
Route 2: ADJUSTER_REVIEW
Route 3: FRAUD_INVESTIGATION
Default: ADJUSTER_REVIEW
```

---

### NODE 8A — AUTO_APPROVE Branch

**8A-1: Code Node — Calculate final payout**
```javascript
const deductible = $json.deductible || 0;
const midpoint = ($json.payout_min + $json.payout_max) / 2;
const final_payout = Math.max(0, midpoint - deductible);

return [{ json: { ...$json, final_payout, approval_status: "APPROVED" } }];
```

**8A-2: HTTP Request — Trigger payment in your claims system**
```
Method: POST
URL: https://your-claims-system.com/api/payments
Body:
{
  "claim_id": "{{ $json.claim_id }}",
  "amount": {{ $json.final_payout }},
  "policy": "{{ $json.policy_number }}"
}
```

**8A-3: Send Email — Notify claimant**
```
To:      {{ $json.claimant_email }}
Subject: Your Claim {{ $json.claim_id }} Has Been Approved
Body:
Hi {{ $json.claimant_name }}, your claim has been approved.
Payout of ${{ $json.final_payout }} will be processed within 2 business days.
Summary: {{ $json.ai_summary }}
```

---

### NODE 8B — ADJUSTER_REVIEW Branch

**8B-1: HTTP Request — Create ticket in your adjuster system (e.g. Jira, ServiceNow, or Salesforce)**
```json
{
  "title": "Claim Review: {{ $json.claim_id }}",
  "priority": "{{ $json.severity_score > 7 ? 'HIGH' : 'NORMAL' }}",
  "description": "{{ $json.ai_summary }}",
  "estimated_payout": "{{ $json.payout_min }} - {{ $json.payout_max }}",
  "assigned_queue": "claims_adjusters"
}
```

**8B-2: Slack Node — Alert adjuster team**
```
Channel: #claims-queue
Message:
🔍 *New Claim for Review*
Claim: {{ $json.claim_id }} | Severity: {{ $json.severity_score }}/10
Claimant: {{ $json.claimant_name }}
Est. Payout: ${{ $json.payout_min }}–${{ $json.payout_max }}
AI Notes: {{ $json.ai_notes }}
```

---

### NODE 8C — FRAUD_INVESTIGATION Branch

**8C-1: Set Node — Flag the claim**
```
fraud_flagged:        true
freeze_payment:       true
investigation_status: "OPEN"
```

**8C-2: Slack Node — Immediate alert to fraud team**
```
Channel: #fraud-alerts
Message:
🚨 *FRAUD ALERT — Claim {{ $json.claim_id }}*
Fraud Score: {{ $json.fraud_score }}/100
Claimant: {{ $json.claimant_name }}
Vision Flags: {{ $json.vision_flags }}
Immediate review required.
```

**8C-3: HTTP Request — Freeze claim in system and log for SIU**

---

### NODE 9 — Google Sheets / Database (Store Claim Record)

**Purpose:** Persist every claim with full AI analysis for audit, reporting, and retraining.

**Node Type:** `Google Sheets` (or `Postgres`, `Airtable`, `Supabase`)

**Google Sheets config:**
```
Operation: Append Row
Sheet: Claims_Log

Column Mapping:
  A: claim_id
  B: received_at
  C: claimant_name
  D: policy_number
  E: incident_type
  F: severity_score
  G: fraud_score
  H: routing
  I: final_payout (or "PENDING")
  J: ai_summary
  K: status
  L: vision_severity
```

---

### NODE 10 — Error Handling (On Error Branch)

**Step 1:** On any node → go to **Settings → "On Error"** and set to **"Continue (using error output)"**

**Step 2:** Add an **Error Trigger** at the workflow level.

**Step 3: Code node to format the error:**
```javascript
const error = $input.first().json;
return [{
  json: {
    workflow: "P&C Claims Agent",
    error_message: error.error?.message || "Unknown error",
    node: error.node?.name,
    claim_id: error.execution?.data?.claim_id || "UNKNOWN",
    timestamp: new Date().toISOString()
  }
}];
```

**Step 4:** Route to Slack error channel + log to a Google Sheets error tab.

---

### NODE 11 — Retry Logic via Wait + Loop

For external API calls (OpenAI, policy API), add retry logic with a Code node:

```javascript
const retryCount = $json.retry_count || 0;

if (retryCount >= 3) {
  throw new Error(`Max retries reached for claim ${$json.claim_id}`);
}

return [{
  json: {
    ...$json,
    retry_count: retryCount + 1
  }
}];
```

Then use a **Wait node** (30 seconds) before looping back to the HTTP Request node via an **IF** check.

---

## PART 4 — Memory & Data Persistence

### Short-Term Memory (Within Execution)
n8n automatically passes data between nodes. Use **Set nodes** liberally to carry all needed fields forward.

### Cross-Execution Memory Options

**Option A: Redis via HTTP**
```
Use case: Store fraud patterns, adjuster notes, past decisions per policy number.
Node: HTTP Request → Redis REST API
Key:   "claim_history:{{ $json.policy_number }}"
Value: JSON.stringify(claimRecord)
```

**Option B: Pinecone / Qdrant (Vector DB for semantic search)**
```
Use case: Find similar past claims to benchmark payout fairness.

Step 1 — Embed the claim description:
  POST https://api.openai.com/v1/embeddings
  Body: { "input": "{{ $json.description }}", "model": "text-embedding-3-small" }

Step 2 — Query Pinecone for top 5 similar claims:
  POST https://your-index.pinecone.io/query
  Body: { "vector": [embedding], "topK": 5, "includeMetadata": true }

Step 3 — Feed similar claims into GPT prompt for grounded payout estimation.
```

**Option C: Google Sheets as persistent store** — simplest approach, already covered in Node 9.

---

## PART 5 — Authentication Setup for External APIs

| Service | Auth Type | n8n Setup |
|---|---|---|
| OpenAI | API Key | Credentials → OpenAI API → paste key |
| Google Sheets | OAuth2 | Credentials → Google Sheets OAuth2 |
| Slack | OAuth2 | Credentials → Slack OAuth2 |
| Policy Internal API | Bearer Token | Credentials → Header Auth → `Authorization: Bearer <token>` |
| Pinecone | API Key | Credentials → HTTP Header Auth → `Api-Key: <key>` |
| Salesforce/CRM | OAuth2 | Credentials → Salesforce OAuth2 |

> **Pro tip:** Store all secrets in n8n's **Credentials vault** — never hardcode in node settings or code.

---

## PART 6 — Complete Workflow Summary

```
[Webhook: POST /claims/new]
        │
[Set: Normalize fields]
        │
[HTTP: Fetch Policy from API]
        │
[IF: Policy valid + covered?]
   │              │
[YES]           [NO → Email Rejection + Log]
   │
[HTTP: OpenAI Vision → Damage Analysis]
        │
[Code: Parse Vision JSON]
        │
[OpenAI: Full Claim Analysis]
        │
[Code: Parse AI JSON]
        │
[Switch: routing field]
   │           │           │
[AUTO]    [ADJUSTER]  [FRAUD]
   │           │           │
[Calc      [CRM Ticket]  [Freeze +
 Payout]   [Slack Alert]  Fraud Alert]
[Pay API]
[Email]
   │
[ALL PATHS MERGE]
        │
[Google Sheets: Log Claim]
        │
[Webhook Response: 200 OK + claim_id]
```

---

## PART 7 — Making the Agent More Powerful

### 🔗 Chain Sub-Workflows
Split into modular workflows and call them via the **Execute Workflow** node:
- `WF: Document Extraction`
- `WF: Fraud Analysis`
- `WF: Payout Calculation`
- `WF: Notification Dispatch`

This makes each piece independently testable, reusable, and scalable.

### 📡 Inbound Webhooks from Multiple Channels
- **Email trigger** (via IMAP node) → parse claims submitted by email
- **Twilio SMS webhook** → claimants can text "FILE CLAIM"
- **WhatsApp via Twilio** → photo + description via chat
- **Mobile app** → direct webhook POST

### 🧠 Add a Persistent AI Memory Layer
Use **Supabase + pgvector** or **Pinecone** to:
- Retrieve past claims by same claimant (detect serial filers)
- Benchmark payouts against similar historical claims
- Feed retrieval-augmented context into GPT for better decisions

### 📊 Build a Live Dashboard
- Connect Google Sheets to **Looker Studio** or push to **Supabase** for a real-time claims dashboard
- Track: avg resolution time, fraud rate, auto-approval rate, payout accuracy

### 🤖 Add a Human-in-the-Loop Step
```
1. Send adjuster an email with Approve/Reject links containing unique tokens
2. Each link hits a different webhook path
3. Wait node pauses workflow until adjuster responds
4. Resume workflow based on the adjuster's decision
```

### 🔄 Scheduled Reprocessing
Use a **Cron trigger** to:
- Re-run stale claims stuck in ADJUSTER_REVIEW > 48 hours
- Generate daily fraud pattern summary reports
- Sync claim statuses from external systems

### 🛡️ Add PII Masking Before AI Calls
```javascript
// Code node — run BEFORE any OpenAI call
const masked = {
  ...$json,
  claimant_name: "REDACTED",
  policy_number: $json.policy_number.replace(/.(?=.{4})/g, '*')
};
return [{ json: masked }];
```

---

## Quick-Start Checklist

- [ ] Set up n8n instance (cloud or self-hosted)
- [ ] Configure OpenAI API credentials
- [ ] Configure Google Sheets OAuth2
- [ ] Configure Slack OAuth2
- [ ] Build and test Node 1–4 (intake + validation loop)
- [ ] Add OpenAI Vision (Node 5) with a real damage photo
- [ ] Add core GPT-4o analysis (Node 6) and verify JSON output
- [ ] Build all three routing branches (Node 8A/8B/8C)
- [ ] Set up Google Sheets claim log (Node 9)
- [ ] Add error handling to every HTTP/AI node (Node 10)
- [ ] Test end-to-end with a mock claim payload
- [ ] Add retry logic for flaky external APIs (Node 11)
- [ ] Harden with PII masking before production

---

*Start by building the core 5-node loop (Webhook → Validate → OpenAI Analysis → Switch → Log), verify it works end to end with test data, then layer in the branches, error handling, and memory. Each section above is independently buildable and testable — treat it like assembling modules, not one big monolith.*

---

<br/>

---

> © 2026 Srinivasan Sankar · @AgentisIQ · All Rights Reserved.  
> This document is proprietary and confidential. No part of this publication may be reproduced, distributed, or transmitted in any form or by any means — including photocopying, recording, or other electronic or mechanical methods — without the prior written permission of the author, except in the case of brief quotations embodied in critical reviews and certain other noncommercial uses permitted by copyright law.  
> For permissions requests, contact: **@AgentisIQ**
