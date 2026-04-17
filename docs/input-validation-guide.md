# Input Validation Guide for CostGuard

**Document Date:** 2026-04-17  
**Root Cause:** Weak boundary validation on upstream Hydra Pulse cost tracking JSON

## Problem Statement

Fix-commit churn analysis revealed 5 fixes (38.5%) all traced to **missing input validation at system boundaries**:

1. `.usage` vs `.message.usage` JSON structure inconsistency
2. Subagent transcript parsing format drift
3. Pricing fuzzy matching failures
4. Model/offset calibration stale state
5. Statusline token count inaccuracy

**Root Cause:** CostGuard trusts upstream outputs without defensive parsing. When Hydra Pulse changes shape, multiple cascading fixes are required instead of one hardened boundary.

## Upstream Contracts to Validate

### 1. Cost Tracking JSON (from Hydra Pulse)

**Expected Structure:**
```json
{
  "message": {
    "usage": {
      "input_tokens": 1000,
      "output_tokens": 500,
      "cache_creation_input_tokens": 0,
      "cache_read_input_tokens": 0
    }
  },
  "model": "claude-opus-4-6",
  "timestamp": "2026-04-17T12:34:56Z"
}
```

**Validation Rules:**
- Must have `message.usage` (not `usage` at root)
- Must have `input_tokens` and `output_tokens` fields
- Must have `model` field (non-empty string)
- Must have `timestamp` in ISO-8601 format

**What to Do If Invalid:**
1. Log with endpoint URL and raw JSON (20 chars snippet)
2. Fall back to zeros: `{"input_tokens": 0, "output_tokens": 0}`
3. Alert monitoring: invalid cost tracking structure

### 2. Subagent Transcript Format

**Expected Fields:**
- `session_id` (UUID string)
- `agent_name` (string)
- `model_used` (string: `haiku`, `sonnet`, `opus`)
- `tokens_used` (dict with `input`, `output`, `cached_read`)
- `cost_usd` (float)
- `timestamp` (ISO-8601)

**Validation Rules:**
- Reject transcripts missing `session_id` or `agent_name`
- Reject if `tokens_used` is not a dict
- Reject if `cost_usd` is negative
- Reject if `timestamp` is unparseable

**What to Do If Invalid:**
1. Log transcript ID + error reason
2. Skip this transcript in aggregation
3. Notify Hydra Pulse of malformed data

### 3. Model Parameter Snapshots

**Validation Rules:**
- All model offsets must be captured at initialization
- Offsets cannot change after snapshot
- If offset access before snapshot complete, raise error
- Log parameter values for debugging

**Implementation:**
```python
class ModelSnapshot:
    def __init__(self, model_name: str):
        if not model_name:
            raise ValueError("model_name required")
        self.model_name = model_name
        self.snapshot_time = time.time()
        self.offset = self._load_offset()
        self._sealed = True

    def get_offset(self) -> float:
        if not self._sealed:
            raise RuntimeError("Snapshot not sealed")
        return self.offset
```

### 4. Session Token Counts

**Validation Rules:**
- Cache tokens must not be negative
- Total tokens = input + output + cache_read
- Timestamp must be recent (< 60s old)
- If timestamp > 60s old, fetch fresh

**Implementation:**
```python
def get_token_count(session_id: str, max_age_seconds: int = 60) -> int:
    cached = _token_cache.get(session_id)
    if cached and (time.time() - cached["timestamp"]) < max_age_seconds:
        return cached["total"]
    # Fetch fresh
    live = _fetch_live_transcript(session_id)
    return live["total_tokens"]
```

## Pre-Commit Validation Checklist

- [ ] All JSON parsing wrapped in try/except with logging
- [ ] Pydantic models defined for all upstream inputs
- [ ] Type hints on all parsing functions
- [ ] Unit tests for malformed/missing fields
- [ ] CI linter checks for hardcoded field access (use constants)

## Testing Strategy

### Unit Tests
```python
def test_cost_tracking_missing_usage_field():
    """Should handle missing .message.usage gracefully."""
    malformed = {"model": "claude-opus-4-6"}
    result = parse_cost_tracking(malformed)
    assert result["input_tokens"] == 0

def test_transcript_negative_cost():
    """Should reject negative cost_usd."""
    invalid = {"session_id": "123", "cost_usd": -1.0}
    with pytest.raises(ValueError):
        parse_transcript(invalid)
```

### Integration Tests
- Simulate Hydra Pulse returning old structure
- Verify CostGuard degrades gracefully
- Verify alerts are triggered for malformed data

## Remediation Timeline

**Immediate (this session):**
1. Add Pydantic models for all upstream inputs
2. Update parsing functions to validate before use
3. Add comprehensive unit tests

**Next Sprint:**
1. Add CI linter rule: block field access without validation
2. Deploy monitoring alert for malformed inputs
3. Document upstream contract with Hydra Pulse team

## References

- **Hydra Pulse Docs:** cost tracking JSON structure
- **Pydantic Guide:** https://docs.pydantic.dev/
- **Related Fixes:** commits 419a003, 0afbbc3, 0b2dc1c, 1de3267, 0700814
