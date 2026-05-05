# 🤖 LLM-Powered Telecom Customer Complaint Classifier

> **Course:** BUS5001 – Cloud Platforms and Analytics | La Trobe University  
> **Tools:** Claude Sonnet (Anthropic API) · Prompt Engineering · Zero-Shot Classification · Azure OpenAI  
> **Author:** Rahul Narayan Bale | Student ID: 22380204

---

## 📌 Project Overview

This project designs and evaluates an **LLM-based complaint classification system** for a telecom company. Customer complaints arrive as free-text messages and need to be automatically categorised, prioritised, and routed to the right team — without human intervention.

The project compares two approaches:
- A **zero-shot classifier** (Hugging Face NLI-based model)
- An **LLM prompt** using **Claude Sonnet** (Anthropic API) with a structured JSON schema

---

## 🏗️ System Architecture

```
Customer Complaint (free-text input)
            │
            ▼
    ┌───────────────────┐        ┌─────────────────────────┐
    │  Zero-Shot Model  │        │   Claude Sonnet (LLM)   │
    │ (Hugging Face NLI)│        │   Anthropic API         │
    │                   │        │   /v1/messages endpoint  │
    └────────┬──────────┘        └───────────┬─────────────┘
             │                               │
             ▼                               ▼
     Single category label          12-field JSON output
     + probability scores           (structured, validated)
             │                               │
             └──────────┬────────────────────┘
                        ▼
              Comparison & Evaluation
                        │
                        ▼
            Findings + Production Recommendations
```

---

## 📊 Performance Comparison

| Metric | Zero-Shot Classifier | Claude Sonnet (LLM) |
|--------|---------------------|---------------------|
| Complaint 1 — Billing | ✅ Correct (score: 0.471) | ✅ Correct (confidence: 0.88) |
| Complaint 2 — Connectivity | ✅ Correct (score: 0.526) | ✅ Correct (confidence: 0.86) |
| Complaint 3 — Authentication | ❌ Wrong — predicted OTHER (0.291) | ✅ Correct (confidence: 0.84) |
| **Overall Accuracy** | **66.7% (2/3)** | **100% (3/3)** |
| **Improvement** | — | **+33%** |
| Provides urgency score | ❌ No | ✅ Yes |
| Provides routing team | ❌ No | ✅ Yes |
| Provides evidence spans | ❌ No | ✅ Yes |
| Detects PII | ❌ No | ✅ Yes |
| Provides confidence score | Partial (probabilities only) | ✅ Yes (0–1 float) |

---

## 🔧 The LLM Prompt (Production-Ready)

The improved prompt instructs Claude Sonnet to return **strict JSON only** with the following 12 fields:

```json
{
  "category": "<BILLING | CONNECTIVITY | AUTHENTICATION | OUTAGE | SPEED | ACCOUNT | PLAN_CHANGE | DEVICE | GENERAL_COMPLAINT | OTHER>",
  "subcategories": ["<short labels>"],
  "urgency": "<LOW | MEDIUM | HIGH | CRITICAL>",
  "requires_followup": "<Y or N>",
  "route_team": "<BILLING_TEAM | NOC_NETWORK | APP_AUTH | FIELD_TECH | T1_GENERAL | ACCOUNT_SERVICES>",
  "escalate_now": "<Y or N>",
  "customer_sentiment": "<POSITIVE | NEUTRAL | NEGATIVE>",
  "rationale": "<max 200 characters>",
  "evidence_spans": ["<exact quotes from the message>"],
  "pii_detected": "<Y or N>",
  "safety_flags": ["<NONE | SELF_HARM | THREAT | ABUSE | OTHER>"],
  "confidence": 0.00
}
```

### Key Prompt Rules
- Use **message content only** — no hallucination
- Urgency is based on **service impact** (outages, financial harm, work impact = HIGH/CRITICAL)
- `escalate_now = Y` only for CRITICAL, monetary loss, fraud, or safety risk
- Evidence spans must be **exact quotes** from the customer message
- `pii_detected = Y` only if names, addresses, phone numbers, emails, or account numbers appear

---

## 🧪 Test Cases and Outputs

### Complaint 1 — Billing Double Charge
**Input:**
> "I've been charged twice for the same service. This is really frustrating and I want a refund."

**LLM Output:**
```json
{
  "category": "BILLING",
  "subcategories": ["DOUBLE_CHARGE", "REFUND_REQUEST"],
  "urgency": "HIGH",
  "requires_followup": "Y",
  "route_team": "BILLING_TEAM",
  "escalate_now": "Y",
  "customer_sentiment": "NEGATIVE",
  "rationale": "Double charge with explicit refund request indicates monetary harm.",
  "evidence_spans": ["charged twice", "want a refund"],
  "pii_detected": "N",
  "safety_flags": ["NONE"],
  "confidence": 0.88
}
```

**Zero-Shot Result:** BILLING (0.471) — correct but with significant probability leaking to GENERAL_COMPLAINT (0.123) and OTHER (0.115)

---

### Complaint 2 — NBN Outage
**Input:**
> "My NBN connection has been down since last night. I work from home and can't wait on the line forever."

**LLM Output:**
```json
{
  "category": "CONNECTIVITY",
  "subcategories": ["NBN_DOWN", "SERVICE_UNAVAILABLE"],
  "urgency": "CRITICAL",
  "requires_followup": "Y",
  "route_team": "NOC_NETWORK",
  "escalate_now": "Y",
  "customer_sentiment": "NEGATIVE",
  "rationale": "Prolonged outage impacting work implies critical priority and NOC routing.",
  "evidence_spans": ["NBN connection has been down", "work from home"],
  "pii_detected": "N",
  "safety_flags": ["NONE"],
  "confidence": 0.86
}
```

**Zero-Shot Result:** CONNECTIVITY (0.526) — correct but OUTAGE also scored 0.192, showing ambiguity

---

### Complaint 3 — Login Failure
**Input:**
> "Why can't I log into my app even after resetting the password three times?"

**LLM Output:**
```json
{
  "category": "AUTHENTICATION",
  "subcategories": ["LOGIN_FAILURE", "PASSWORD_RESET_ISSUE"],
  "urgency": "MEDIUM",
  "requires_followup": "Y",
  "route_team": "APP_AUTH",
  "escalate_now": "N",
  "customer_sentiment": "NEGATIVE",
  "rationale": "Repeated password resets and continued login failure indicate app auth issue.",
  "evidence_spans": ["can't I log into my app", "resetting the password three times"],
  "pii_detected": "N",
  "safety_flags": ["NONE"],
  "confidence": 0.84
}
```

**Zero-Shot Result:** OTHER (0.291) — ❌ **incorrect**. Failed to link "log into my app" and "resetting the password" to an authentication issue.

---

## ⚠️ Production Risks Identified

| Risk | Description |
|------|-------------|
| Hallucination | Model may invent subcategories or urgency levels in ambiguous cases |
| Over-escalation | Emotional language (e.g. "frustrated") may trigger unnecessary HIGH urgency |
| Data Privacy | Evidence spans may inadvertently echo PII from the message |
| Multi-issue complaints | Messages with 2+ issues may be inconsistently classified |
| No deterministic guardrails | Urgency and escalation rely on model judgment, not fixed rules |

---

## 🚀 Recommended Azure Deployment Architecture

```
Email / Chat Input
        │
        ▼
Azure API Management (Front Door)
        │
        ▼
Azure Service Bus Queue (sbq-incoming)
        │
        ▼
Azure Functions — Pre-processing
(Normalise text, detect/redact PII, add message_id)
        │
        ▼
Azure Functions — LLM Classification
(Call Claude Sonnet via Anthropic API)
        │
        ▼
Azure Cosmos DB (Store structured triage output)
        │
        ├──► If escalate_now = Y → Azure DevOps / ServiceNow ticket
        └──► Otherwise → Route to team queue via Teams / email
        │
        ▼
Power BI Dashboard
(Volume, top categories, SLA, confidence distribution)
```

### Deployment Strategy
- **Stage 1:** 10% traffic → supervised review
- **Stage 2:** 50% traffic → spot-check escalations
- **Stage 3:** 100% traffic → full automation
- **Kill switch:** Toggle in Azure App Configuration to revert to rules-only routing

---

## 📁 Repository Structure

```
telecom-complaint-classifier/
│
├── README.md                            ← You are here
│
├── prompt/
│   └── production_prompt.md            ← Full production-ready prompt template
│
├── test-cases/
│   ├── complaint1_billing.json         ← Input + LLM output for billing case
│   ├── complaint2_connectivity.json    ← Input + LLM output for connectivity case
│   └── complaint3_authentication.json  ← Input + LLM output for auth case
│
├── screenshots/
│   ├── 01_zero_shot_complaint1.png     ← Zero-shot classifier results for complaint 1
│   ├── 02_zero_shot_complaint2.png     ← Zero-shot classifier results for complaint 2
│   ├── 03_zero_shot_complaint3.png     ← Zero-shot classifier results for complaint 3
│   ├── 04_llm_output_complaint1.png    ← LLM JSON output for complaint 1
│   ├── 05_llm_output_complaint2.png    ← LLM JSON output for complaint 2
│   ├── 06_llm_output_complaint3.png    ← LLM JSON output for complaint 3
│   └── 07_azure_architecture.png      ← Azure deployment architecture diagram
│
└── docs/
    └── production_readiness_report.md  ← Risks, recommendations, and readiness assessment
```

---

## 📸 Screenshots

### Zero-Shot Classifier — Complaint 3 (Incorrect Result)
![Zero Shot Complaint 3](screenshots/03_zero_shot_complaint3.png)

### LLM Output — Complaint 3 (Correct Result)
![LLM Output Complaint 3](screenshots/06_llm_output_complaint3.png)

---

## 💡 Key Takeaways

- LLMs significantly outperform simple zero-shot classifiers on **contextual and implicit complaints** (e.g. authentication issues described without using the word "authentication")
- Structured JSON prompts with strict schema enforcement produce **consistent, auditable outputs**
- Production deployment requires **deterministic guardrails** on escalation logic to prevent over-automation
- A **staged rollout** with human oversight is essential before full automation

---

## 🛠️ Technologies Used

| Technology | Purpose |
|------------|---------|
| Claude Sonnet (claude-sonnet-4-20250514) | LLM classification via Anthropic API |
| Hugging Face Zero-Shot (NLI) | Baseline classifier for comparison |
| Azure OpenAI / Azure Functions | Deployment architecture |
| Azure Service Bus | Message queue for decoupling |
| Azure Cosmos DB | Structured triage output storage |
| Power BI | Dashboard for monitoring and reporting |

---

## 👤 About

**Rahul Narayan Bale**  
Master of Business Analytics — La Trobe University, Melbourne  
📧 rahulbale2804@gmail.com  
🔗 [GitHub Profile](https://github.com/RahulBale28)

---

> *This project was completed as part of BUS5001 – Cloud Platforms and Analytics at La Trobe University.*
