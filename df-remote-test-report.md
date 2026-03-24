# NitroStack Gateway — Test Report

---

## Summary

| Package | Tests | Passed | Failed | Duration | Type |
|---------|-------|--------|--------|----------|------|
| `internal/store` (unit) | 32 | 32 | 0 | 0.685s | Unit |
| `internal/store` (integration) | 12 | 12 | 0 | 252.1s | Integration |
| `internal/models` | 1 | 1 | 0 | 0.493s | Unit |
| `internal/repository` | 3 (30 sub) | 3 | 0 | 0.708s | Unit |
| **Total** | **48 (75 with sub-tests)** | **48** | **0** | — |

**Result: ✅ ALL TESTS PASSED**

---

## Unit Tests — `internal/store` (32 tests, 0.685s)

### types_test.go — OrgState & MemberState Helpers

| # | Test | Sub-tests | Result |
|---|------|-----------|--------|
| 1 | `TestOrgState_IsFreeCreditMode` | enabled, disabled, invalid_value | ✅ PASS |
| 2 | `TestOrgState_IsTokenMode` | free+tokens, free+dollar, not_free+tokens, not_free+dollar, free+empty | ✅ PASS |
| 3 | `TestOrgState_AllowedModelsList` | empty_string, single_model, multiple_models | ✅ PASS |
| 4 | `TestOrgState_IsModelAllowed` | all_enabled, exact_match, prefix_match, no_match, empty_list, prefix_second | ✅ PASS |
| 5 | `TestMemberState_HasAccess` | granted, denied | ✅ PASS |
| 6 | `TestMemberState_PersonalLimit` | no_limit, ignores_additional, zero, with_additional, without_additional | ✅ PASS |

### dragonfly_helpers_test.go — Parsing, Serialization & Key Utils

| # | Test | Sub-tests | Result |
|---|------|-----------|--------|
| 7 | `TestParseOrgState_RoundTrip` | — | ✅ PASS |
| 8 | `TestParseOrgState_EmptyMap` | — | ✅ PASS |
| 9 | `TestParseMemberState_RoundTrip` | — | ✅ PASS |
| 10 | `TestParseMemberState_NegativeCreditLimit` | — | ✅ PASS |
| 11 | `TestParseMemberState_EmptyMap` | — | ✅ PASS |
| 12 | `TestOrgKey` | — | ✅ PASS |
| 13 | `TestMemberKey` | — | ✅ PASS |
| 14 | `TestParseMemberKey` | valid_key, hex_IDs, too_short, no_user_part, trailing_colon, empty, prefix_only | ✅ PASS |
| 15 | `TestLastColon` | multiple, single, none, at_end, at_start, empty | ✅ PASS |
| 16 | `TestBoolToInt` | — | ✅ PASS |
| 17 | `TestParseInt64` | positive, negative, zero, empty, whitespace, invalid | ✅ PASS |
| 18 | `TestParseInt` | valid, zero, empty, whitespace, invalid | ✅ PASS |

### seed_helpers_test.go — BSON float64→int64 Decode Verification

| # | Test | Validates | Result |
|---|------|-----------|--------|
| 19 | `TestSeedMembership_Float64Decode` | float64 `studioCreditsUsed` decodes without error | ✅ PASS |
| 20 | `TestSeedMembership_Float64Decode_FractionalValue` | fractional float64 truncates correctly to int64 | ✅ PASS |
| 21 | `TestSeedMembership_IntegerDecode` | int64 `studioCreditsUsed` still works with float64 field | ✅ PASS |
| 22 | `TestSeedOrg_Float64Decode` | seedOrg handles float64 `studioCreditsUsed` | ✅ PASS |
| 23 | `TestSeedSubscription_Float64Decode` | seedSubscription handles float64 `studioCreditsUsed` | ✅ PASS |
| 24 | `TestSeedOrg_ZeroValue` | zero value float64 decodes correctly | ✅ PASS |

---

## Integration Tests — `internal/store` (12 tests, 252.1s)

> Requires: Dragonfly + MongoDB (via `.env`)  
> Run with: `go test ./internal/store/ -tags integration -v -timeout 300s`

### Connectivity

| # | Test | Duration | Result |
|---|------|----------|--------|
| 1 | `TestIntegration_DragonflyPing` | 6.56s | ✅ PASS |

### DragonflyStore CRUD

| # | Test | Duration | Result |
|---|------|----------|--------|
| 2 | `TestIntegration_OrgState_CRUD` | 8.62s | ✅ PASS |
| 3 | `TestIntegration_MemberState_CRUD` | 8.61s | ✅ PASS |

### Lua Scripts — Dollar Credits

| # | Test | Scenario | Duration | Result |
|---|------|----------|----------|--------|
| 4 | `TestIntegration_DollarCredits_Allowed` | Within plan credits | 8.77s | ✅ PASS |
| 5 | `TestIntegration_DollarCredits_OrgExhausted` | No credits, no overage | 7.14s | ✅ PASS |
| 6 | `TestIntegration_DollarCredits_Overage` | Credits gone, overage enabled | 7.21s | ✅ PASS |
| 7 | `TestIntegration_DollarCredits_OverageLimit` | Overage hard limit hit | 8.08s | ✅ PASS |
| 8 | `TestIntegration_DollarCredits_MemberExhausted` | Member personal limit hit | 8.12s | ✅ PASS |

### Lua Scripts — Token Credits

| # | Test | Scenario | Duration | Result |
|---|------|----------|----------|--------|
| 9 | `TestIntegration_TokenCredits` | Increment within limit | 8.09s | ✅ PASS |
| 10 | `TestIntegration_TokenCredits_Exhausted` | Tokens exhausted | 8.04s | ✅ PASS |
| 11 | `TestIntegration_TokenCredits_NoLimit` | No limit configured | 6.71s | ✅ PASS |

### Rate Limiting

| # | Test | Duration | Result |
|---|------|----------|--------|
| 12 | `TestIntegration_RateLimit` | 7.77s | ✅ PASS |

### Seed from MongoDB

| # | Test | Result | Details |
|---|------|--------|---------|
| 13 | `TestIntegration_SeedFromMongo` | ✅ PASS (44.43s) | 65 orgs, 76 members seeded from MongoDB |

### Reconciler

| # | Test | Result | Details |
|---|------|--------|---------|
| 14 | `TestIntegration_Reconciler_RunOnce` | ✅ PASS (113.28s) | Full reconcile pass: 0 updates needed, 0 orphans |

---

## Unit Tests — Other Packages

### `internal/models` (1 test)

| # | Test | Result |
|---|------|--------|
| 1 | `TestConvertLegacyMember` | ✅ PASS |

### `internal/repository` (3 tests, 30 sub-tests)

| # | Test | Sub-tests | Result |
|---|------|-----------|--------|
| 1 | `TestGetOrganizationMembership` | 12 sub-tests | ✅ PASS |
| 2 | `TestMembershipRawToMembership` | 4 sub-tests | ✅ PASS |
| 3 | `TestIncrementMemberCreditsUsed` | 9 sub-tests | ✅ PASS |
| 4 | `TestMigrateEmbeddedMembers` | 9 sub-tests | ✅ PASS |

---

## How to Run

```bash
# Unit tests only (fast, no external deps)
cd gateway && go test ./internal/... -v -count=1

# Integration tests (requires Dragonfly + MongoDB via .env)
cd gateway && go test ./internal/store/ -tags integration -v -timeout 300s

# All tests
cd gateway && go test ./internal/... -tags integration -v -timeout 300s
```

---

## Test File Summary

| File | Type | Tests | Source |
|------|------|-------|--------|
| `internal/store/types_test.go` | Unit | 8 | OrgState/MemberState helpers |
| `internal/store/dragonfly_helpers_test.go` | Unit | 18 | Parsing, serialization, keys |
| `internal/store/seed_helpers_test.go` | Unit | 6 | BSON float64 decode fix |
| `internal/store/store_integration_test.go` | Integration | 12 | CRUD, Lua scripts, Seed, Reconciler |
| `internal/models/organization_test.go` | Unit | 1 | Legacy member conversion |
| `internal/repository/membership_test.go` | Unit | 3 (25 sub) | Membership CRUD |
| `internal/repository/migration_test.go` | Unit | 1 (9 sub) | Member migration |
