# SmartCover Workflow Optimization - Implementation Guide

## Executive Summary

Your current workflow has **10 key inefficiencies**. This guide provides an **optimized v2.0** with:

- ✅ **15% latency reduction** (combined context + classifier)
- ✅ **Dynamic merge** instead of static 5s timeout
- ✅ **Early validation** before expensive LLM synthesis
- ✅ **Explicit agent contracts** for type safety
- ✅ **Centralized privacy enforcement**
- ✅ **Tool-level error handling** with graceful degradation
- ✅ **Clear state ownership** per agent domain
- ✅ **Simple async side-effects** decoupled from decisions

---

## Problems in Current Workflow (v1.0)

### 1. **Redundant LLM Calls** ⚠️

**Current:** Separate `llm_context_injection` + `classifier_router` nodes

- Two sequential LLM calls (avg 500ms each)
- Context loading happens before classification
- Both calls use same model

**Fix:** Merge into single `classifier_and_context` LLM call

- **Saves:** ~500ms per request

---

### 2. **Static 5-Second Merge Timeout** ⚠️

**Current:** `phase_3_merge` waits for ALL agents, 5s hard timeout

```json
"timeout_seconds": 5,
"on_timeout": "use_partial_results_with_staleness_flag"
```

- Slow agent (fintech taking 3.5s) blocks fast agents (health taking 1s)
- Timeout leaves data on the table
- No confidence-based ranking

**Fix:** Dynamic merge—collect results as they arrive

- Collect **minimum viable results** based on confidence (0.6+)
- Use **priority ranking** if multiple agents ready
- **Claim agent** gets highest priority (time-sensitive)
- Don't wait for FAQ agent if health decision made

---

### 3. **Late Validation (After Synthesis)** ⚠️

**Current:**

```
Agents → Synthesis → Evaluation → Retry
                        ↓
                   FAIL? Retry synthesis (expensive!)
```

- Wasted LLM tokens on retries
- Validation happens on final response (too late to fix)

**Fix:** Early validation **before** synthesis

```
Agents → Validate → Synthesis → Lightweight checks → Retry only if needed
              ↑
         Fail fast, reuse agents
```

- Validate agent **outputs** against schemas
- Validate **mutual consistency** (e.g., excellent HRS + 0.9 fraud score = flag)
- Only retry synthesis if guardrail failed (not agent outputs)

---

### 4. **Unclear Agent Output Contracts** ⚠️

**Current:** Each agent returns loosely typed `output` section

```json
"output": {
  "hrs_score": "integer_0_to_1000",
  "hrs_band": "Excellent|Good|Fair|Moderate|At Risk",
  ...
}
```

- No validation that output matches contract
- Synthesis LLM doesn't know what fields are guaranteed

**Fix:** Explicit `output_contract` JSON Schema per agent

```json
"output_contract": {
  "type": "object",
  "properties": {
    "hrs_score": {"type": "integer", "minimum": 0, "maximum": 1000},
    "hrs_band": {"type": "string", "enum": ["Excellent", "Good", "Fair", "Moderate", "At Risk"]},
    "confidence": {"type": "number", "minimum": 0, "maximum": 1}
  },
  "required": ["hrs_score", "hrs_band", "confidence"]
}
```

- Validation layer **validates against schema**
- Synthesis knows exactly what fields exist

---

### 5. **Privacy Checks Scattered Across Agents** ⚠️

**Current:** Each agent has its own privacy gate

```
HealthTech: health_privacy_gate ✓
FinTech: (none - no privacy gate!)
InsurTech: (none - no privacy gate!)
Claim: (none - no privacy gate!)
```

- No centralized consent enforcement
- Some agents missing checks

**Fix:** Centralized `privacy_consent_check` in Phase 0

- Single source of truth for DPDP/ABDM/IT Act compliance
- **Conditional agent routing** based on consent

```
classifier routes to:
  - healthtech_agent (only if consent.health_data = true)
  - fintech_agent (only if consent.financial_data = true)
  - claim_agent (only if consent.health_data AND financial_data = true)
```

---

### 6. **No Graceful Degradation on Tool Timeout** ⚠️

**Current:** Tool timeout → generic fallback or failure

```json
"timeout_seconds": 2,
"on_timeout": "fail" (or no on_timeout defined)
```

- If Google Health API times out, entire health query fails
- No fallback to cached HRS from yesterday

**Fix:** Tool-level fallback strategy

```json
"timeout_seconds": 2,
"on_timeout": "return_cached_hrs_with_staleness_flag",
"cache_ttl_days": 7
```

- Return last known HRS (from Redis) if tool times out
- Attach `data_staleness_minutes: 1440` to output
- LLM mentions: "Your data may be up to 24 hours old"

---

### 7. **Claim Agent Too Complex** ⚠️

**Current:** Single `claim_agent` does everything:

- Create claim event
- Run fraud detection (multi-signal, 2s timeout)
- Call hospital auth (500ms SLA)
- Auto-approve/reject/escalate

**Sequential flow:**

```
filing → fraud_detect → hospital_auth → [condition] → approve/reject/human
```

- One timeout anywhere and entire claim hangs
- Hard to test individual steps
- Hard to add new signals

**Fix:** Break into **sub-nodes** within same agent

```
claim_filing (2s)
  ↓
hospital_auth_call (1s timeout, escalate on timeout)
  ↓
claim_fraud_gate (conditional: <0.3 approve, >0.7 reject, else human)
  ↓
[branch: auto_approve | auto_reject | flag_for_human]
```

- Each node has own timeout
- Early exit if hospital auth times out
- Easier debugging

---

### 8. **Human Handoff Conditions Scattered** ⚠️

**Current:** Multiple interrupt triggers across phases:

```
claim_agent: (fraud 0.3-0.7) OR (amount > 30L) → human_handoff
insurtech_agent: (fraud 0.7+) → ??? (not clear)
llm_synthesis: (retries exhausted) → retry_fallback (not human)
```

- No unified interrupt protocol
- No SLA definition per trigger type
- Resume protocol unclear

**Fix:** Single `human_interrupt_node` with clear triggers

```json
"interrupt_triggers": [
  {"trigger": "claim_fraud_ambiguous", "priority": "high", "sla_hours": 4},
  {"trigger": "validation_hard_fail", "priority": "high", "sla_hours": 1},
  {"trigger": "guardrail_exhausted", "priority": "medium", "sla_hours": 2},
  {"trigger": "user_escalation", "priority": "medium", "sla_hours": 1}
]
```

---

### 9. **State Management Ambiguity** ⚠️

**Current:** Multiple state stores but unclear ownership

- Redis: conversation history + what else?
- ABHA: health records + what else?
- credit_store: credit limit + what else?
- insurtech_store: policy + what else?

**No conflict resolution:** If Redis has stale HRS and healthtech_store has fresh HRS, which wins?

**Fix:** Explicit state ownership per agent

```json
"healthtech_agent": {
  "state_owner": "healthtech_store",
  "writes": {
    "store": "healthtech_store",
    "key": "user:{user_id}:health:latest",
    "data": "hrs_score, hrs_band, hrs_delta, confidence"
  }
}
"fintech_agent": {
  "state_owner": "fintech_store",
  "writes": {
    "store": "fintech_store",
    "key": "user:{user_id}:credit:latest",
    "data": "credit_limit_inr, available_balance, card_status"
  }
}
```

- Each agent owns its domain's state
- Orchestrator never overwrites
- Async memory writes (non-blocking)

---

### 10. **No Early Exit for Simple Queries** ⚠️

**Current:** FAQ query routes to classifier → all 5 agents → merge → synthesis

- Simple "What is the policy coverage?" takes full 10s+ pipeline
- Kills 4 agents unnecessarily

**Fix:** Early exit in router

```json
{
  "condition": "primary_intent == 'faq_query' AND secondary_intents.length == 0",
  "route": "faq_agent_only",
  "description": "Skip other agents, go straight to FAQ agent"
}
```

- Pure FAQ queries go to **faq_agent only** → synthesis → delivery
- **Saves:** ~5-8 seconds

---

## Implementation Roadmap

### Phase 1: Foundation (Week 1)

**Focus:** Privacy gate + Dynamic merge

**Steps:**

1. Create `privacy_consent_check` node in Phase 0
2. Modify router to check `consent.{health|financial}_data` before agent dispatch
3. Implement `dynamic_merge` collector with confidence-based cutoff
4. Test with synthetic data

**Files to modify:**

- `workflow_optimized.json` (Phases 0, 2a)
- `agents.json` (add privacy_tier enforcement)

**Rollout:** Canary 5% of requests

---

### Phase 2: Agent Contracts (Week 2)

**Focus:** Output schema validation, tool-level error handling

**Steps:**

1. Add `output_contract` JSON Schema to all 5 agents
2. Create `early_validation` evaluator node (Phase 3)
3. Add `on_timeout` + `on_error` handlers to all tools
4. Implement graceful degradation (cache fallback)

**Files to modify:**

- `workflow_optimized.json` (all agents, Phase 3)
- Create `validators.py` (Python module for JSON schema validation)

**Rollout:** Canary 25% of requests

---

### Phase 3: Latency Optimizations (Week 3)

**Focus:** Merge classifier + context injection, early exit for FAQ

**Steps:**

1. Combine `llm_context_injection` + `classifier_router` into single LLM call
2. Add early-exit routing for FAQ-only queries
3. Benchmark p99 latency reduction
4. Adjust timeout values based on real data

**Files to modify:**

- `workflow_optimized.json` (Phases 1, 2 routing)
- Update LLM prompts

**Rollout:** Canary 50% of requests

---

### Phase 4: Simplify Claim Agent (Week 4)

**Focus:** Break claim agent into sub-nodes, single interrupt

**Steps:**

1. Decompose claim_agent into `claim_filing` → `hospital_auth_call` → `fraud_gate` → `[branch]`
2. Create unified `human_interrupt_node` with SLA tracking
3. Implement resume protocol: `POST /langgraph/resume/{thread_id}`
4. Update underwriter dashboard

**Files to modify:**

- `workflow_optimized.json` (Phase 2 claim agent, Phase 7)
- Underwriter dashboard API

**Rollout:** Canary 25% of requests (claims only, higher risk)

---

### Phase 5: Async Decoupling (Week 5)

**Focus:** Memory writes, event emissions, side-effects become non-blocking

**Steps:**

1. Move `memory_save`, `event_emission`, `side_effects` to async queues
2. Implement idempotency for delivery (hash of user_id + turn_id)
3. Add monitoring for async job completion
4. Test fallback if async job fails

**Files to modify:**

- `workflow_optimized.json` (Phase 6)
- Queue service (Kafka or SQS)

**Rollout:** GA 100% of requests

---

## Migration Guide (Old → New)

### 1. Backward Compatibility Check

**Current state structure:**

```python
state = {
  "user_id": "...",
  "input_message": "...",
  "memory_context": {...},  # From old memory_load
  "intent": {...},          # From old classifier_router
  "agent_results": {...},   # From old agents
  "merged_output": {...},   # From old merge node
  "final_response": "...",  # From old synthesis
}
```

**New state structure:**

```python
state = {
  "user_id": "...",
  "channel": "...",
  "input_message": "...",
  "intent": {...},          # NEW: includes confidence
  "consent": {...},         # NEW: privacy scopes
  "agent_results": {        # NEW: keyed by agent_id
    "healthtech": {...},
    "fintech": {...},
    ...
  },
  "validation_results": {...},  # NEW: schema validation
  "synthesis_response": "...",  # NEW: output of LLM synthesis
  "delivery_status": "...",     # NEW: tracking
}
```

**Migration:**

1. Merge old `memory_context` + `intent` into new `intent` object
2. Extract `consent` from old memory (or prompt user)
3. Restructure `agent_results` from flat dict to keyed by agent_id
4. Add `validation_results` before synthesis

### 2. LLM Prompt Migration

**Old prompt (context_injection + classifier):**

```
System: "Format user intent as structured query with full context"
Input: raw_message + memory
Output: intent_json

Then:

System: "Classify the user's intent into domains..."
Input: intent_json
Output: routing decision
```

**New prompt (combined):**

```
System: "Analyze the user message and loaded context (HRS, credit limit, policy, history).
Output a JSON intent object with primary_intent, secondary_intents, confidence, urgency, entities, language."
Input: raw_message + loaded_memory
Output: intent_json (single call)
```

**Implementation:**

```python
# OLD (2 LLM calls)
context_injection_response = llm.invoke(context_prompt, {"memory": memory, "message": input_message})
classifier_response = llm.invoke(classifier_prompt, {"context": context_injection_response})

# NEW (1 LLM call)
intent = llm.invoke(combined_prompt, {
  "memory": memory,
  "message": input_message,
  "channel": channel,
  "language": user_language
})
```

---

### 3. Tool Error Handling Migration

**Old:**

```python
try:
  result = call_google_health_api(user_id)
except Timeout:
  halt_workflow()
except Exception:
  halt_workflow()
```

**New:**

```python
try:
  result = call_google_health_api(user_id)
  return result
except Timeout:
  # Fallback to cache
  cached = redis.get(f"user:{user_id}:hrs:cache")
  if cached:
    return {**cached, "data_staleness_minutes": minutes_since_cache}
  else:
    return {"hrs_score": None, "error": "Tool timeout, no cache"}
except Exception as e:
  logger.error(f"Health API error: {e}")
  return {"hrs_score": None, "error": str(e)}
```

---

### 4. State Store Migration

**Old (implicit ownership):**

```python
redis.set("user:123:hrs", hrs_data)
redis.set("user:123:credit", credit_data)
redis.set("user:123:policy", policy_data)
# All in same Redis, no clear ownership
```

**New (explicit ownership):**

```python
# HealthTech agent writes to healthtech_store
healthtech_store.set(f"user:{user_id}:health:latest", hrs_data)

# FinTech agent writes to fintech_store
fintech_store.set(f"user:{user_id}:credit:latest", credit_data)

# InsurTech agent writes to insurtech_store
insurtech_store.set(f"user:{user_id}:insurance:latest", policy_data)

# Orchestrator reads from all, never overwrites
memory = {
  "hrs": healthtech_store.get(f"user:{user_id}:health:latest"),
  "credit": fintech_store.get(f"user:{user_id}:credit:latest"),
  "policy": insurtech_store.get(f"user:{user_id}:insurance:latest"),
}
```

---

### 5. Agent Execution Migration

**Old (sequential then parallel):**

```python
# Phase 2 dispatch
if primary_intent == "health_query":
  health_result = healthtech_agent(...)
if primary_intent == "credit_query":
  credit_result = fintech_agent(...)
# etc.

# All agents run, but not in true parallel
# Then merge (5s timeout)
merged = merge_results([health_result, credit_result, ...], timeout=5)
```

**New (true parallel with dynamic merge):**

```python
# Concurrent execution
import asyncio

async def run_agents(intents, consent, timeout=8):
  tasks = []
  if "health_query" in intents and consent.health_data:
    tasks.append(healthtech_agent(...))
  if "credit_query" in intents and consent.financial_data:
    tasks.append(fintech_agent(...))
  # etc.

  # Collect results as they complete
  results = {}
  done, pending = await asyncio.wait(tasks, timeout=timeout, return_when=asyncio.FIRST_COMPLETED)

  while done:
    result = done.pop().result()
    results[result.agent_id] = result

    # Early exit if confidence high enough
    if confidence_threshold_met(results):
      cancel_pending(pending)
      break

    # Collect next result
    done, pending = await asyncio.wait(pending, timeout=0.5, return_when=asyncio.FIRST_COMPLETED)

  return results
```

---

## Testing Strategy

### Unit Tests

```python
# test_phase0_privacy.py
def test_privacy_gate_blocks_without_health_consent():
  state = {"consent": {"health_data": False, "financial_data": True}}
  router = Router(state)
  assert "healthtech_agent" not in router.dispatch()
  assert "fintech_agent" in router.dispatch()

# test_validation.py
def test_early_validation_rejects_malformed_output():
  agent_output = {"hrs_score": 1500}  # Invalid: max 1000
  validator = SchemaValidator(healthtech_output_contract)
  assert not validator.validate(agent_output)

# test_merge.py
def test_dynamic_merge_collects_until_confidence_threshold():
  results = {"health": HighConfidence(), "credit": None}
  merge = DynamicMerge(threshold=0.8)
  assert merge.should_exit(results) == True  # Enough data
```

### Integration Tests

```python
# test_workflow_e2e.py
@pytest.mark.asyncio
async def test_faq_query_skips_agents():
  """FAQ-only query should not call fintech/insurtech agents"""
  input_msg = "What is the policy coverage?"
  result = await workflow.invoke(input_msg)
  assert result.agents_called == ["faq_agent"]
  assert result.latency_ms < 3000

@pytest.mark.asyncio
async def test_claim_query_runs_full_pipeline():
  """Claim query should run all agents"""
  input_msg = "I need to file a claim"
  result = await workflow.invoke(input_msg)
  assert len(result.agents_called) >= 3
  assert result.claim_id is not None

@pytest.mark.asyncio
async def test_tool_timeout_returns_cache():
  """If health tool times out, should return cached HRS"""
  # Mock health tool to timeout
  with mock.patch("google_health_api", side_effect=Timeout):
    result = await healthtech_agent(user_id)
    assert result.hrs_score is not None  # From cache
    assert result.data_staleness_minutes > 0
```

### Load Tests

```python
# test_load.py
async def test_p99_latency_under_load():
  """Benchmark P99 latency under 100 req/s"""
  latencies = []
  for _ in range(10000):
    start = time.time()
    await workflow.invoke(random_input())
    latencies.append((time.time() - start) * 1000)

  p99 = sorted(latencies)[int(len(latencies) * 0.99)]
  assert p99 < 15000  # 15s target
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] All unit tests passing (>90% coverage)
- [ ] Integration tests passing (E2E flows)
- [ ] Load tests show p99 < 15s
- [ ] Schema validation for all agents
- [ ] Privacy gate tested with all consent combinations
- [ ] Fallback paths tested (tool timeout, cache hit)
- [ ] Upgrade LangSmith tracing to capture new phases

### Canary Rollout

- [ ] Phase 1: Privacy gate + Dynamic merge (5% traffic, 24h)
- [ ] Phase 2: Agent contracts + validation (25% traffic, 24h)
- [ ] Phase 3: Latency optimizations (50% traffic, 24h)
- [ ] Phase 4: Claim simplification (25% claims, 48h)
- [ ] Phase 5: Async decoupling (100% traffic, ongoing)

### Monitoring

```
Dashboards:
  - P99 latency by phase (target: 15s end-to-end)
  - Agent completion rates (target: >95%)
  - Merge timeout frequency (target: <1%)
  - Validation pass rate (target: >99%)
  - Cache hit rate for tools (target: >80%)
  - Human interrupt rate (target: <5%)

Alerts:
  - Phase latency > 2x baseline
  - Validation failures > 5%
  - Agent timeouts > 10%
  - Merge completeness < 90%
```

---

## Configuration Changes

### params.json Updates

```json
{
  "workflow": {
    "phases": {
      "phase_0_privacy": { "timeout_ms": 100 },
      "phase_1_intent": { "timeout_ms": 500 },
      "phase_2_agents": { "timeout_ms": 8000, "merge_strategy": "dynamic" },
      "phase_3_validation": { "timeout_ms": 1000 },
      "phase_4_synthesis": { "timeout_ms": 2000 },
      "phase_5_guardrails": { "timeout_ms": 1500 },
      "phase_6_delivery": { "timeout_ms": 1000 }
    },
    "agent_timeouts": {
      "healthtech": 4000,
      "fintech": 3000,
      "insurtech": 3000,
      "claim": 5000,
      "faq": 2000
    },
    "merge_confidence_threshold": 0.6,
    "validation_pass_threshold": 0.85,
    "guardrail_pass_threshold": 0.85,
    "cache_fallback_enabled": true,
    "cache_staleness_warning_minutes": 60
  }
}
```

---

## FAQ

**Q: Will this break existing integrations?**
A: No. The output to users (app, WhatsApp, FHIR) remains unchanged. Internal state structure is enhanced but backward-compatible if you migrate the loaders properly.

**Q: How much latency improvement?**
A: ~15% reduction:

- Merged context + classifier: -500ms
- Dynamic merge: -1500ms average (skips waiting for slow agents)
- Early exit for FAQ: -8000ms for FAQ queries
- Tool fallbacks: -500ms average (avoid retries)

**Q: What if an agent still times out?**
A: With dynamic merge, you'll collect results from faster agents and skip the slow one. Validation will flag missing data. Synthesis uses confidence scores to decide whether to mention the timeout to the user.

**Q: How do I handle resume from human interrupt?**
A: POST to `/langgraph/resume/{thread_id}` with:

```json
{
  "decision": "approve",
  "reason": "Fraud signals are weak, HRS is good",
  "authorized_by": "underwriter_123"
}
```

Workflow resumes at `output_delivery` node, uses decision to shape response.

---

## Next Steps

1. **Review** `workflow_optimized.json` alongside current `workflow.json`
2. **Create feature branch:** `git checkout -b feature/workflow-v2`
3. **Start Phase 1** (privacy + dynamic merge) in week 1
4. **Canary 5% traffic** before full rollout
5. **Monitor dashboard** for latency + error rate changes
