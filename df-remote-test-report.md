# NitroStack Gateway — Test Report

---

## Summary

| Package | Tests | Passed | Failed | Duration |
|---------|-------|--------|--------|----------|
| `internal/store` | 32 | 32 | 0 | 0.685s |
| `internal/models` | 1 | 1 | 0 | 0.493s |
| `internal/repository` | 3 (30 sub) | 3 | 0 | 0.708s |
| **Total** | **63** | **63** | **0** | **1.886s** |

**Result: ✅ ALL TESTS PASSED**

---

## Detailed Results

### `internal/store` — Dragonfly Store (32 tests)

#### types_test.go — OrgState & MemberState Helpers

| # | Test | Sub-tests | Result |
|---|------|-----------|--------|
| 1 | `TestOrgState_IsFreeCreditMode` | enabled, disabled, invalid_value | ✅ PASS |
| 2 | `TestOrgState_IsTokenMode` | free+tokens, free+dollar, not_free+tokens, not_free+dollar, free+empty | ✅ PASS |
| 3 | `TestOrgState_AllowedModelsList` | empty_string, single_model, multiple_models | ✅ PASS |
| 4 | `TestOrgState_IsModelAllowed` | all_enabled, exact_match, prefix_match, no_match, empty_list, prefix_second | ✅ PASS |
| 5 | `TestMemberState_HasAccess` | granted, denied | ✅ PASS |
| 6 | `TestMemberState_PersonalLimit` | no_limit, ignores_additional, zero, with_additional, without_additional | ✅ PASS |

#### dragonfly_helpers_test.go — Parsing, Serialization & Key Utils

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

#### seed_helpers_test.go — BSON float64→int64 Decode Verification

| # | Test | Validates | Result |
|---|------|-----------|--------|
| 19 | `TestSeedMembership_Float64Decode` | float64 `studioCreditsUsed` decodes without error | ✅ PASS |
| 20 | `TestSeedMembership_Float64Decode_FractionalValue` | fractional float64 truncates correctly to int64 | ✅ PASS |
| 21 | `TestSeedMembership_IntegerDecode` | int64 `studioCreditsUsed` still works with float64 field | ✅ PASS |
| 22 | `TestSeedOrg_Float64Decode` | seedOrg handles float64 `studioCreditsUsed` | ✅ PASS |
| 23 | `TestSeedSubscription_Float64Decode` | seedSubscription handles float64 `studioCreditsUsed` | ✅ PASS |
| 24 | `TestSeedOrg_ZeroValue` | zero value float64 decodes correctly | ✅ PASS |

---

### `internal/models` — Organization Models (1 test)

| # | Test | Result |
|---|------|--------|
| 1 | `TestConvertLegacyMember` | ✅ PASS |

---

### `internal/repository` — MongoDB Repository (3 tests, 30 sub-tests)

#### membership_test.go

| # | Test | Sub-tests | Result |
|---|------|-----------|--------|
| 1 | `TestGetOrganizationMembership` | found, legacy_user_field, credit_limit, no_studioAccess, legacy_fallback, correct_legacy, not_found, empty_legacy, nil_legacy, db_error, invalid_orgID, invalid_userID | ✅ PASS |
| 2 | `TestMembershipRawToMembership` | canonical, legacy_normalize, both_fields, all_studio_fields | ✅ PASS |
| 3 | `TestIncrementMemberCreditsUsed` | found, legacy_fallback, legacy_field, not_found, db_errors (×3), invalid_orgID, invalid_userID | ✅ PASS |

#### migration_test.go

| # | Test | Sub-tests | Result |
|---|------|-----------|--------|
| 4 | `TestMigrateEmbeddedMembers` | all_created, idempotent, mixed_duplicates, empty, normalized, query_failure, insert_error, multiple_orgs, mixed_write_errors | ✅ PASS |

---

## Test Coverage Notes

| Area | Covered | Not Covered (requires external deps) |
|------|---------|--------------------------------------|
| OrgState/MemberState helpers | ✅ | — |
| State serialization round-trips | ✅ | — |
| Key builders & parsers | ✅ | — |
| BSON float64→int64 decode fix | ✅ | — |
| Utility functions | ✅ | — |
| DragonflyStore CRUD | — | Requires Redis/Dragonfly |
| Lua credit scripts | — | Requires Redis/Dragonfly |
| Seed from MongoDB | — | Requires MongoDB |
| Reconciler loop | — | Requires MongoDB + Dragonfly |
