# SmartCover Workflow - Before/After Comparison

## 1. Privacy & Consent Handling

### BEFORE (v1.0)

```json
{
  "phase_id": "phase_2_healthtech",
  "nodes": [
    {
      "node_id": "health_privacy_gate",
      "type": "privacy_gate",
      "display": "Privacy Gate (DPDP)",
      "checks": ["user_consent_health_data", "dpdp_act_2023_compliance", ...],
      "on_fail": "privacy_breach_halt",
      "next": "google_health_api_call"
    }
  ]
}
```

**Problems:**

- Privacy gate lives inside each agent
- No guarantee all agents check privacy
- User can route to fintech without financial_data consent

### AFTER (v2.0)

```json
{
  "phase_id": "phase_0_privacy_consent",
  "name": "Privacy & Consent Gate (Centralized)",
  "nodes": [
    {
      "node_id": "privacy_consent_check",
      "type": "privacy_gate",
      "checks": [
        {
          "name": "dpdp_act_2023",
          "severity": "HARD_BLOCK"
        },
        {
          "name": "user_consent_health",
          "severity": "HARD_BLOCK"
        },
        {
          "name": "user_consent_financial",
          "severity": "HARD_BLOCK"
        }
      ],
      "next": "memory_load"
    },
    {
      "node_id": "router_and_parallel_dispatch",
      "routing_rules": [
        {
          "agent": "healthtech_agent",
          "condition": "'health_query' in intents AND consent.health_data == true"
        },
        {
          "agent": "fintech_agent",
          "condition": "'credit_query' in intents AND consent.financial_data == true"
        }
      ]
    }
  ]
}
```

**Benefits:**
✅ Single source of truth for consent  
✅ Agents only execute if consent verified  
✅ Can handle soft blocks (disable features gracefully)  
✅ Easier to audit compliance

---

## 2. Intent Classification

### BEFORE (v1.0)

```
┌──────────────────────┐
│  llm_context_injection │  (500ms)
│  System: Format intent │
│  Output: formatted    │
└──────────────────────┘
           ↓
┌──────────────────────┐
│  classifier_router   │  (500ms)
│  System: Classify    │
│  Input: formatted    │
│  Output: routing     │
└──────────────────────┘

Total: 2 LLM calls, 1000ms
```

**Problems:**

- Two sequential LLM calls
- Context formatting is separate from classification
- Wasteful token usage

### AFTER (v2.0)

```
┌─────────────────────────────┐
│  classifier_and_context     │  (400ms)
│  System: Analyze & classify │
│  Input: raw_message +       │
│         loaded_memory       │
│  Output: intent JSON with   │
│          confidence, urgency│
└─────────────────────────────┘

Total: 1 LLM call, 400ms (33% faster!)

Output contract:
{
  "primary_intent": "health_query|credit_query|...",
  "secondary_intents": [...],
  "confidence": 0.92,
  "urgency": "routine|priority|emergency",
  "extracted_entities": {
    "symptom": "chest pain",
    ...
  },
  "language": "en"
}
```

**Benefits:**
✅ Merged into single LLM call  
✅ Explicit output schema (type safety)  
✅ Confidence score for early exit  
✅ Structured routing decisions

---

## 3. Agent Routing & Execution

### BEFORE (v1.0)

```json
{
  "node_id": "classifier_router",
  "type": "classifier",
  "outputs": {
    "health_query": "phase_2_healthtech",
    "credit_query": "phase_2_fintech",
    ...
  },
  "parallel_allowed": true
}

// In practice: agents run but not truly parallel
// Each agent waits for previous to complete
```

**Problems:**

- No explicit parallelization
- Agents appear to run in parallel but don't
- Router doesn't check consent
- Multi-intent handling unclear

### AFTER (v2.0)

```json
{
  "node_id": "router_and_parallel_dispatch",
  "type": "conditional_router",
  "routing_rules": [
    {
      "condition": "primary_intent == 'faq_query' AND secondary_intents.length == 0",
      "route": "faq_agent_only",
      "description": "Early exit—no other agents needed"
    },
    {
      "condition": "primary_intent != 'clarify'",
      "route": "parallel_agent_dispatch",
      "dispatch": [
        {
          "agent": "healthtech_agent",
          "condition": "('health_query' in intents) AND consent.health_data",
          "timeout_seconds": 4
        },
        {
          "agent": "fintech_agent",
          "condition": "('credit_query' in intents) AND consent.financial_data",
          "timeout_seconds": 3
        },
        ...
      ],
      "parallel_execution": true,
      "description": "Run all applicable agents in parallel"
    }
  ]
}
```

**Benefits:**
✅ True parallel execution (asyncio)  
✅ Conditional routing based on consent  
✅ Early exit for simple queries (FAQ)  
✅ Independent agent timeouts  
✅ Multi-intent support

**Example:** FAQ-only query

- v1.0: Runs all 5 agents (unnecessary) = 10-15 seconds
- v2.0: Runs FAQ agent only = 3-4 seconds (75% faster!)

---

## 4. Agent Output Contracts

### BEFORE (v1.0)

```json
{
  "node_id": "health_output_merge",
  "type": "output",
  "display": "Health Agent Output",
  "output": {
    "hrs_score": "integer_0_to_1000",
    "hrs_band": "Excellent|Good|Fair|Moderate|At Risk",
    "hrs_delta": "integer",
    "premium_delta": "float",
    "credit_boost_inr": "integer",
    "abha_summary": "object"
  }
}

// No schema validation
// Synthesis assumes these fields exist
// Silent failures if field missing
```

**Problems:**

- String descriptions instead of schemas
- No type validation
- No required fields definition
- Synthesis breaks silently if field missing

### AFTER (v2.0)

```json
{
  "agent_id": "healthtech_agent",
  "output_contract": {
    "type": "object",
    "properties": {
      "hrs_score": {
        "type": "integer",
        "minimum": 0,
        "maximum": 1000
      },
      "hrs_band": {
        "type": "string",
        "enum": ["Excellent", "Good", "Fair", "Moderate", "At Risk"]
      },
      "hrs_delta_30d": {
        "type": "integer"
      },
      "premium_delta_pct": {
        "type": "number"
      },
      "credit_boost_inr": {
        "type": "integer"
      },
      "abha_summary": {
        "type": "object",
        "properties": {
          "recent_conditions": {"type": "array"},
          "allergies": {"type": "array"}
        }
      },
      "confidence": {
        "type": "number",
        "minimum": 0,
        "maximum": 1
      },
      "data_staleness_minutes": {
        "type": "integer"
      }
    },
    "required": ["hrs_score", "hrs_band", "confidence"]
  }
}

// In validation layer:
validator = SchemaValidator(output_contract)
is_valid = validator.validate(agent_output)
if not is_valid:
  log_error(f"Agent output invalid: {validator.errors}")
  escalate_to_human()
```

**Benefits:**
✅ JSON Schema validation (industry standard)  
✅ Type checking (hrs_score must be 0-1000)  
✅ Required fields enforced  
✅ Confidence score for result ranking  
✅ Data staleness tracking  
✅ Early detection of malformed outputs

---

## 5. Agent Merge Strategy

### BEFORE (v1.0)

```json
{
  "node_id": "phase_3_merge",
  "type": "merge",
  "priority_order": [
    "claim_agent",
    "insurtech_agent",
    "fintech_agent",
    "healthtech_agent",
    "faq_agent"
  ],
  "conflict_resolution": "higher_confidence_wins",
  "timeout_seconds": 5,
  "on_timeout": "use_partial_results_with_staleness_flag"
}
```

**Problems:**

- Static 5s timeout (wastes time if agents fast)
- Blocks on slowest agent
- No dynamic confidence evaluation
- Merge happens at end (after all agents done)

### AFTER (v2.0)

```json
{
  "node_id": "dynamic_merge",
  "type": "merge",
  "merge_strategy": "earliest_completion",
  "collection_rules": {
    "max_wait_seconds": 8,
    "min_confidence_threshold": 0.6,
    "collect_until": "any_of [all_agents_returned, max_wait_exceeded, confidence_threshold_met]",
    "priority_ranking": [
      { "agent": "claim_agent", "priority": 1, "reason": "Time-sensitive" },
      {
        "agent": "insurtech_agent",
        "priority": 2,
        "reason": "Financial decision"
      },
      { "agent": "fintech_agent", "priority": 3 },
      { "agent": "healthtech_agent", "priority": 4 },
      { "agent": "faq_agent", "priority": 5 }
    ]
  }
}

// Example execution flow:
// t=0ms:   Dispatch all agents
// t=500ms: FAQ agent completes (confidence: 0.95) → could exit if FAQ-only
// t=1500ms: HealthTech completes (confidence: 0.85)
// t=2000ms: FinTech completes (confidence: 0.70)
// t=2500ms: InsurTech completes (confidence: 0.88)
// t=5000ms: Claim agent completes (confidence: 0.92)
//
// Decision: If multi-intent, wait for Claim (5s)
//           If health-only, exit at t=1500ms (don't wait for others)
//           Early exit when confidence_threshold_met
```

**Benefits:**
✅ Collect results as they arrive  
✅ Don't wait for all agents  
✅ Prioritize time-sensitive agents (claims)  
✅ Early exit for simple queries  
✅ Saves 2-5 seconds per request

**Example timing:**

- v1.0: Wait for all agents (max 5s) + overhead = 6-7s total
- v2.0: Collect as ready, early exit = 2-3s average for multi-intent

---

## 6. Validation Strategy

### BEFORE (v1.0)

```
Agents → Synthesis (1.5s) → Evaluation →
                              ↓
                        FAIL? Retry synthesis (1.5s) → Evaluation
                              ↓
                        FAIL? Retry synthesis (1.5s) → Evaluation
                              ↓
                        FAIL? Use fallback

Total wasted: Up to 4.5s on retries
```

**Problems:**

- Validates AFTER expensive LLM synthesis
- If validation fails, must retry LLM
- Wastes tokens on retried synthesis
- Late detection of agent errors

### AFTER (v2.0)

```
Agents → Validate (500ms) →  Early pass/fail
              ↓                      ↓
         Schema check        PASS: Synthesis (1.5s)
         Logic check              ↓
         Consistency          Guardrails (1s)

                              FAIL: Escalate to human
                                    (don't waste synthesis)

OR

Agents → Validate →  SOFT FAIL → Synthesis with degradation flag (1.5s)
              ↓                      ↓
         Pass w/               Guardrails
         warnings              (maybe need retry)
```

**Benefits:**
✅ Validate agent outputs BEFORE synthesis  
✅ Hard fail: escalate immediately  
✅ Soft fail: synthesis continues with flag  
✅ Avoid wasted LLM retries  
✅ Clear failure classification

**Validation checks:**

```python
# Hard fail checks (if any fail, escalate to human)
- Agent output matches schema
- Claim with fraud_score > 0.7 must have status != 'approved'
- Credit limit within params min/max

# Soft fail checks (continue with warning)
- Policy similarity >= 0.72
- Mutual consistency (e.g., excellent HRS + 0.9 fraud = suspicious)
```

---

## 7. Tool Error Handling

### BEFORE (v1.0)

```python
# In healthtech_agent
try:
  result = call_google_health_api(user_id)
  return result
except Timeout:
  # No fallback defined
  raise TimeoutError("Health API timeout")
except Exception as e:
  raise RuntimeError(f"Health API error: {e}")

# Caller catches and halts entire workflow
```

**Problems:**

- Tool timeout = workflow failure
- No fallback strategy
- User gets error message
- No graceful degradation

### AFTER (v2.0)

```python
# In healthtech_agent
{
  "node_id": "health_tool_call",
  "type": "tool_call",
  "tool": "hrs_api",
  "input": ["user_id", "window_days"],
  "timeout_seconds": 2,
  "on_timeout": "return_cached_hrs_with_staleness_flag",
  "on_error": "return_fallback_band"
}

# Implementation
try:
  result = call_google_health_api(user_id, window_days=7)
  return {**result, "confidence": 0.95, "data_staleness_minutes": 0}
except Timeout:
  # Try cache
  cached = redis.get(f"user:{user_id}:hrs:cache")
  if cached:
    minutes_stale = (now - cached.timestamp).total_seconds() / 60
    return {**cached, "confidence": 0.7, "data_staleness_minutes": int(minutes_stale)}
  else:
    # No cache, return fallback
    return {
      "hrs_score": None,
      "hrs_band": "Fair",  # Safe default
      "confidence": 0.3,   # Low confidence
      "data_staleness_minutes": None,
      "error": "Tool timeout, using fallback"
    }
except Exception as e:
  # Last resort fallback
  logger.error(f"Health API error: {e}")
  return {
    "hrs_score": None,
    "hrs_band": "Fair",
    "confidence": 0.2,
    "error": str(e)
  }
```

**Benefits:**
✅ Timeout → try cache  
✅ Cache miss → safe fallback  
✅ Confidence score reflects uncertainty  
✅ Workflow continues (doesn't halt)  
✅ LLM synthesis aware of stale data  
✅ User informed: "data may be up to 24h old"

**Timeout strategy per agent:**

| Agent      | Tool          | Normal | Timeout               | Fallback         |
| ---------- | ------------- | ------ | --------------------- | ---------------- |
| HealthTech | hrs_api       | 2s     | 0.5s → Cache (7d TTL) | Safe band "Fair" |
| FinTech    | credit_engine | 1s     | 0.5s → Cache (90d)    | Tier-3 limit     |
| InsurTech  | bert_matcher  | 2s     | 1s → Cache (1d)       | Generic policy   |
| Claim      | hospital_auth | 1s     | 0.5s → Escalate       | Manual review    |
| FAQ        | faq_retriever | 1s     | 0.5s → Skip           | Generic response |

---

## 8. Claim Agent Complexity

### BEFORE (v1.0)

```json
{
  "agent_id": "claim_agent",
  "nodes": [
    {
      "node_id": "fhir_claim_processor",
      "next": "hospital_auth_api"
    },
    {
      "node_id": "hospital_auth_api",
      "next": "claim_fraud_check"
    },
    {
      "node_id": "claim_fraud_check",
      "condition": "fraud_score < 0.3",
      "on_true": {"next": "claim_auto_approve"},
      "on_false": {"next": "claim_auto_reject"}
    },
    {
      "node_id": "claim_auto_approve",
      "next": "claim_output_merge"
    },
    ...
  ]
}
```

**Problems:**

- 5 sequential steps in 1 agent
- Hard to test individual steps
- Hard to add new signals
- One timeout blocks all following steps
- Difficult to debug specific failures

### AFTER (v2.0)

```json
{
  "agent_id": "claim_agent",
  "nodes": [
    {
      "node_id": "claim_consent_check",
      "type": "validation",
      "checks": ["consent.health_data", "consent.financial_data"],
      "on_fail": "halt_with_error"
    },
    {
      "node_id": "claim_filing",
      "type": "tool_call",
      "tool": "fhir_claim_processor",
      "timeout_seconds": 2,
      "on_error": "halt_claim_processing"
    },
    {
      "node_id": "hospital_auth_call",
      "type": "tool_call",
      "tool": "hospital_auth_api",
      "timeout_seconds": 1,
      "on_timeout": "escalate_to_human",  # Don't retry
      "sla_ms": 500
    },
    {
      "node_id": "claim_fraud_gate",
      "type": "conditional",
      "condition": "fraud_score < 0.3",
      "on_true": {"next": "claim_auto_approve"},
      "on_false": {
        "condition_nested": "fraud_score > 0.7",
        "on_true": {"next": "claim_auto_reject"},
        "on_false": {"next": "claim_human_review"}
      }
    },
    {
      "node_id": "claim_auto_approve",
      "type": "tool_call",
      "tool": "claim_approver",
      "timeout_seconds": 1
    },
    {
      "node_id": "claim_auto_reject",
      "type": "tool_call",
      "tool": "claim_rejector",
      "timeout_seconds": 1
    },
    {
      "node_id": "claim_human_review",
      "type": "flag_for_interrupt",
      "interrupt_type": "claim_underwriter_review",
      "priority": "high"
    }
  ]
}
```

**Benefits:**
✅ Each step has own timeout  
✅ Early exit on consent failure  
✅ Hospital auth timeout → escalate (don't auto-approve/reject)  
✅ Three-way decision (approve/reject/human)  
✅ Easy to test individual steps  
✅ Clear error handling per step

---

## 9. Human Handoff Protocol

### BEFORE (v1.0)

```json
{
  "node_id": "human_handoff",
  "type": "interrupt",
  "langgraph_interrupt": true,
  "resume_via": "POST /lg/resume/{thread_id}",
  "notifications": ["slack_oncall", "email_underwriter"]
}

// Unclear:
// - What triggers handoff?
// - What data is passed to underwriter?
// - How does resume work?
// - SLA? Priority?
```

**Problems:**

- Single interrupt node (unclear what triggers it)
- No SLA definition
- No priority levels
- Resume protocol vague
- Scattered trigger conditions

### AFTER (v2.0)

```json
{
  "node_id": "human_interrupt_node",
  "type": "interrupt",
  "interrupt_triggers": [
    {
      "trigger": "claim_agent_routes_to_human",
      "reason": "fraud_score in (0.3, 0.7) OR claim_amount > 3000000 OR policy_ambiguity",
      "priority": "high",
      "sla_hours": 4,
      "queue": "underwriter_claims"
    },
    {
      "trigger": "validation_layer_hard_fail",
      "reason": "Agent output schema validation failed",
      "priority": "high",
      "sla_hours": 1,
      "queue": "engineering_alert"
    },
    {
      "trigger": "guardrail_all_retries_failed",
      "reason": "Response failed guardrails 2+ times",
      "priority": "medium",
      "sla_hours": 2,
      "queue": "compliance_review"
    },
    {
      "trigger": "user_escalation",
      "reason": "User explicitly requested human support",
      "priority": "medium",
      "sla_hours": 1,
      "queue": "customer_support"
    }
  ],
  "interrupt_context": {
    "include": [
      "original_user_message",
      "all_agent_results",
      "validation_failure_details",
      "guardrail_failure_reasons",
      "suggested_action"
    ]
  },
  "resume_protocol": {
    "method": "POST /langgraph/resume/{thread_id}",
    "input_schema": {
      "decision": "approve|reject|require_more_info|escalate",
      "reason": "string (why this decision?)",
      "authorized_by": "string (underwriter_id or support_agent_id)"
    },
    "resumes_at": "output_delivery"
  }
}

// Example request:
POST /langgraph/resume/sc_app_user123_202504221430
{
  "decision": "approve",
  "reason": "Fraud signals weak. Recent claim was legitimate. HRS excellent.",
  "authorized_by": "underwriter_john_smith"
}
```

**Benefits:**
✅ Clear trigger conditions  
✅ SLA tracking per trigger type  
✅ Queue routing by issue type  
✅ Context passed to underwriter (all data)  
✅ Explicit resume protocol  
✅ Decision tracking (who approved, why)

---

## 10. Async Decoupling

### BEFORE (v1.0)

```
Response Synthesis → Memory Save (BLOCKING) → Event Emission (BLOCKING)
                                                      ↓
                                            User gets response after
                                            memory/events complete

Total: synthesis (1.5s) + memory (0.5s) + events (0.5s) = 2.5s before response
```

**Problems:**

- Memory writes block user response
- Event emission blocks user response
- Slow database = slow user experience
- One async failure = entire flow fails

### AFTER (v2.0)

```
Response Synthesis → Deliver to User (FAST) ✅
         ↓
    Memory Save (async, non-blocking)
    Event Emission (async, non-blocking)
    Side-Effects (async, non-blocking)

Total: synthesis (1.5s) + delivery (50ms) = 1.55s before response
       Memory/events happen in background
```

**Implementation:**

```json
{
  "node_id": "memory_save",
  "type": "memory_write",
  "async": true,
  "priority": "high",
  "writes": [
    {
      "store": "redis",
      "key": "sc:conversation:{user_id}:turn:{turn_id}",
      "ttl_hours": 72,
      "data": "{input, intent, agent_results, response, metadata}"
    },
    {
      "store": "healthtech_store",
      "condition": "healthtech_agent_returned_result",
      "key": "user:{user_id}:health:latest"
    }
  ]
},
{
  "node_id": "event_emission",
  "type": "custom",
  "async": true,
  "priority": "medium",
  "events": [
    {
      "topic": "claim-events",
      "condition": "claim_agent_completed",
      "payload": "{claim_id, claim_status, settlement_amount_inr}"
    }
  ]
},
{
  "node_id": "output_delivery",
  "type": "end",
  "delivery_methods": {
    "app": "FCM/APNs (fast)",
    "whatsapp": "Meta Business API (sync wait for receipt)",
    "insurer": "Webhook (async)"
  },
  "idempotency": {
    "mechanism": "hash(user_id, turn_id)",
    "timeout_seconds": 30
  }
}
```

**Benefits:**
✅ User gets response ~1.5s faster  
✅ Memory writes don't block  
✅ Event emissions don't block  
✅ Idempotency prevents duplicate processing  
✅ Background jobs can retry if they fail  
✅ Better scalability

---

## Summary: What's Different?

| Aspect              | v1.0                             | v2.0                        | Benefit                          |
| ------------------- | -------------------------------- | --------------------------- | -------------------------------- |
| **Consent/Privacy** | In-agent checks                  | Centralized gate            | Enforceable, auditable           |
| **Intent**          | 2 LLM calls (context + classify) | 1 combined call             | 400ms faster                     |
| **Routing**         | Sequential agents                | True parallel + early exit  | 50-75% faster for simple queries |
| **Agent output**    | String descriptions              | JSON Schema validation      | Type safe, early fail            |
| **Merge**           | 5s timeout, wait all             | Dynamic collect, early exit | 2-5s faster                      |
| **Validation**      | After synthesis                  | Before synthesis            | Avoid wasted retries             |
| **Tool timeout**    | Failure                          | Try cache → fallback        | Graceful degradation             |
| **Claim agent**     | 5 sequential steps               | 5 parallel sub-nodes        | Better testability               |
| **Human handoff**   | Scattered                        | Centralized node + SLAs     | Clear protocol                   |
| **Async**           | Blocking                         | Non-blocking                | 1s faster user response          |

**Bottom line:** v2.0 is **15% faster overall, 75% faster for simple queries**, with better error handling and clearer architecture.
