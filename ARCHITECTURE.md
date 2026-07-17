# Care-Tech Implementation Architecture

**Status:** concept-to-MVP technical note  
**Prototype boundary:** the accompanying HTML files use synthetic data and local browser state. They do not connect to a model, clinician, health record, notification service, crisis line, or emergency service.

## 1. Product invariant

> **The model proposes. The consent engine governs. The human determines.**

The conversational model is not the source of authority for disclosure, safety-plan execution, clinical interpretation, or changes to consent. It produces typed candidate outputs. Deterministic application services validate those candidates against versioned policy and consent state.

## 2. Surfaces

### Client companion

The client-facing surface supports:

- private reflection;
- preparation of a summary candidate;
- explicit sharing or withholding;
- review of active consent scopes;
- review of safety-plan rules, recipients, exact disclosure, version, review date, and expiry;
- visible separation of self-report, extracted evidence, and model inference.

### Clinician dashboard

The clinician-facing surface displays only records that have passed consent and policy evaluation. It should never expose a raw model conversation by default.

Each visible record must carry:

- source type;
- evidence span or source reference;
- transformation type;
- model and prompt-policy version;
- consent scope ID and version;
- recipient;
- disclosure timestamp;
- limitations;
- review status.

## 3. Trust boundaries

```text
Browser / client app
        |
        v
API gateway + authenticated session
        |
        +--> Conversation service ------> Model provider
        |          |                         |
        |          |<-- typed candidate -----+
        |
        +--> Consent & policy engine
        |          |
        |          +--> allow / deny / redact / require confirmation
        |
        +--> Encrypted record store
        |
        +--> Audit ledger
        |
        +--> Notification adapter (disabled in prototype)
        |
        +--> Clinician dashboard projection
```

The model provider must not receive authorization to disclose data. The model output is treated as untrusted structured input.

## 4. Authoritative state

Consent, safety plans, recipients, and expiry dates belong in a database—not solely in prompts.

### Consent scope

```json
{
  "scope_id": "scope_sum_01J...",
  "client_id": "client_01J...",
  "data_class": "session_summary",
  "recipient_ids": ["clinician_01J..."],
  "status": "active",
  "effective_at": "2026-07-12T14:00:00Z",
  "expires_at": "2026-08-31T23:59:59Z",
  "confirmed_at": "2026-07-12T14:00:00Z",
  "version": 4,
  "conditions": {
    "include_full_transcript": false,
    "include_evidence_spans": false,
    "require_each_item_confirmation": false
  }
}
```

### Safety plan

```json
{
  "plan_id": "plan_01J...",
  "client_id": "client_01J...",
  "version": "3.2",
  "status": "active",
  "authored_by": ["client_01J...", "clinician_01J..."],
  "confirmed_at": "2026-07-14T15:30:00Z",
  "review_at": "2026-07-22T15:30:00Z",
  "expires_at": "2026-08-14T15:30:00Z",
  "rules": [
    {
      "rule_id": "rule_1",
      "signal_type": "authorized_phrase_match",
      "threshold": {"count": 1, "window_minutes": 0},
      "authorized_disclosure": [
        "exact_evidence_span",
        "rule_id",
        "event_timestamp",
        "session_id"
      ],
      "recipients": ["clinician_01J..."],
      "action": "create_human_review_event"
    }
  ]
}
```

## 5. Prompt layering

Prompts should be assembled server-side from independently versioned layers.

```text
SYSTEM CHARTER
- role and non-clinical boundaries
- prohibited claims
- response style

ORGANIZATION POLICY
- privacy defaults
- labeling requirements
- allowed output schema
- no authority to disclose or notify

CLIENT PREFERENCES
- conversational preferences
- active check-in preferences
- no authoritative consent logic

SESSION CONTEXT
- minimum necessary synthetic or authorized context
- structured, provenance-linked facts

USER MESSAGE
- current input
```

Consent state may be summarized for conversational transparency, but the prompt copy is not authoritative. The policy engine always reads the database record directly.

## 6. Typed model output

The model should return a candidate object validated against a strict schema.

```json
{
  "reply": "That call sounds like it left you carrying a lot.",
  "summary_candidate": {
    "text": "Jamie described a difficult call with a parent.",
    "basis": [
      {
        "type": "explicit_user_statement",
        "evidence_span_id": "span_01J..."
      }
    ],
    "clinical_claim": false
  },
  "observations": [
    {
      "label": "family conflict",
      "basis_type": "model_organized_theme",
      "confidence": 0.91,
      "evidence_span_ids": ["span_01J..."]
    }
  ],
  "safety_candidate": {
    "matched": false,
    "candidate_rule_ids": [],
    "evidence_span_ids": []
  },
  "requested_system_action": "none"
}
```

Reject responses that fail schema validation. Never parse disclosure instructions from free-form model prose.

## 7. Consent engine

The consent engine should be deterministic, testable, and independent of the model.

```text
function evaluate(candidate, client, now):
    active_scopes = load_active_scopes(client, now)
    active_plan = load_active_safety_plan(client, now)

    decision = new Decision(default = DENY)

    if candidate.summary_candidate exists:
        scope = find_scope(active_scopes, "session_summary")
        if scope allows automatic summary sharing:
            decision.add(ALLOW_SUMMARY, scope.version)
        else:
            decision.add(REQUIRE_CLIENT_CONFIRMATION)

    if candidate.safety_candidate.matched:
        for rule_id in candidate.safety_candidate.candidate_rule_ids:
            rule = active_plan.rule(rule_id)
            if rule exists and evidence satisfies deterministic rule:
                decision.add(
                    ALLOW_EXACT_DISCLOSURE,
                    fields = rule.authorized_disclosure,
                    recipients = rule.recipients,
                    plan_version = active_plan.version
                )

    return decision
```

The engine must default to deny when:

- consent is missing, expired, revoked, or ambiguous;
- the requested recipient is not authorized;
- the candidate cannot be traced to evidence;
- the plan version changed during evaluation;
- the requested disclosure exceeds the explicit field list.

## 8. Safety flow

```text
1. Model returns candidate match + evidence span IDs.
2. Server validates the output schema.
3. Policy engine loads the active plan by version.
4. Deterministic logic evaluates threshold and time window.
5. Server redacts everything outside the authorized disclosure fields.
6. Audit ledger records the decision and policy versions.
7. Human-review event is created.
8. A notification adapter may act only when explicitly configured,
   tested, legally reviewed, and enabled for that environment.
9. Clinician makes the clinical determination.
```

For early MVPs, keep external notification adapters disabled. Demonstrate the event lifecycle without implying that a real alert was sent.

## 9. Data classes

Treat these as separate data classes with distinct scopes:

- raw conversation transcript;
- exact evidence span;
- user-authored journal entry;
- structured self-report;
- generated summary;
- model-organized theme;
- model inference;
- safety-plan candidate event;
- clinician annotation;
- consent and policy metadata.

A summary scope must not silently imply transcript access. An evidence-span disclosure must not expose surrounding transcript unless the active scope explicitly permits it.

## 10. Provenance record

```json
{
  "record_id": "rec_01J...",
  "record_type": "generated_summary",
  "source_ids": ["session_01J..."],
  "evidence_span_ids": ["span_01J..."],
  "model": "provider/model-version",
  "prompt_policy_version": "policy-0.4.0",
  "generated_at": "2026-07-16T03:10:00Z",
  "consent_scope_id": "scope_sum_01J...",
  "consent_scope_version": 4,
  "recipient_ids": ["clinician_01J..."],
  "limitations": ["not_a_diagnosis", "human_review_required"]
}
```

## 11. Security baseline

Before any real-person pilot:

- threat-model the system and document trust boundaries;
- encrypt data in transit and at rest;
- use short-lived authenticated sessions;
- isolate tenants and roles;
- minimize data sent to model providers;
- prohibit secrets and model keys in browser code;
- validate all model output;
- encode or safely render every untrusted string;
- maintain append-only audit records with tamper evidence;
- support revocation, export, retention limits, and deletion workflows;
- prevent sensitive data from entering general application logs;
- complete independent privacy, security, accessibility, clinical, and legal review.

Do not present a prototype as compliant with any regulatory regime until qualified reviewers have evaluated the actual deployment, contracts, data flows, and operating procedures.

## 12. API sketch

### Conversation turn

```http
POST /v1/sessions/{session_id}/turns
Authorization: Bearer <short-lived token>
Content-Type: application/json
```

```json
{
  "message": "I had another difficult call with my parent.",
  "client_visible_context_version": 12
}
```

Response:

```json
{
  "turn_id": "turn_01J...",
  "reply": "That sounds difficult. What feels most important about it now?",
  "summary_candidate_id": "sum_01J...",
  "sharing_state": "private",
  "safety_event_state": "none",
  "provenance": {
    "model": "provider/model-version",
    "policy_version": "policy-0.4.0"
  }
}
```

### Share a summary

```http
POST /v1/summary-candidates/{summary_id}/share
```

```json
{
  "recipient_ids": ["clinician_01J..."],
  "scope_id": "scope_sum_01J...",
  "scope_version": 4
}
```

The server revalidates the scope at action time. It must not trust a browser-provided assertion that consent is active.

## 13. MVP sequence

### Phase A — static concept

- landing page;
- clinician dashboard;
- scripted companion preview;
- synthetic provenance and consent ledger;
- no model and no backend.

### Phase B — local model sandbox

- one conversation endpoint;
- strict output schema;
- synthetic personas only;
- no external notifications;
- consent engine with automatic tests;
- local audit ledger.

### Phase C — supervised closed pilot preparation

- identity and role system;
- encryption and tenant isolation;
- clinician workflow testing;
- incident response and rollback;
- qualified legal, privacy, security, clinical, and accessibility review;
- documented model-provider data handling;
- no real-person deployment until gates are passed.

## 14. Required tests

- expired consent denies disclosure;
- revoked consent denies disclosure immediately;
- summary scope never exposes a transcript;
- exact-excerpt safety scope never exposes surrounding text;
- recipient mismatch denies disclosure;
- stale plan version denies action;
- model hallucinated rule ID denies action;
- malformed model JSON is rejected;
- model prompt injection cannot alter consent state;
- every allowed disclosure creates an audit entry;
- every denied disclosure creates a reason code without logging sensitive content;
- keyboard-only and screen-reader workflows remain usable;
- day, night, and chocolate themes meet contrast requirements.

## 15. Product language

Prefer:

- “candidate match” over “risk detected”;
- “self-reported” over inferred certainty;
- “model-organized theme” over fact;
- “human review required” over “escalated”; 
- “simulated notification workflow” in prototypes;
- “exact authorized excerpt” over “full excerpt available on request.”

Avoid language that implies the companion is a clinician, understands more than its evidence supports, or has authority to widen disclosure.
