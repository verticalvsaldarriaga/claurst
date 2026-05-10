# Codex OAuth / Provider Change Summary

Status: updated 2026-05-10 with session 2 fixes.
Date: 2026-05-02 (initial) / 2026-05-10 (follow-up)
Repo: `claurst-git`

## Executive summary

This document summarizes the current Codex-related changes and adjacent cleanup in the repo, with emphasis on OAuth behavior, provider request/response handling, build status, known warnings, and implementation cautions.

The current implementation in `src-rust/crates/api/src/providers/codex.rs` is already structured around persisted OAuth tokens rather than a raw API key. It preserves the current request-building and response-parsing model needed for tool calling, while adding token lifecycle support and shared error parsing.

A small adjacent cleanup was also made in `src-rust/crates/core/src/context_collapse.rs`.

## Files covered

- `src-rust/crates/api/src/providers/codex.rs`
- `src-rust/crates/core/src/context_collapse.rs`

## Build status

Verified with:

```bash
cargo check -p claurst-api
```

Result:
- build passes
- warnings remain in `codex.rs`
- no blocking compile errors for `claurst-api`

Observed warnings:
- unused label in `codex.rs`
- dead-code warnings for helper functions in `codex.rs`

## Detailed changes

### 1. Codex provider now uses stored OAuth tokens

File:
- `src-rust/crates/api/src/providers/codex.rs`

The provider is structured around persisted `CodexTokens` loaded from core config instead of a single API key string.

Current imports confirm use of:
- `claurst_core::oauth_config::{get_codex_tokens, save_codex_tokens, CodexTokens}`
- `claurst_core::codex_oauth::{CODEX_API_ENDPOINT, CODEX_MODELS, CODEX_TOKEN_URL, DEFAULT_CODEX_MODEL}`

#### Provider state

`CodexProvider` stores:
- `id: ProviderId`
- `http_client: reqwest::Client`
- `tokens: Arc<Mutex<CodexTokens>>`

This means token state can be refreshed and updated in place.

### 2. Added/confirmed stored-token constructor

The provider includes:

- `pub fn new(tokens: CodexTokens) -> Self`
- `pub fn from_stored() -> Option<Self>`

`from_stored()`:
- loads stored tokens with `get_codex_tokens()`
- returns `None` if no usable token is present
- returns a ready provider otherwise

This supports OAuth-backed initialization without changing request translation semantics.

### 3. Expiry-aware access token resolution

The provider includes token lifecycle helpers:

- `fn is_expired(tokens: &CodexTokens) -> bool`
- `async fn access_token(&self) -> Result<String, ProviderError>`
- `async fn refresh_token(&self, refresh_token: &str) -> Result<String, ProviderError>`

Behavior:
- token expiry is checked using `expires_at`
- a 60-second skew is used to proactively refresh before expiry
- if expired and a refresh token exists, the provider refreshes before making the API request
- refreshed tokens are persisted with `save_codex_tokens(...)`
- the in-memory token cache is updated after refresh

### 4. Codex OAuth refresh endpoint usage

Refresh requests use:
- `CODEX_TOKEN_URL`
- `CODEX_CLIENT_ID`

Refresh body:

```json
{
  "grant_type": "refresh_token",
  "client_id": "...",
  "refresh_token": "..."
}
```

Refresh response handling includes:
- HTTP status capture
- body text capture
- JSON parse validation
- validation that `access_token` exists
- optional update of `refresh_token`
- optional update of `expires_at` from `expires_in`

### 5. One-time retry on 401

The provider’s request path now includes a safe one-time retry policy.

In `send_responses_request(...)`:
- request is sent with current/proactively refreshed access token
- if HTTP 401 occurs on first attempt:
  - a refresh is attempted using stored refresh token
  - the request is retried once
- if no refresh token is available, an auth failure is returned
- non-2xx responses are classified via shared error parsing

Equivalent logic also exists for `send_responses_streaming_request(...)`.

This is the key reliability improvement for OAuth without changing request/message semantics.

### 6. Shared error parsing integration

The provider uses:
- `crate::error_handling::parse_error_response`

This improves consistency with other providers and avoids ad hoc response classification in the main HTTP request path.

Benefits:
- more consistent auth/rate-limit/request failures
- standardized provider errors
- easier maintenance across providers

### 7. Account-scoped header support

The provider supports account-scoped requests via:
- `fn account_id(&self) -> Option<String>`
- conditional `ChatGPT-Account-Id` header injection in `codex_headers(...)`

Header behavior:
- bearer token is always attached
- `ChatGPT-Account-Id` is attached only if account metadata exists

### 8. Current request translation remains compatible-focused

Important design choice:
- the provider reuses Copilot’s public Responses-API translation helper for input generation
- it keeps current body-building and parsing behavior for tool-calling compatibility

Request-building path includes:
- `CopilotProvider::to_responses_input_pub(request)`
- `system_prompt_to_text(request)`
- current request tool serialization

This is significant because the goal was to improve OAuth/error handling without reintroducing the regressions seen in the older patched `0.0.8` provider.

### 9. Tool-calling support remains in response parsing

`parse_responses_response(...)` currently handles:
- text output blocks
- reasoning blocks
- function/tool call blocks

Specifically:
- `function_call` output items are parsed into `ContentBlock::ToolUse`
- parsed tool calls set `has_tool_call = true`
- `stop_reason` becomes `StopReason::ToolUse`

This is one of the critical reasons not to replace the entire provider implementation with the older patched version.

### 10. Synthetic streaming helpers still exist

The following helpers are present in `codex.rs`:
- `parse_stream_frame(...)`
- `stream_synthetic_response(...)`

Current status:
- they are compiled
- they are currently unused according to compiler warnings
- they were not removed

This matters because they may be intended delegates for future or partially wired streaming behavior, and should not be deleted blindly just to silence warnings.

### 11. Non-stream response parsing flow fixed to preserve status context

The current implementation captures `(status, text)` before JSON deserialization in `send_responses_request(...)`, which ensures JSON parse errors can still produce a `ProviderError` that includes the original HTTP status and response body context.

This is a small but important robustness improvement for diagnostics.

### 12. Auth error variant normalization

The provider uses `ProviderError::AuthFailed` for rejected/invalid token cases.

This avoids mismatches with nonexistent or inconsistent auth error variants and aligns provider auth failures with the repo’s current error model.

## Context collapse cleanup

File:
- `src-rust/crates/core/src/context_collapse.rs`

Change:
- removed an unused `Role` import from the top of the file

Current top-level import is:

```rust
use crate::types::Message;
```

Important note:
- no context collapse functions were removed
- the service logic remains intact

## Current warnings / unresolved follow-up items

### In `codex.rs`

1. Unused label
- warning points at a labeled loop: `'outer:`
- this can likely be removed if no loop control uses it
- this is warning-only, not a behavior failure

2. Dead-code warnings
- `parse_stream_frame(...)`
- `stream_synthetic_response(...)`

These helpers appear to be intended delegates for stream mediation / synthetic streaming paths, but are not currently called.

Important clarification:
- they were identified by the compiler as unused
- they should not be removed merely to silence warnings
- they are important to the intended Codex streaming/delegate architecture and should instead be wired correctly or explicitly retained as intentional scaffolding

Recommended next step:
- do not delete them blindly
- either wire them into the intended streaming path or explicitly annotate/document why they are retained

## Why the provider was not wholesale replaced with the older patched version

The older patched `0.0.8` Codex provider reportedly introduced regressions in:
- tool calling
- skill generation / spec-defined behavior

Because of that, the safe strategy used here is:
- keep current request-building semantics
- keep current response parsing semantics
- add OAuth token lifecycle improvements
- add shared error handling
- avoid broad translation-layer replacement

## Recommended next refactor steps

### Safe next step
1. Wire the existing stream delegates correctly
   - review where `parse_stream_frame(...)` should mediate SSE frame parsing
   - review whether non-native stream fallback should use `stream_synthetic_response(...)`
   - preserve current tool-calling behavior while doing so

### Optional cleanup
2. Remove the unused loop label if truly unnecessary

### Verification
3. After any delegate wiring:
   - run `cargo check -p claurst-api`
   - test Codex non-stream requests
   - test Codex streaming requests
   - test tool-calling path
   - test skill-generation/spec-sensitive path

## TODO / follow-up tracking

The following items remain intentionally open and should be tracked as next-step work rather than treated as completed refactor work:

- Wire Codex delegate helpers into the intended streaming path, or explicitly document them as retained scaffolding.
- Validate Codex OAuth refresh behavior with an end-to-end authenticated request.
- Validate Codex tool-calling behavior after any streaming/delegate changes.
- Validate skill-generation/spec-sensitive behavior after any Codex provider changes.
- Clean up warning-only issues such as the unused loop label once delegate intent is resolved.

## Risk notes

- Do not replace request translation wholesale without re-validating tool calling.
- Do not delete “unused” stream helpers unless confirmed obsolete.
- Do not change context-collapse behavior as part of Codex OAuth work.
- Keep OAuth/token logic isolated from message semantic changes.

---

## Session 2 fixes (2026-05-10)

### Anthropic OAuth token refresh — three bugs fixed

**Root cause chain:** The access token in `~/.claurst/oauth_tokens.json` expired on ~May 3. The silent refresh was failing for three independent reasons:

#### Fix 1 — Missing `anthropic-beta` header (`lib.rs`)
The refresh POST to `https://platform.claude.com/v1/oauth/token` was missing the required `anthropic-beta: oauth-2025-04-20` header. The endpoint rejects requests without it.

#### Fix 2 — Invalid scope in refresh body (`lib.rs`)
The refresh body included `"scope": ALL_SCOPES.join(" ")` which contains `org:create_api_key`. The token endpoint rejects this with `invalid_scope` on a refresh grant. Per RFC 6749 the scope field should be omitted entirely on refresh — the server reissues the original scopes.

#### Fix 3 — Silent failure swallowed the error (`lib.rs`)
The original code used `break 'refresh None` on any non-2xx without logging the HTTP status or body, making the `invalid_scope` response invisible. Now logs status code and response body at `WARN` level.

### Refactor — `OAuthTokens::refresh()` method (`lib.rs`)

The inline 40-line refresh block in `resolve_anthropic_auth_async` was extracted into a proper `pub async fn refresh(&self) -> Option<Self>` method on `OAuthTokens` inside `pub mod oauth`. `resolve_anthropic_auth_async` now calls `tokens.refresh().await`. All three fixes above are in the new method.

### Voice PTT — plain-V binding removed (`tui/src/app.rs`)

`app.rs` was restored from upstream main (recovering the full voice recording implementation lost in the merge conflict at `370c7fa`), then the plain `V` key hold-to-talk binding and its key-release handler were intentionally removed. The `Alt+V` toggle remains the only PTT activation path.

---

## Short summary

The repo’s Codex provider is now or remains in a safer architecture for current needs:
- OAuth token persistence and refresh support
- one-time retry on 401
- shared error parsing
- preserved tool-calling response parsing
- minimal adjacent cleanup in context collapse

The code builds for `claurst-api`, but there are still warning-level follow-ups in Codex streaming delegate wiring.
