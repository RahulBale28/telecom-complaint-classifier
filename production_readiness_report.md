# Production Readiness Report — Telecom Complaint Classifier

## Summary

The LLM-based classifier using Claude Sonnet is **suitable for a limited supervised pilot** but is **not yet ready for full production automation**. This document outlines the risks identified and the improvements required before scaling.

---

## Current Performance

| Test | Zero-Shot | LLM (Claude Sonnet) |
|------|-----------|---------------------|
| Billing double-charge | ✅ Correct | ✅ Correct (0.88) |
| NBN connectivity outage | ✅ Correct | ✅ Correct (0.86) |
| App login failure | ❌ Wrong | ✅ Correct (0.84) |
| Overall accuracy | 66.7% | 100% |

---

## Risks

### 1. Hallucination Risk
The model may invent subcategories, urgency levels, or escalation decisions in ambiguous or multi-issue complaints. Evidence spans must be verbatim quotes — if the model paraphrases, it may introduce inaccurate attributions.

**Fix:** Provide a closed list of allowed subcategories. Instruct the model not to create free-form labels.

### 2. Over-Escalation Risk
Emotionally loaded language (e.g. "I'm furious", "this is unacceptable") may inflate urgency to HIGH or CRITICAL even when the actual issue is low priority.

**Fix:** Add deterministic escalation rules (e.g. "Escalate only if: full outage, safety risk, fraud, or imminent financial loss").

### 3. Data Privacy Risk
Evidence spans may inadvertently repeat PII from the customer message (e.g. account numbers, names). Even though `pii_detected` flags its presence, the raw PII could appear in the evidence_spans field.

**Fix:** Add a post-processing step to strip PII from evidence_spans before storing in Cosmos DB.

### 4. Multi-Issue Handling
When a complaint contains two or more distinct issues, the model may inconsistently assign category, urgency, or routing.

**Fix:** Instruct the model to classify based on the dominant actionable issue only.

### 5. No Deterministic Guardrails
Currently, urgency, escalation, and routing are determined entirely by model judgment. This is not suitable for automated production workflows.

**Fix:** Apply rule-based overrides after the LLM output (e.g. if category = OUTAGE and urgency = LOW, override to HIGH).

---

## Recommendations Before Production

- [ ] Add a closed subcategory ontology (no free-form creation)
- [ ] Add deterministic escalation logic as post-processing rules
- [ ] Strip PII from evidence_spans before storage
- [ ] Add fallback: if confidence < 0.70, route to human review queue
- [ ] Sample 5–10% of cases weekly for human quality review
- [ ] Run nightly evaluation pipeline against reviewed cases to detect drift

---

## Recommended Deployment Stages

| Stage | Traffic | Oversight |
|-------|---------|-----------|
| Stage 1 | 10% | Full human review of all outputs |
| Stage 2 | 50% | Spot-check escalations and billing cases |
| Stage 3 | 100% | Automated with weekly human sampling |

A **kill switch** in Azure App Configuration allows instant rollback to rules-only routing if issues are detected.
