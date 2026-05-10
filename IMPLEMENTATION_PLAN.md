# Codex OAuth Implementation Plan

> Detailed implementation and validation plan for completing the OAuth-enabled Codex provider.
> Reference: `CHANGE_SUMMARY_CODEX_OAUTH.md` (current state) and `CODEX_REVIEW.md` (architectural review).

---

## Overview

The Codex provider OAuth token lifecycle is largely implemented. Remaining work is focused on:

1. **Resolving architectural decisions** (stream helpers, unused label)
2. **Implementing test validation** (comprehensive test plan)
3. **Creating feature branch and committing** to the repo

---

## Phase 1: Architectural clarification

### Task 1.1: Stream helpers decision

**Status**: ✅ **COMPLETED** — Retain both functions with annotation

**Files**: `src-rust/crates/api/src/providers/codex.rs`

**Functions investigated**:
- `parse_stream_frame()` at line 306 — parses SSE frame JSON
- `stream_synthetic_response()` at line 637 — wraps ProviderResponse as synthetic stream

**Investigation result**:
- Both currently unused in direct SSE streaming path
- Git history confirms `stream_synthetic_response()` was part of earlier working implementation:
  - Previously called: `Ok(self.stream_synthetic_response(response))`
  - Current approach uses direct SSE with inline parsing
- **Both are proven code from working implementation**, not experimental scaffolding

**Decision**: **RETAIN both functions with `#[allow(dead_code)]` annotation + documentation**
- **Future-proof**: If API changes or non-streaming fallback needed, infrastructure is ready
- **Defensive design**: Better to preserve battle-tested code than delete and re-implement
- **Zero cost**: Compiler still type-checks; no runtime overhead
- **Consistent pattern**: Aligns with streaming helpers in CopilotProvider and other providers

Why not delete: Deleting working code just to silence warnings is poor practice.

**Implementation**:
- [ ] Add `#[allow(dead_code)]` above `parse_stream_frame()` with doc: "SSE frame parsing utility for streaming optimization"
- [ ] Add `#[allow(dead_code)]` above `stream_synthetic_response()` with doc: "Fallback wrapper for non-streaming responses; used in earlier implementation, available for future non-streaming or cached-response scenarios"
- [ ] Run `cargo check -p claurst-api` to verify warning suppressed

---

### Task 1.2: Unused loop label

**Status**: ✅ **COMPLETED** — Delete the label

**File**: `src-rust/crates/api/src/providers/codex.rs` (line 748)

**Investigation result**:
- `'outer:` label found on while-let loop at line 748
- **No references found** — no `break 'outer` or `continue 'outer` statements exist in codebase
- Label serves no purpose; safe to delete

**Decision**: **DELETE the label**
- Removes unused label warning
- No other code depends on it
- Simplifies loop structure

**Implementation**:
- [ ] Delete `'outer:` label from line 748
- [ ] Run `cargo check -p claurst-api` to verify warning gone

---

## Phase 2: Test implementation

### Test 2.1: Non-OAuth baseline

**Purpose**: Verify no regression in main provider pipeline

**Command**:
```bash
cargo build --release 2>&1 | grep -E "^error|Finished"
./target/release/claurst --provider anthropic --model claude-opus-4-7 \
  -p "Say pong"
```

**Expected**: Builds cleanly; Anthropic provider responds normally

**Pass criteria**: Response received, no errors

---

### Test 2.2: Codex OAuth initialization

**Purpose**: Verify stored tokens are loadable and valid

**Precondition**: User must have run `/connect → OpenAI Codex` in TUI at least once  
**Token location**: `~/.claurst/codex_tokens.json`

**Command**:
```bash
cat ~/.claurst/codex_tokens.json | jq . 2>&1
```

**Expected output**:
```json
{
  "access_token": "eyJ...",
  "refresh_token": "rt_...",
  "account_id": "...",
  "expires_at": 1778559610
}
```

**Pass criteria**: All fields present, valid JSON, expires_at is Unix timestamp in future

---

### Test 2.3: Codex non-streaming request

**Purpose**: Verify OAuth token is loaded and request succeeds

**Command**:
```bash
./target/release/claurst --provider codex --model gpt-5.4-mini \
  -p "Say pong"
```

**Expected behavior**:
1. Provider loads token from `~/.claurst/codex_tokens.json`
2. Checks expiry (now + 60 < expires_at, so no refresh needed)
3. Sends POST to `https://chatgpt.com/backend-api/codex/responses` with `Authorization: Bearer <token>`
4. Parses response JSON for `output[].content[].text`
5. Returns assembled text response

**Pass criteria**: 
- No auth errors
- Response contains text (e.g., "pong")
- No timeout or connection errors

---

### Test 2.4: Codex streaming request

**Purpose**: Verify streaming path emits SSE events correctly

**Setup**:
```bash
./target/release/claurst  # Start TUI
```

**In TUI**:
```
/model                    # Should show Codex models available
<select gpt-5.4-mini>
/effort low               # Optional
Say hello                 # Send query
```

**Expected behavior**:
1. TUI sends streaming request to provider
2. Provider yields `StreamEvent::MessageStart`
3. Provider yields `StreamEvent::ContentBlockStart`
4. Provider yields multiple `StreamEvent::TextDelta` as bytes arrive
5. Provider yields `StreamEvent::ContentBlockStop`
6. Provider yields `StreamEvent::MessageDelta` with final usage
7. Provider yields `StreamEvent::MessageStop`
8. Text assembles on screen incrementally

**Pass criteria**:
- Text appears on screen as it arrives (not buffered)
- Response completes without error
- No missing events

---

### Test 2.5: Token refresh validation

**Purpose**: Verify expired tokens are refreshed before request

**Setup**:
```bash
# Edit ~/.claurst/codex_tokens.json
# Set expires_at to current Unix time - 10 (in the past)
# Example: current time is 1777695610, set expires_at to 1777695600

jq '.expires_at = (now | floor) - 10' ~/.claurst/codex_tokens.json > /tmp/codex_tokens.json.tmp
mv /tmp/codex_tokens.json.tmp ~/.claurst/codex_tokens.json
```

**Command**:
```bash
./target/release/claurst --provider codex --model gpt-5.4-mini \
  -p "Say pong" 2>&1
```

**Expected behavior**:
1. Provider checks expiry: `now + 60 >= expires_at` is true
2. Provider calls `refresh_token(&refresh_token)` 
3. Sends POST to `CODEX_TOKEN_URL` with `grant_type=refresh_token`
4. Parses response, extracts `access_token`, `refresh_token`, `expires_in`
5. Updates in-memory token cache (Arc<Mutex>)
6. Persists to `~/.claurst/codex_tokens.json` with `save_codex_tokens()`
7. Retries original request with new token
8. Request succeeds

**Pass criteria**:
- Response received
- `~/.claurst/codex_tokens.json` shows updated `expires_at` (future timestamp)
- No auth errors
- Completes in reasonable time (refresh + request)

---

### Test 2.6: Tool-calling end-to-end

**Purpose**: Verify tool-calling detection and parsing still works with OAuth tokens

**Setup**:
```bash
./target/release/claurst  # Start TUI
```

**In TUI**:
```
/model                    # Select gpt-5.4-mini
/connect tools            # Ensure tools/skills are loaded
<query that should invoke a tool>
```

**Examples**:
- "What time is it?" (system call, if available)
- "Search for [topic]" (search tool, if available)
- "Calculate 2+2" (calculator tool, if available)

**Expected behavior**:
1. Request sent with tool definitions
2. Codex responds with `output[].content[].function_call` block
3. Response parser detects `function_call` and creates `ContentBlock::ToolUse`
4. Stop reason set to `StopReason::ToolUse`
5. Tool invocation passed to skill/tool handler
6. Tool result assembled and sent in next turn

**Pass criteria**:
- Tool invocation detected (shows tool name, arguments)
- Tool executed without error
- Follow-up response incorporates tool result

---

### Test 2.7: 401 retry with token refresh (optional)

**Purpose**: Verify one-time retry on 401 with token refresh

**Setup** (manual/advanced):
```bash
# Corrupt the access_token in ~/.claurst/codex_tokens.json
# to an invalid value (short string), but keep refresh_token valid
jq '.access_token = "invalid"' ~/.claurst/codex_tokens.json > /tmp/codex_tokens.json.tmp
mv /tmp/codex_tokens.json.tmp ~/.claurst/codex_tokens.json
```

**Command**:
```bash
./target/release/claurst --provider codex --model gpt-5.4-mini \
  -p "Say pong" 2>&1
```

**Expected behavior**:
1. First request with invalid token → 401 from server
2. Provider detects 401 and calls `refresh_token(&refresh_token)`
3. Gets new valid `access_token`
4. Updates in-memory cache and persists
5. Retries request with new token
6. Request succeeds

**Pass criteria**:
- Response received (no persistent auth error)
- `~/.claurst/codex_tokens.json` shows refreshed token
- Request succeeded on retry

---

## Phase 3: Code finalization

### Task 3.1: Remove or suppress warnings

**After** completing architectural decisions (1.1, 1.2):

**Actions**:
- [ ] If `parse_stream_frame()` and `stream_synthetic_response()` are deleted: no warning needed
- [ ] If retained: add `#[allow(dead_code)]` above each function and doc comment
- [ ] If unused label is deleted: no warning needed
- [ ] If retained: add `#[allow(unused_labels)]` or document why it exists

**Verify**:
```bash
cargo check -p claurst-api 2>&1 | grep -i warning
```

Expected: No warnings (or only documented ones)

---

### Task 3.2: Create feature branch

**Commands**:
```bash
cd /home/vic/Downloads/claurst-git/
git checkout -b feature/codex-oauth-tokens
```

---

### Task 3.3: Commit changes

**Commit message** (use exact message from CODEX_REVIEW.md):

```
feat(provider/codex): OAuth token lifecycle and refresh support

- Store and load OAuth tokens from ~/.claurst/codex_tokens.json
- Proactively refresh tokens 60 seconds before expiry
- One-time 401 retry on auth failure with token refresh
- Consolidate error handling via shared parse_error_response()
- Support account-scoped requests via ChatGPT-Account-Id header
- Preserve tool-calling behavior by reusing CopilotProvider translation
- Add from_stored() constructor for OAuth-backed provider initialization

Token refresh uses OpenAI's /token endpoint with grant_type=refresh_token.
Refreshed tokens are persisted immediately and cached in-memory.

Implements OAuth token persistence as specified in CHANGE_SUMMARY_CODEX_OAUTH.md

Cleanup: Remove unused Role import from context_collapse.rs
```

**Commands**:
```bash
git add -A
git commit -m "$(cat <<'EOF'
feat(provider/codex): OAuth token lifecycle and refresh support

- Store and load OAuth tokens from ~/.claurst/codex_tokens.json
- Proactively refresh tokens 60 seconds before expiry
- One-time 401 retry on auth failure with token refresh
- Consolidate error handling via shared parse_error_response()
- Support account-scoped requests via ChatGPT-Account-Id header
- Preserve tool-calling behavior by reusing CopilotProvider translation
- Add from_stored() constructor for OAuth-backed provider initialization

Token refresh uses OpenAI's /token endpoint with grant_type=refresh_token.
Refreshed tokens are persisted immediately and cached in-memory.

Implements OAuth token persistence as specified in CHANGE_SUMMARY_CODEX_OAUTH.md

Cleanup: Remove unused Role import from context_collapse.rs
EOF
)"

git log --oneline -3
```

---

## Success criteria

- [x] Phase 1: Architectural decisions made and documented
  - [x] Stream helpers: RETAIN with `#[allow(dead_code)]` annotations (proven code, future-proof design)
  - [x] Unused label: DELETE 'outer: (no references, safe removal)
  - [x] Auth pattern: SOUND (no changes needed)
- [ ] Phase 2: All 7 tests pass (or optional ones noted as skipped)
- [ ] Phase 3: Feature branch created and committed
- [ ] No compiler warnings in `claurst-api` (or suppressed with documented rationale)
- [ ] `git log` shows the feature commit with correct message

---

## Next steps after commit

1. **Push to remote** (if desired):
   ```bash
   git push origin feature/codex-oauth-tokens
   ```

2. **Create pull request** (on GitHub or via gh CLI):
   ```bash
   gh pr create --title "feat(provider/codex): OAuth token lifecycle and refresh support" \
     --body "Implements OAuth token persistence, expiry-aware refresh, and 401 retry logic for the Codex provider. See CHANGE_SUMMARY_CODEX_OAUTH.md and CODEX_REVIEW.md for details."
   ```

3. **Merge to main** (after review):
   ```bash
   git checkout main
   git pull
   git merge --no-ff feature/codex-oauth-tokens
   git push origin main
   ```

---

## Quick reference: test commands

```bash
# Build
cargo build --release

# Test 2.1: baseline
./target/release/claurst --provider anthropic --model claude-opus-4-7 -p "Say pong"

# Test 2.2: token file
cat ~/.claurst/codex_tokens.json | jq .

# Test 2.3: non-streaming
./target/release/claurst --provider codex --model gpt-5.4-mini -p "Say pong"

# Test 2.4: streaming (TUI)
./target/release/claurst

# Test 2.5: refresh (expire token first)
jq '.expires_at = (now | floor) - 10' ~/.claurst/codex_tokens.json > /tmp/t.json && mv /tmp/t.json ~/.claurst/codex_tokens.json
./target/release/claurst --provider codex --model gpt-5.4-mini -p "Say pong"

# Test 2.7: retry (corrupt token first)
jq '.access_token = "invalid"' ~/.claurst/codex_tokens.json > /tmp/t.json && mv /tmp/t.json ~/.claurst/codex_tokens.json
./target/release/claurst --provider codex --model gpt-5.4-mini -p "Say pong"
```
