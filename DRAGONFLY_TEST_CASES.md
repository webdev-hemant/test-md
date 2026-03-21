# Dragonfly Integration — Test Cases

> **Feature**: Dragonfly hot-path cache for credit enforcement, membership checks, and rate limiting  
> **Scope**: NitroStudio Gateway (`gateway/`)  
> **Last updated**: 2026-03-21

---

## Table of Contents

1. [Connection & Initialization](#1-connection--initialization)
2. [Startup Seeding](#2-startup-seeding)
3. [Studio Access Middleware — Dragonfly Fast Path](#3-studio-access-middleware--dragonfly-fast-path)
4. [Studio Access Middleware — MongoDB Fallback](#4-studio-access-middleware--mongodb-fallback)
5. [Rate Limiting](#5-rate-limiting)
6. [Billing & Credit Increment — Dollar Mode](#6-billing--credit-increment--dollar-mode)
7. [Billing & Credit Increment — Token Mode](#7-billing--credit-increment--token-mode)
8. [Model Validation](#8-model-validation)
9. [Webhook Handlers](#9-webhook-handlers)
10. [Period Reset](#10-period-reset)
11. [Reconciliation](#11-reconciliation)
12. [Health Check](#12-health-check)
13. [Field Protection & Allowlists](#13-field-protection--allowlists)
14. [Degraded Mode & Failure Scenarios](#14-degraded-mode--failure-scenarios)
15. [Concurrency & Atomicity](#15-concurrency--atomicity)
16. [Performance & Latency](#16-performance--latency)
17. [Security](#17-security)

---

## 1. Connection & Initialization

### TC-1.1: Successful Dragonfly connection

| | |
|---|---|
| **Precondition** | Dragonfly instance running at configured `DRAGONFLY_ADDR` |
| **Steps** | 1. Set `DRAGONFLY_ENABLED=true` in `.env` <br> 2. Start the gateway |
| **Expected** | Gateway logs `Dragonfly store initialized`. `NewDragonflyStore` returns no error. PING succeeds within 5s timeout. |

### TC-1.2: Connection with TLS enabled

| | |
|---|---|
| **Precondition** | Dragonfly instance with TLS configured |
| **Steps** | 1. Set `DRAGONFLY_TLS=true`, `DRAGONFLY_TLS_INSECURE=false` <br> 2. Provide valid TLS certs <br> 3. Start the gateway |
| **Expected** | TLS handshake succeeds. PING returns no error. Store is initialized. |

### TC-1.3: Connection with TLS — insecure skip verify

| | |
|---|---|
| **Precondition** | Dragonfly with self-signed cert |
| **Steps** | 1. Set `DRAGONFLY_TLS=true`, `DRAGONFLY_TLS_INSECURE=true` <br> 2. Start the gateway |
| **Expected** | Connection succeeds despite self-signed cert. No TLS verification errors. |

### TC-1.4: Connection failure — wrong address

| | |
|---|---|
| **Precondition** | No Dragonfly on `localhost:9999` |
| **Steps** | 1. Set `DRAGONFLY_ADDR=localhost:9999` <br> 2. Start the gateway |
| **Expected** | `NewDragonflyStore` returns error containing "dragonfly ping". Gateway falls back to MongoDB-only mode. |

### TC-1.5: Connection failure — wrong password

| | |
|---|---|
| **Precondition** | Dragonfly requires auth |
| **Steps** | 1. Set `DRAGONFLY_PASSWORD=wrong_password` <br> 2. Start the gateway |
| **Expected** | PING fails with auth error. Gateway falls back to MongoDB-only mode. |

### TC-1.6: Connection with username + password

| | |
|---|---|
| **Precondition** | Dragonfly configured with ACL user |
| **Steps** | 1. Set `DRAGONFLY_USERNAME=studio`, `DRAGONFLY_PASSWORD=<correct>` <br> 2. Start the gateway |
| **Expected** | Connection succeeds with rueidis username/password auth. |

### TC-1.7: Dragonfly disabled via config

| | |
|---|---|
| **Precondition** | `DRAGONFLY_ENABLED=false` |
| **Steps** | 1. Start the gateway |
| **Expected** | No Dragonfly connection is attempted. All middleware operates in MongoDB-only mode. `df` is nil in middleware structs. |

---

## 2. Startup Seeding

### TC-2.1: Full seed on fresh Dragonfly

| | |
|---|---|
| **Precondition** | Empty Dragonfly. MongoDB has 3 orgs, 2 with active subscriptions, 8 members. |
| **Steps** | 1. Set `DRAGONFLY_SEED_ON_STARTUP=true` <br> 2. Start the gateway |
| **Expected** | - 3 org keys created (`gway:org:{id}`) <br> - 8 member keys created (`gway:member:{orgId}:{userId}`) <br> - Log shows `Completed in Xms — 3 orgs, 8 members seeded` <br> - Org without subscription has zero overage fields |

### TC-2.2: Seed preserves existing counters

| | |
|---|---|
| **Precondition** | Dragonfly already has `gway:org:ABC` with `creditsUsed=500`. MongoDB has `creditsTotal=5000` for the same org. |
| **Steps** | 1. Restart the gateway with `DRAGONFLY_SEED_ON_STARTUP=true` |
| **Expected** | - `creditsTotal` updated to 5000 (config field) <br> - `creditsUsed` remains at 500 (counter field NOT overwritten) <br> - Seeder uses `orgStateToConfigOnlyMap` and `SetOrgConfigFields` for existing keys |

### TC-2.3: Seed preserves existing member counters

| | |
|---|---|
| **Precondition** | Dragonfly has `gway:member:ABC:USER1` with `creditsUsed=200`. MongoDB has `studioAccess=true`, `studioCreditLimit=1000`. |
| **Steps** | 1. Restart the gateway |
| **Expected** | - `studioAccess`, `creditLimit`, `additionalLimit`, `version` updated from MongoDB <br> - `creditsUsed` remains at 200 (counter preserved) |

### TC-2.4: Seed with org credits from `studio_org_credits` collection

| | |
|---|---|
| **Precondition** | Org has both `studioCredits: 1000` and a `studio_org_credits` record with `creditsTotal: 5000`. |
| **Steps** | 1. Start the gateway |
| **Expected** | `creditsTotal` in Dragonfly is 5000 (`studio_org_credits` takes priority over org.studioCredits). |

### TC-2.5: Seed with legacy `credits` field

| | |
|---|---|
| **Precondition** | Org has `studioCredits: 0`, `credits: 3000` (legacy field). No `studio_org_credits` record. |
| **Steps** | 1. Start the gateway |
| **Expected** | `creditsTotal` in Dragonfly is 3000 (falls back to legacy `credits` field). |

### TC-2.6: Seed skips deleted memberships

| | |
|---|---|
| **Precondition** | MongoDB has a membership with `isDeleted: true` |
| **Steps** | 1. Start the gateway |
| **Expected** | Deleted membership is NOT seeded into Dragonfly. No `gway:member:` key created for it. |

### TC-2.7: Seed with no active subscriptions

| | |
|---|---|
| **Precondition** | All subscriptions in MongoDB have `status: "CANCELLED"` |
| **Steps** | 1. Start the gateway |
| **Expected** | Orgs seeded with zero overage fields. `OverageEnabled=0`, `OverageLimit=0`, `SubCredits=0`. |

### TC-2.8: Seed disabled

| | |
|---|---|
| **Precondition** | `DRAGONFLY_SEED_ON_STARTUP=false` |
| **Steps** | 1. Start the gateway |
| **Expected** | No seeding occurs. Dragonfly starts empty. Keys populated on-demand via `SeedSingleOrg`/`SeedSingleMember` when first request arrives. |

### TC-2.9: Seed with subscription `planType` override

| | |
|---|---|
| **Precondition** | Org has `plan: "FREE"`. Subscription has `planType: "PRO"`, `status: "ACTIVE"`. |
| **Steps** | 1. Start the gateway |
| **Expected** | Dragonfly `plan` field is `"PRO"` (subscription `planType` overrides org `plan`). |

---

## 3. Studio Access Middleware — Dragonfly Fast Path

### TC-3.1: Owner access — happy path

| | |
|---|---|
| **Precondition** | Org state exists in Dragonfly. `ownerId` matches request `user_id`. Credits available. |
| **Steps** | 1. Send `POST /v1/chat/completions` with valid JWT |
| **Expected** | - Only 1 Dragonfly HGETALL (org state) — no member lookup for owner <br> - `c.Locals("is_owner")` = true <br> - `c.Locals("df_org_state")` populated <br> - `c.Locals("df_member_state")` is nil <br> - Request proceeds to chat handler |

### TC-3.2: Member access — happy path

| | |
|---|---|
| **Precondition** | Org + member state in Dragonfly. `studioAccess=1`. Credits available. |
| **Steps** | 1. Send request as non-owner member |
| **Expected** | - 2 Dragonfly HGETALLs (org + member) <br> - `c.Locals("is_owner")` = false <br> - `c.Locals("df_member_state")` populated <br> - Request proceeds |

### TC-3.3: Member denied — studioAccess disabled

| | |
|---|---|
| **Precondition** | Member has `studioAccess=0` in Dragonfly |
| **Steps** | 1. Send request as member |
| **Expected** | 403 Forbidden. `code: "studio_access_denied"`. Message: "Studio access has not been enabled for your account." |

### TC-3.4: Non-member access denied

| | |
|---|---|
| **Precondition** | No member key in Dragonfly for user. MongoDB membership query returns `ErrNoDocuments`. |
| **Steps** | 1. Send request as user who is not a member |
| **Expected** | 403 Forbidden. "You are not a member of this organization". |

### TC-3.5: Missing user or org context

| | |
|---|---|
| **Precondition** | JWT parsed but `user_id` or `organization_id` missing from locals |
| **Steps** | 1. Send request without org context |
| **Expected** | 401 Unauthorized. "Missing user or organization context". |

### TC-3.6: Org credits exhausted — no overage

| | |
|---|---|
| **Precondition** | `creditsTotal=5000`, `creditsUsed=5000`, `overageEnabled=0` |
| **Steps** | 1. Send request as owner |
| **Expected** | 402 Payment Required. `code: "user_limit_exceeded"`. Message about exhausted credits. |

### TC-3.7: Org credits exhausted — member sees different error

| | |
|---|---|
| **Precondition** | Same as TC-3.6 but request from a non-owner member |
| **Steps** | 1. Send request as member |
| **Expected** | 402 Payment Required. `code: "hard_limit_reached"`. Message: "Your organization has exhausted its studio credits. Please contact your organization admin." |

### TC-3.8: Org credits exhausted — overage enabled, within limit

| | |
|---|---|
| **Precondition** | `creditsTotal=5000`, `creditsUsed=5000`, `overageEnabled=1`, `overageLimit=10000`, `overageUsed=3000` |
| **Steps** | 1. Send request |
| **Expected** | Request proceeds. `c.Locals("is_overage")` = true. |

### TC-3.9: Overage limit reached

| | |
|---|---|
| **Precondition** | `creditsUsed >= creditsTotal`, `overageEnabled=1`, `overageUsed >= overageLimit` |
| **Steps** | 1. Send request |
| **Expected** | 402 Payment Required. `code: "hard_limit_reached"`. Details include overage limit and used. |

### TC-3.10: Member personal limit exhausted

| | |
|---|---|
| **Precondition** | Member has `creditLimit=1000`, `additionalLimit=500`, `creditsUsed=1500` |
| **Steps** | 1. Send request as member |
| **Expected** | 402 Payment Required. `code: "credits_exhausted"`. Details: `limit=1500`, `used=1500`. |

### TC-3.11: Member with no personal limit (`creditLimit=-1`)

| | |
|---|---|
| **Precondition** | Member has `creditLimit=-1` (use org default) |
| **Steps** | 1. Send request as member with available org credits |
| **Expected** | No personal limit check applied. `PersonalLimit()` returns -1. Request proceeds based on org-level limits only. |

### TC-3.12: Free token mode — tokens available

| | |
|---|---|
| **Precondition** | `isFreeCreditStudio=1`, `freeStudioCreditType="tokens"`, `freeCreditAmount=100000`, `freeTokensUsed=50000` |
| **Steps** | 1. Send request |
| **Expected** | `c.Locals("is_free_token_mode")` = true. Request proceeds. |

### TC-3.13: Free token mode — tokens exhausted

| | |
|---|---|
| **Precondition** | `isFreeCreditStudio=1`, `freeStudioCreditType="tokens"`, `freeCreditAmount=100000`, `freeTokensUsed=100000` |
| **Steps** | 1. Send request |
| **Expected** | 402 Payment Required. `code: "credits_exhausted"`. Details: `creditType: "tokens"`, `limit: 100000`, `used: 100000`. |

### TC-3.14: On-demand seed when org key missing

| | |
|---|---|
| **Precondition** | Org NOT in Dragonfly but exists in MongoDB |
| **Steps** | 1. Send request for this org |
| **Expected** | `SeedSingleOrg` called. Org state populated in Dragonfly. Request proceeds with seeded data. Subsequent requests hit Dragonfly directly. |

### TC-3.15: On-demand seed when member key missing

| | |
|---|---|
| **Precondition** | Org in Dragonfly. Member NOT in Dragonfly but exists in MongoDB. |
| **Steps** | 1. Send request as that member |
| **Expected** | `SeedSingleMember` called. Member state populated. Request proceeds. |

### TC-3.16: Usage status endpoint bypasses middleware

| | |
|---|---|
| **Precondition** | Any state |
| **Steps** | 1. Send `GET /v1/studio/usage/status` |
| **Expected** | Middleware calls `c.Next()` immediately — no access checks. |

### TC-3.17: Free credit mode — org credits check skipped

| | |
|---|---|
| **Precondition** | `isFreeCreditStudio=1`, `freeStudioCreditType="dollar"`, `creditsTotal=0`, `creditsUsed=0` |
| **Steps** | 1. Send request |
| **Expected** | Org-level dollar credit check skipped (guarded by `!orgState.IsFreeCreditMode()`). Request proceeds based on free credit amount. |

### TC-3.18: MongoDB repo nil — service unavailable

| | |
|---|---|
| **Precondition** | Gateway started in degraded mode with `repo=nil` |
| **Steps** | 1. Send request |
| **Expected** | 503 Service Unavailable. `code: "studio_service_unavailable"`. |

---

## 4. Studio Access Middleware — MongoDB Fallback

### TC-4.1: Fallback when Dragonfly returns error

| | |
|---|---|
| **Precondition** | Dragonfly connected but `HGETALL` fails (e.g., timeout) |
| **Steps** | 1. Send request |
| **Expected** | Log: "Dragonfly fast path unavailable...falling back to MongoDB". Full validation proceeds via MongoDB path. |

### TC-4.2: Fallback when org key missing and MongoDB unavailable

| | |
|---|---|
| **Precondition** | Org not in Dragonfly. `db` is nil (MongoDB down). |
| **Steps** | 1. Send request |
| **Expected** | Falls back to `validateViaMongo`. If MongoDB also fails, returns appropriate error. |

### TC-4.3: Fallback preserves original MongoDB behavior

| | |
|---|---|
| **Precondition** | Dragonfly unavailable |
| **Steps** | 1. Run same scenarios as TC-3.1 through TC-3.13 via MongoDB path |
| **Expected** | Identical HTTP status codes and error shapes. Context locals (`studio_org`, `studio_sub`, `studio_member`) populated instead of `df_*` locals. |

### TC-4.4: Fallback enabled flag

| | |
|---|---|
| **Precondition** | `DRAGONFLY_FALLBACK_TO_MONGO=true` |
| **Steps** | 1. Verify `FallbackEnabled()` returns true |
| **Expected** | Middleware attempts MongoDB when Dragonfly fails. |

---

## 5. Rate Limiting

### TC-5.1: First request in window — allowed

| | |
|---|---|
| **Precondition** | No rate limit key exists for current minute |
| **Steps** | 1. Send a chat completion request |
| **Expected** | Lua script INCR returns 1. EXPIRE set to 120s. Request allowed. |

### TC-5.2: Under limit — allowed

| | |
|---|---|
| **Precondition** | 5 requests already in current window. `RateLimitRPM=60`. |
| **Steps** | 1. Send another request |
| **Expected** | Lua returns 6 (allowed). Request proceeds. |

### TC-5.3: At exact limit — allowed

| | |
|---|---|
| **Precondition** | 59 requests in window. `RateLimitRPM=60`. |
| **Steps** | 1. Send request #60 |
| **Expected** | Lua returns 60. `60 > 60` is false → allowed. |

### TC-5.4: Over limit — rejected

| | |
|---|---|
| **Precondition** | 60 requests in window. `RateLimitRPM=60`. |
| **Steps** | 1. Send request #61 |
| **Expected** | 429 Too Many Requests. `code: "rate_limit_exceeded"`. Message: "Rate limit exceeded. Maximum requests per minute reached." |

### TC-5.5: Window rolls over — counter resets

| | |
|---|---|
| **Precondition** | Hit rate limit at minute `:30` |
| **Steps** | 1. Wait for minute `:31` <br> 2. Send request |
| **Expected** | New key `gway:rl:{orgId}:{userId}:2026-03-21T14:31`. Counter starts at 1. Request allowed. |

### TC-5.6: TTL auto-expires old windows

| | |
|---|---|
| **Precondition** | Rate limit key from 3 minutes ago |
| **Steps** | 1. Wait 120s+ <br> 2. Check key |
| **Expected** | Key auto-expired by Dragonfly. No stale rate limit data. |

### TC-5.7: Missing orgID/userID — rate limit skipped

| | |
|---|---|
| **Precondition** | Request context missing `organization_id` or `user_id` |
| **Steps** | 1. Send request |
| **Expected** | `rateLimitViaDragonfly` calls `c.Next()` without checking rate limit. |

### TC-5.8: Dragonfly error — fallback to MongoDB

| | |
|---|---|
| **Precondition** | Dragonfly rate limit Lua script fails |
| **Steps** | 1. Send request |
| **Expected** | Log: "Dragonfly error...falling back to MongoDB". MongoDB `GetRateLimitEntry` used. |

### TC-5.9: UpdateRateLimit skipped when Dragonfly active

| | |
|---|---|
| **Precondition** | `df != nil` |
| **Steps** | 1. Call `UpdateRateLimit()` after request |
| **Expected** | Returns nil immediately. No MongoDB `IncrementRateLimit` call (Dragonfly Lua script already handled it atomically). |

---

## 6. Billing & Credit Increment — Dollar Mode

### TC-6.1: Dollar credit increment — within plan (owner)

| | |
|---|---|
| **Precondition** | `creditsTotal=5000`, `creditsUsed=1000`. Cost=50 cents. Owner request. |
| **Steps** | 1. Complete a chat request |
| **Expected** | Lua returns `1` (CreditAllowed). `creditsUsed` incremented to 1050. No member key touched. `CreditUpdate` sent to UsageWorker. |

### TC-6.2: Dollar credit increment — within plan (member)

| | |
|---|---|
| **Precondition** | Same org. Member with `creditLimit=2000`, `creditsUsed=100`. Cost=50. |
| **Steps** | 1. Complete a chat request as member |
| **Expected** | Lua increments both `gway:org:*` creditsUsed (+50) AND `gway:member:*` creditsUsed (+50). Returns `1`. |

### TC-6.3: Dollar credit increment — overage mode

| | |
|---|---|
| **Precondition** | `creditsTotal=5000`, `creditsUsed=5000`, `overageEnabled=1`, `overageLimit=10000`, `overageUsed=100`. Cost=50. |
| **Steps** | 1. Complete a chat request |
| **Expected** | Lua returns `2` (CreditOverage). `overageUsed` incremented to 150. `creditsUsed` NOT incremented. |

### TC-6.4: Dollar credit — org exhausted, no overage

| | |
|---|---|
| **Precondition** | `creditsTotal=5000`, `creditsUsed=5000`, `overageEnabled=0`. Cost=50. |
| **Steps** | 1. Complete a chat request |
| **Expected** | Lua returns `-1` (CreditOrgExhausted). `creditsUsed` still incremented (post-request — tokens already consumed). Log: "Org credits exhausted". |

### TC-6.5: Dollar credit — overage limit reached

| | |
|---|---|
| **Precondition** | `overageEnabled=1`, `overageLimit=1000`, `overageUsed=980`. Cost=50. |
| **Steps** | 1. Complete a chat request |
| **Expected** | Lua returns `-2` (CreditOverageLimit). `overageUsed` incremented to 1030 (exceeds limit). Log: "Overage limit reached". |

### TC-6.6: Dollar credit — member personal limit reached

| | |
|---|---|
| **Precondition** | Member `creditLimit=1000`, `additionalLimit=0`, `creditsUsed=980`. Cost=50. |
| **Steps** | 1. Complete a chat request as member |
| **Expected** | Lua returns `-3` (CreditMemberExhausted). Member `creditsUsed` still incremented to 1030. Org counter also incremented. Log: "Member limit reached". |

### TC-6.7: Dollar credit — member with no personal limit

| | |
|---|---|
| **Precondition** | Member `creditLimit=-1` |
| **Steps** | 1. Complete a chat request |
| **Expected** | Lua skips member limit check (`creditLimit < 0`). Member `creditsUsed` still incremented. Returns org-level result. |

### TC-6.8: Zero cost — no increment

| | |
|---|---|
| **Precondition** | `costCents=0` (e.g., free model, zero tokens) |
| **Steps** | 1. Complete a chat request |
| **Expected** | Dragonfly Lua script NOT called. No increment. Analytics still tracked via UsageWorker. |

### TC-6.9: Dragonfly fails during billing — MongoDB fallback

| | |
|---|---|
| **Precondition** | Dragonfly times out on Lua execution (>2s) |
| **Steps** | 1. Complete a chat request |
| **Expected** | Log: "Dragonfly dollar increment error...falling back to MongoDB". `updateStudioCredits` called synchronously via MongoDB. |

### TC-6.10: Failed response (4xx/5xx) — no billing

| | |
|---|---|
| **Precondition** | Chat completion returns 400/500 from upstream provider |
| **Steps** | 1. Receive error response |
| **Expected** | `statusCode >= 400` check triggers early return. No credit increment. No usage logged to worker (except the initial `c.Next()` pass-through). |

### TC-6.11: CreditUpdate sent to UsageWorker

| | |
|---|---|
| **Precondition** | Dragonfly credit increment succeeds |
| **Steps** | 1. Verify CreditUpdate message |
| **Expected** | Worker receives `CreditUpdate{OrgID, UserID, CostCents, TotalTokens, IsOverage, IsOwner, IsFreeTokenMode}`. Async MongoDB writes follow. |

---

## 7. Billing & Credit Increment — Token Mode

### TC-7.1: Token increment — within limit

| | |
|---|---|
| **Precondition** | `isFreeCreditStudio=1`, `freeStudioCreditType="tokens"`, `freeCreditAmount=100000`, `freeTokensUsed=50000`. Total tokens=1500. |
| **Steps** | 1. Complete a chat request |
| **Expected** | Lua returns 51500 (new total used). `freeTokensUsed` incremented by 1500. |

### TC-7.2: Token increment — tokens exhausted

| | |
|---|---|
| **Precondition** | `freeCreditAmount=100000`, `freeTokensUsed=100000`. Tokens=500. |
| **Steps** | 1. Complete a chat request |
| **Expected** | Lua returns `-1` (TokensExhausted). `freeTokensUsed` still incremented to 100500 (post-request). Log: "Free tokens exhausted". |

### TC-7.3: Token increment — no limit configured

| | |
|---|---|
| **Precondition** | `freeCreditAmount=0` |
| **Steps** | 1. Complete a chat request in token mode |
| **Expected** | Lua returns `-2` (TokenNoLimit). No increment. |

### TC-7.4: Token increment — zero tokens

| | |
|---|---|
| **Precondition** | `totalTokens=0` |
| **Steps** | 1. Complete a request with zero token usage |
| **Expected** | Token credit Lua script NOT called. Guard: `if totalTokens > 0`. |

### TC-7.5: Token mode Dragonfly failure — MongoDB fallback

| | |
|---|---|
| **Precondition** | Dragonfly fails during token credit check |
| **Steps** | 1. Complete a chat request |
| **Expected** | Falls through to MongoDB `IncrementOrgFreeTokensUsed`. |

---

## 8. Model Validation

### TC-8.1: All models enabled

| | |
|---|---|
| **Precondition** | `allModelsEnabled=1`, not in free credit mode |
| **Steps** | 1. Request with `model: "gpt-4o"` |
| **Expected** | Model allowed. No further checks. |

### TC-8.2: Free credit mode — free model allowed

| | |
|---|---|
| **Precondition** | `isFreeCreditStudio=1`. Model: `"llama-3:free"` |
| **Steps** | 1. Request with free model |
| **Expected** | Model allowed (`strings.HasSuffix(model, ":free")` = true). `allModelsEnabled` forced to 0 in free mode. |

### TC-8.3: Free credit mode — paid model rejected

| | |
|---|---|
| **Precondition** | `isFreeCreditStudio=1`. Model: `"gpt-4o"` |
| **Steps** | 1. Request with paid model |
| **Expected** | 403 Forbidden. `code: "model_not_allowed"`. "Only free models are available on the free plan." |

### TC-8.4: Paid models disabled

| | |
|---|---|
| **Precondition** | `paidModelsEnabled=0`, `allModelsEnabled=0`. Model: `"gpt-4o"` |
| **Steps** | 1. Request with paid model |
| **Expected** | 403 Forbidden. "Paid models are not enabled for your organization." |

### TC-8.5: No allowed models configured

| | |
|---|---|
| **Precondition** | `paidModelsEnabled=1`, `allowedModels=""` |
| **Steps** | 1. Request with any model |
| **Expected** | 403 Forbidden. "No models have been configured for your organization." |

### TC-8.6: Model in allowlist

| | |
|---|---|
| **Precondition** | `allowedModels="gpt-4o,claude-3-opus"` |
| **Steps** | 1. Request with `model: "gpt-4o"` |
| **Expected** | Model allowed. Exact match found. |

### TC-8.7: Model matches allowlist prefix

| | |
|---|---|
| **Precondition** | `allowedModels="gpt-4"` |
| **Steps** | 1. Request with `model: "gpt-4o-2024-08-06"` |
| **Expected** | Model allowed. `strings.HasPrefix("gpt-4o-2024-08-06", "gpt-4")` is true. |

### TC-8.8: Model not in allowlist

| | |
|---|---|
| **Precondition** | `allowedModels="gpt-4o,claude-3-opus"` |
| **Steps** | 1. Request with `model: "gemini-pro"` |
| **Expected** | 403 Forbidden. "The selected model is not available for your organization." AllowedModels returned in details. |

### TC-8.9: Empty model in request — validation skipped

| | |
|---|---|
| **Precondition** | Request body has no `model` field |
| **Steps** | 1. Send request |
| **Expected** | `validateModel` returns nil immediately. No model check. |

---

## 9. Webhook Handlers

### TC-9.1: Org updated — subscription changed

| | |
|---|---|
| **Precondition** | Org exists in Dragonfly. Admin changes subscription in NitroCloud. |
| **Steps** | 1. `POST /internal/events/org-updated` with `{"orgId": "ABC", "event": "subscription_updated"}` |
| **Expected** | Config fields updated (creditsTotal, overageEnabled, etc.). Counter fields preserved. Log: "Org ABC config updated". Response: `{"status": "ok"}`. |

### TC-9.2: Org updated — org not yet in Dragonfly

| | |
|---|---|
| **Precondition** | New org not yet cached |
| **Steps** | 1. Send org-updated webhook |
| **Expected** | `UpdateSingleOrgConfig` detects nil existing state → calls `SeedSingleOrg` → full seed including counters. |

### TC-9.3: Org updated — version incremented

| | |
|---|---|
| **Precondition** | Org has `version=3` in Dragonfly |
| **Steps** | 1. Send org-updated webhook |
| **Expected** | Config fields include `version=4`. Monotonic version bump for optimistic concurrency. |

### TC-9.4: Member updated — member added

| | |
|---|---|
| **Precondition** | New member added to org in NitroCloud |
| **Steps** | 1. `POST /internal/events/member-updated` with `{"orgId": "ABC", "userId": "USER1", "event": "member_added"}` |
| **Expected** | `SeedSingleMember` called. Member state populated. Response: `{"status": "ok"}`. |

### TC-9.5: Member updated — member removed

| | |
|---|---|
| **Precondition** | Member exists in Dragonfly |
| **Steps** | 1. Send webhook with `event: "member_removed"` |
| **Expected** | `DeleteMemberState` called. Key `gway:member:ABC:USER1` deleted. Log: "Member ABC/USER1 removed from Dragonfly". |

### TC-9.6: Member updated — access toggled

| | |
|---|---|
| **Precondition** | Member has `studioAccess=true`. Admin disables access. |
| **Steps** | 1. Send webhook with `event: "member_updated"` |
| **Expected** | Member re-seeded from MongoDB. `studioAccess=0` in Dragonfly. Subsequent requests denied. |

### TC-9.7: Webhook with invalid orgId format

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Send webhook with `{"orgId": "not-a-valid-objectid"}` |
| **Expected** | 400 Bad Request. `{"error": "invalid orgId format"}`. |

### TC-9.8: Webhook with empty payload

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Send webhook with `{}` |
| **Expected** | 400 Bad Request. `{"error": "invalid payload"}`. |

### TC-9.9: Trigger manual reconciliation

| | |
|---|---|
| **Precondition** | Reconciler available and idle |
| **Steps** | 1. `POST /internal/reconcile` |
| **Expected** | 200 OK. `{"status": "reconciliation triggered"}`. Reconciliation runs in background goroutine. |

### TC-9.10: Trigger reconciliation while already running

| | |
|---|---|
| **Precondition** | Reconciliation in progress |
| **Steps** | 1. `POST /internal/reconcile` |
| **Expected** | 409 Conflict. `{"error": "reconciliation already in progress"}`. |

---

## 10. Period Reset

### TC-10.1: Period reset — counters zeroed atomically

| | |
|---|---|
| **Precondition** | `creditsUsed=3000`, `freeTokensUsed=50000`, `overageUsed=500`, `subCreditsUsed=3000` |
| **Steps** | 1. `POST /internal/events/period-reset` with `{"orgId": "ABC"}` |
| **Expected** | Single Lua script execution zeros all four counters. Config fields refreshed from MongoDB in same transaction. |

### TC-10.2: Period reset — config updated during reset

| | |
|---|---|
| **Precondition** | Admin changed `creditsTotal` during the billing cycle |
| **Steps** | 1. Send period-reset webhook |
| **Expected** | `BuildOrgConfigFields` reads fresh MongoDB state. Counters zeroed AND new `creditsTotal` applied atomically. |

### TC-10.3: Period reset — member counters reset

| | |
|---|---|
| **Precondition** | 5 members with non-zero `creditsUsed` |
| **Steps** | 1. Send period-reset webhook |
| **Expected** | `ResetMemberCountersForOrg` scans all `gway:member:ABC:*` keys. All 5 members' `creditsUsed` set to 0. Response: `{"status": "ok", "membersReset": 5}`. |

### TC-10.4: Period reset — member reset partial failure

| | |
|---|---|
| **Precondition** | Dragonfly intermittent issue during member reset loop |
| **Steps** | 1. Send period-reset webhook |
| **Expected** | Org counters already zeroed (atomic). Member reset logs warning but does not fail the response. Partial member resets recorded. |

### TC-10.5: Period reset — invalid orgId

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Send period-reset with invalid orgId |
| **Expected** | 400 Bad Request. |

---

## 11. Reconciliation

### TC-11.1: Reconciler starts on schedule

| | |
|---|---|
| **Precondition** | `DRAGONFLY_RECONCILE_INTERVAL=5m` |
| **Steps** | 1. Start the gateway <br> 2. Wait 5 minutes |
| **Expected** | Log: "Starting reconciliation pass...". Runs every 5m. |

### TC-11.2: Missing org seeded during reconciliation

| | |
|---|---|
| **Precondition** | Org exists in MongoDB but NOT in Dragonfly |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | `SeedSingleOrg` called for missing org. Key created in Dragonfly. |

### TC-11.3: Missing member seeded during reconciliation

| | |
|---|---|
| **Precondition** | Active membership in MongoDB, no Dragonfly key |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | `SeedSingleMember` called. Member key created. |

### TC-11.4: Config drift corrected

| | |
|---|---|
| **Precondition** | MongoDB `creditsTotal=10000`. Dragonfly `creditsTotal=5000` (stale). |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | `SetOrgConfigFields` updates `creditsTotal` to 10000. Version incremented. |

### TC-11.5: Counter drift — Dragonfly behind MongoDB (corrected upward)

| | |
|---|---|
| **Precondition** | MongoDB `studioCreditsUsed=3000`. Dragonfly `creditsUsed=2500` (missed increments). |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | `SetOrgCounterField("creditsUsed", 3000)` called. Log: "Fixed org ABC creditsUsed drift: DF=2500, Mongo=3000 (corrected +500)". |

### TC-11.6: Counter drift — Dragonfly ahead of MongoDB (no action)

| | |
|---|---|
| **Precondition** | Dragonfly `creditsUsed=3500`. MongoDB `studioCreditsUsed=3000` (async persistence lag). |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | No correction. Dragonfly ahead is normal (async writes to MongoDB haven't caught up). |

### TC-11.7: Free token counter drift corrected

| | |
|---|---|
| **Precondition** | MongoDB `freeTokensUsed=80000`. Dragonfly `freeTokensUsed=75000`. |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | `SetOrgCounterField("freeTokensUsed", 80000)`. Drift corrected upward. |

### TC-11.8: Overage counter drift corrected

| | |
|---|---|
| **Precondition** | MongoDB `additionalUsageUsed=2000`. Dragonfly `overageUsed=1500`. |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | `SetOrgCounterField("overageUsed", 2000)`. |

### TC-11.9: Member counter drift corrected

| | |
|---|---|
| **Precondition** | MongoDB member `studioCreditsUsed=800`. Dragonfly `creditsUsed=700`. |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | `SetMemberCounterField("creditsUsed", 800)`. Log includes drift amount. |

### TC-11.10: Orphaned org key removed

| | |
|---|---|
| **Precondition** | Dragonfly has `gway:org:DELETED_ORG`. Org no longer exists in MongoDB. |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | `DeleteOrgState` called. Key removed. Log: "Removed orphaned org key: DELETED_ORG". |

### TC-11.11: Orphaned member key removed

| | |
|---|---|
| **Precondition** | Dragonfly has `gway:member:ABC:DELETED_USER`. Membership is `isDeleted: true` in MongoDB. |
| **Steps** | 1. Wait for reconciliation |
| **Expected** | Key deleted. Log: "Removed orphaned member key". |

### TC-11.12: Skip tick when reconciliation already in progress

| | |
|---|---|
| **Precondition** | Reconciliation takes >5 minutes |
| **Steps** | 1. Observe next ticker fire |
| **Expected** | `running.CompareAndSwap(false, true)` returns false. Log: "Skipping tick — reconciliation already in progress". |

### TC-11.13: Member counter reconciliation — batch loading

| | |
|---|---|
| **Precondition** | 500+ member keys in Dragonfly |
| **Steps** | 1. Trigger reconciliation |
| **Expected** | Members loaded in batches of 200 (`memberCounterBatchSize=200`). No single query fetches >200 memberships. |

---

## 12. Health Check

### TC-12.1: All systems healthy

| | |
|---|---|
| **Precondition** | MongoDB, Dragonfly, ClickHouse all connected |
| **Steps** | 1. `GET /health` |
| **Expected** | `{"status": "healthy", "mongodb": "connected", "dragonfly": "connected", ...}` |

### TC-12.2: Dragonfly disconnected

| | |
|---|---|
| **Precondition** | Dragonfly down |
| **Steps** | 1. `GET /health` |
| **Expected** | `{"status": "degraded", "dragonfly": "disconnected", "mongodb": "connected"}` |

### TC-12.3: MongoDB disconnected

| | |
|---|---|
| **Precondition** | MongoDB down |
| **Steps** | 1. `GET /health` |
| **Expected** | `{"status": "degraded", "mongodb": "disconnected"}` |

### TC-12.4: Dragonfly not configured

| | |
|---|---|
| **Precondition** | `DRAGONFLY_ENABLED=false` (df is nil) |
| **Steps** | 1. `GET /health` |
| **Expected** | `dragonfly` field omitted from response. Status can still be "healthy". |

---

## 13. Field Protection & Allowlists

### TC-13.1: SetOrgConfigFields rejects counter fields

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Call `SetOrgConfigFields(ctx, orgID, {"creditsUsed": "999"})` |
| **Expected** | Error: `field "creditsUsed" is not an allowed config field`. Dragonfly NOT written. |

### TC-13.2: SetOrgConfigFields accepts valid config fields

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Call `SetOrgConfigFields(ctx, orgID, {"creditsTotal": "10000", "plan": "PRO"})` |
| **Expected** | Fields written successfully. Only `creditsTotal` and `plan` updated. |

### TC-13.3: SetOrgConfigFields with empty map

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Call `SetOrgConfigFields(ctx, orgID, {})` |
| **Expected** | Returns nil immediately. No Dragonfly command sent. |

### TC-13.4: SetMemberConfigFields rejects creditsUsed

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Call `SetMemberConfigFields(ctx, orgID, userID, {"creditsUsed": "500"})` |
| **Expected** | Error: `field "creditsUsed" is not an allowed config field`. |

### TC-13.5: SetOrgCounterField rejects config fields

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Call `SetOrgCounterField(ctx, orgID, "creditsTotal", 5000)` |
| **Expected** | Error: `"creditsTotal" is not a counter field`. |

### TC-13.6: SetOrgCounterField accepts counter fields

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Call `SetOrgCounterField(ctx, orgID, "creditsUsed", 3000)` |
| **Expected** | Field written successfully. |

### TC-13.7: SetMemberCounterField rejects non-counter fields

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Call `SetMemberCounterField(ctx, orgID, userID, "studioAccess", 1)` |
| **Expected** | Error: `"studioAccess" is not a counter field`. |

---

## 14. Degraded Mode & Failure Scenarios

### TC-14.1: Dragonfly + MongoDB — full functionality

| | |
|---|---|
| **Precondition** | Both services healthy |
| **Steps** | 1. Send requests |
| **Expected** | Sub-ms credit checks. Async MongoDB writes. Analytics flowing. |

### TC-14.2: Dragonfly down — MongoDB only

| | |
|---|---|
| **Precondition** | Dragonfly unreachable |
| **Steps** | 1. Send requests |
| **Expected** | Higher latency (~25-45ms). All original MongoDB behavior preserved. No data loss. |

### TC-14.3: MongoDB down — Dragonfly only

| | |
|---|---|
| **Precondition** | MongoDB unreachable. Dragonfly has cached state. |
| **Steps** | 1. Send requests |
| **Expected** | Credit enforcement works from Dragonfly cache. UsageWorker retries MongoDB writes with backoff. No analytics persisted until MongoDB recovers. |

### TC-14.4: Both down — fail closed

| | |
|---|---|
| **Precondition** | Both services unreachable |
| **Steps** | 1. Send requests |
| **Expected** | 503 Service Unavailable. No requests allowed through. Fail-closed security. |

### TC-14.5: Dragonfly reconnects after outage

| | |
|---|---|
| **Precondition** | Dragonfly was down, now back up |
| **Steps** | 1. Send requests |
| **Expected** | Rueidis auto-reconnects. Next request uses Dragonfly fast path. Reconciler fills any gaps. |

### TC-14.6: Gateway restart — state continuity

| | |
|---|---|
| **Precondition** | Dragonfly has existing state from before restart |
| **Steps** | 1. Restart gateway |
| **Expected** | Seeder runs. Config fields updated. Counter fields preserved. No double-counting. |

---

## 15. Concurrency & Atomicity

### TC-15.1: Concurrent credit increments — no race condition

| | |
|---|---|
| **Precondition** | `creditsUsed=0`. 100 concurrent requests, each costing 10 cents. |
| **Steps** | 1. Fire 100 parallel chat completions |
| **Expected** | Final `creditsUsed=1000` (exactly 100 * 10). Lua script HINCRBY is atomic. No lost updates. |

### TC-15.2: Concurrent rate limit checks

| | |
|---|---|
| **Precondition** | `maxRPM=60`. 100 concurrent requests in same minute window. |
| **Steps** | 1. Fire 100 parallel requests |
| **Expected** | Exactly 60 allowed, 40 rejected with 429. Lua INCR is atomic. |

### TC-15.3: Period reset during active billing

| | |
|---|---|
| **Precondition** | Billing requests actively incrementing counters |
| **Steps** | 1. Send period-reset webhook while requests are in-flight |
| **Expected** | `periodResetScript` Lua atomically zeros counters and applies config. No partially-reset state observable. Billing requests after the reset start from zero. |

### TC-15.4: Webhook update during request processing

| | |
|---|---|
| **Precondition** | Request in StudioAccessMiddleware. Webhook changes `creditsTotal`. |
| **Steps** | 1. Observe behavior |
| **Expected** | The request sees the state at the time of HGETALL. Next request sees updated `creditsTotal`. No inconsistent partial reads (HGETALL is atomic). |

### TC-15.5: Reconciliation concurrent with billing

| | |
|---|---|
| **Precondition** | Reconciler correcting counter drift while billing increments happening |
| **Steps** | 1. Observe Dragonfly counters |
| **Expected** | `SetOrgCounterField` may be slightly stale by the time it executes, but since reconciler only corrects upward (MongoDB > Dragonfly), worst case is a minor under-count that resolves on next cycle. |

### TC-15.6: Duplicate reconciliation prevention

| | |
|---|---|
| **Precondition** | Ticker fires while previous reconciliation is running |
| **Steps** | 1. Observe reconciler |
| **Expected** | `running.CompareAndSwap(false, true)` returns false. Skip message logged. No overlapping reconciliation passes. |

---

## 16. Performance & Latency

### TC-16.1: Dragonfly read latency < 1ms

| | |
|---|---|
| **Precondition** | Dragonfly running locally or low-latency network |
| **Steps** | 1. Measure `GetOrgState` + `GetMemberState` round-trip |
| **Expected** | Total < 1ms (typically 0.3ms per HGETALL). Compare with MongoDB baseline of 25-45ms. |

### TC-16.2: Lua script latency < 0.5ms

| | |
|---|---|
| **Precondition** | Dragonfly under normal load |
| **Steps** | 1. Measure `CheckAndIncrementDollarCredits` execution time |
| **Expected** | < 0.5ms end-to-end. Single round-trip with atomic operations. |

### TC-16.3: Startup seed time

| | |
|---|---|
| **Precondition** | 1000 orgs, 5000 members in MongoDB |
| **Steps** | 1. Start gateway, observe seed duration |
| **Expected** | Seed completes in < 10s. Log shows duration. |

### TC-16.4: Billing middleware timeout

| | |
|---|---|
| **Precondition** | Dragonfly slow to respond |
| **Steps** | 1. Observe billing middleware behavior |
| **Expected** | 2-second timeout (`context.WithTimeout(ctx, 2*time.Second)`). Falls back to MongoDB after timeout. |

---

## 17. Security

### TC-17.1: Webhook authentication — valid secret

| | |
|---|---|
| **Precondition** | `INTERNAL_WEBHOOK_SECRET=mysecret` |
| **Steps** | 1. `POST /internal/events/org-updated` with `Authorization: Bearer mysecret` |
| **Expected** | Request authenticated. Handler proceeds. |

### TC-17.2: Webhook authentication — invalid secret

| | |
|---|---|
| **Precondition** | `INTERNAL_WEBHOOK_SECRET=mysecret` |
| **Steps** | 1. Send webhook with `Authorization: Bearer wrongsecret` |
| **Expected** | 401 Unauthorized. `{"error": "invalid webhook secret"}`. Constant-time comparison prevents timing attacks. |

### TC-17.3: Webhook authentication — missing header

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Send webhook without Authorization header |
| **Expected** | 401 Unauthorized. `{"error": "missing or malformed Authorization header"}`. |

### TC-17.4: Webhook authentication — secret not configured

| | |
|---|---|
| **Precondition** | `INTERNAL_WEBHOOK_SECRET=""` (not set) |
| **Steps** | 1. Send any webhook request |
| **Expected** | 401 Unauthorized. `{"error": "internal webhook secret not configured"}`. Fail-closed. |

### TC-17.5: Webhook authentication — malformed Bearer token

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Send webhook with `Authorization: Basic abc123` |
| **Expected** | 401 Unauthorized. Prefix check rejects non-Bearer tokens. |

### TC-17.6: ObjectID validation on webhook payloads

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Send webhook with `orgId: "'; DROP TABLE"` |
| **Expected** | 400 Bad Request. `isValidObjectID` rejects invalid hex strings. No MongoDB injection possible. |

### TC-17.7: Counter fields cannot be overwritten via config API

| | |
|---|---|
| **Precondition** | N/A |
| **Steps** | 1. Attempt to manipulate counter fields through any config update path (webhook, reconciler, seed) |
| **Expected** | `orgConfigFields` and `memberConfigFields` allowlists prevent counter overwrites. Only Lua scripts and explicit `SetCounterField` can modify counters. |

---

## Summary Matrix

| Area | Total Cases | Critical | High | Medium |
|------|:-----------:|:--------:|:----:|:------:|
| Connection & Init | 7 | 2 | 3 | 2 |
| Startup Seeding | 9 | 2 | 4 | 3 |
| Studio Access (Dragonfly) | 18 | 5 | 8 | 5 |
| Studio Access (MongoDB FB) | 4 | 1 | 2 | 1 |
| Rate Limiting | 9 | 3 | 4 | 2 |
| Billing — Dollar | 11 | 4 | 5 | 2 |
| Billing — Token | 5 | 2 | 2 | 1 |
| Model Validation | 9 | 2 | 4 | 3 |
| Webhooks | 10 | 3 | 5 | 2 |
| Period Reset | 5 | 3 | 1 | 1 |
| Reconciliation | 13 | 3 | 6 | 4 |
| Health Check | 4 | 1 | 2 | 1 |
| Field Protection | 7 | 3 | 3 | 1 |
| Degraded Mode | 6 | 3 | 2 | 1 |
| Concurrency | 6 | 4 | 2 | 0 |
| Performance | 4 | 2 | 2 | 0 |
| Security | 7 | 4 | 2 | 1 |
| **TOTAL** | **134** | **47** | **57** | **30** |
