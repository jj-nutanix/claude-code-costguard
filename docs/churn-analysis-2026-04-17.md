# Churn Analysis: claude-code-costguard

**Audit Date:** 2026-04-17  
**Fix Ratio:** 38.5% (5 of 13 commits)  
**Analysis Period:** Last 3 months

## Commit Categorization

| Type | Count | Commits |
|------|-------|---------|
| feat | 1 | e46946e (fleet costs, efficiency scoring) |
| fix | 5 | 419a003, 0afbbc3, 0b2dc1c, 1de3267, 0700814 |
| test | 1 | 61deace (comprehensive test suite) |
| docs | 2 | 1c4c49b, f4288fe |
| ops/ci/chore | 4 | e5f1568, 2372443, 0863de0, d266ecf |

## Top 3 Recurring Fix Themes

1. **Cost Tracking Parse Errors** (3 fixes: 419a003, 0afbbc3, 0b2dc1c)
   - `.usage` vs `.message.usage` JSON structure inconsistency
   - Subagent transcript parsing mismatch with upstream Hydra Pulse
   - Pricing fuzzy matching and spawn error handling

2. **Model/Offset Calibration** (1 fix: 1de3267)
   - Broken offset calibration logic → replaced with snapshot-delta model
   - Root cause: stateful offset tracking was fragile across restarts

3. **Statusline Token Count Accuracy** (1 fix: 0700814)
   - Always uses live transcript instead of cached session state
   - Root cause: stale session/window token counts from non-live sources

## Root Cause Analysis

**Single Root Cause:** Weak upstream contract enforcement with Hydra Pulse

All 5 fixes trace to **missing validation at system boundaries**:
- No input schema validation on cost tracking JSON (upstream can change `.usage` location)
- No versioning/compatibility layer for subagent transcript format
- No snapshot/validation of model parameters at assignment time (offset calibration)
- No cache invalidation strategy for session tokens

**Design Flaw:** The system trusts upstream outputs without defensive parsing. When Hydra Pulse changes shape or internal tracking breaks, CostGuard cascades with multiple small fixes instead of one hardened boundary.

## Concrete Remediation Actions

### 1. Add Input Validation + Schema (Effort: 15min)
**Code Fix:**
- Create `schemas.py` with Pydantic models for all upstream inputs (cost JSON, transcript format, session state)
- Update all parsing functions to validate before use
- Add type hints and validation errors with helpful messages

**Test:**
- Unit tests for malformed/missing fields (e.g., missing `.message.usage`, unexpected nesting)
- Test that schema validation catches all 3 parse error patterns from history

**CI:**
- Pre-commit hook: `pydantic-validate` on schemas.py changes

**Status:** QUICK WIN — implement immediately (affects cost_tracker.py + transcript_parser.py)

### 2. Snapshot Model Parameters on Assignment (Effort: 10min)
**Code Fix:**
- At initialization, snapshot all offset/calibration parameters with commit hash
- Add immutability check: throw error if offset accessed before snapshot completion
- Log parameter values for debugging

**Status:** QUICK WIN — prevents silent calibration drift

### 3. Add Cache Invalidation TTL for Session Tokens (Effort: 10min)
**Code Fix:**
- Add `cache_key = f"session_{session_id}_{timestamp_bucket}"` instead of static keys
- TTL = 60s (prefer live transcript within 1 minute of session start)
- Fall back to cache only if live fetch fails

**Status:** QUICK WIN — prevents stale token count bugs

## Implementation Order (Est. Total: 35min for all 3)

1. Add Pydantic validation schemas (15min)
2. Update parsing functions to use schemas (10min)
3. Add cache invalidation + parameter snapshots (10min)
4. Run test suite + push

---

## Remediation Status

**Action Taken:** Implementing remediation #1 (validation schemas)  
See commits below for execution.
