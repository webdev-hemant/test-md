# PR #49 — Testing Guide

## Migrate Membership Lookups to Dedicated Collection & Harden Access Middleware

**PR**: [#49](https://github.com/nitrocloudofficial/nitrostudio/pull/49)
**Branch**: `feature/fix-org-members-inconsistency` → `feature/free-credit-mode`
**Author**: Hemant Jadhav

---

## Summary

This PR replaces the legacy embedded `members[]` array on the `organizations` document with lookups against a dedicated `organization_memberships` collection. Key changes:

- New `OrganizationMembership` model mapped to `organization_memberships` collection
- New `GetOrganizationMembership()` repo method with dual-schema support (canonical `userId` + legacy `user`)
- `IncrementMemberCreditsUsed` rewritten to target membership documents first, legacy array second
- `GetStudioUsageStatus` accepts optional `prefetchedMember` to avoid redundant DB calls
- Fail-closed access control (503 instead of silently allowing when repo is nil)
- Migration endpoint `POST /internal/migrate/members` for one-time data migration
- Degraded-mode guards in `LoggingMiddleware`, `UsageWorker`, and `StudioAccessMiddleware`

---

## ER Diagram — Data Model & Testing Flow

```mermaid
erDiagram
    ORGANIZATIONS {
        ObjectId _id PK
        string name
        string slug
        ObjectId owner FK
        array members "Legacy embedded array"
        string plan
        ObjectId subscription FK
        object studioConfig
        int64 studioCredits
        int64 studioCreditsUsed
    }

    ORGANIZATION_MEMBERSHIPS {
        ObjectId _id PK
        ObjectId userId FK "Canonical field"
        ObjectId user FK "Legacy-dump field (normalized away)"
        ObjectId organizationId FK
        ObjectId roleId FK
        time joinedAt
        bool studioAccess
        int64_ptr studioCreditLimit "nil = org default"
        int64 studioCreditsUsed
        int64 studioAdditionalUsageLimit
        bool isDeleted "Soft delete"
        time deletedAt
        time createdAt
        time updatedAt
    }

    SUBSCRIPTIONS {
        ObjectId _id PK
        ObjectId organization FK
        string planType
        string status
        int64 studioCredits
        int64 studioCreditsUsed
        bool studioUsageBillingEnabled
        int64 studioUsageLimit
        int64 additionalUsageUsed
    }

    USERS {
        ObjectId _id PK
        string email
        string firstName
        string lastName
    }

    STUDIO_USAGE_LOGS {
        ObjectId _id PK
        string userId
        string provider
        string model
        int promptTokens
        int completionTokens
        float64 cost
    }

    STUDIO_ORG_CREDITS {
        ObjectId _id PK
        ObjectId organization FK
        int64 creditsTotal
        float64 creditsUsed
        bool isDeleted
    }

    ORGANIZATIONS ||--o{ ORGANIZATION_MEMBERSHIPS : "migrated to"
    ORGANIZATIONS ||--o| SUBSCRIPTIONS : "has"
    ORGANIZATIONS ||--o| STUDIO_ORG_CREDITS : "tracks credits"
    ORGANIZATION_MEMBERSHIPS }o--|| USERS : "belongs to"
    STUDIO_USAGE_LOGS }o--|| USERS : "logged for"
    STUDIO_USAGE_LOGS }o--|| ORGANIZATIONS : "charged to"
```

---

## Request Flow Diagram

```mermaid
flowchart TD
    A[Client Request] --> B{AuthMiddleware}
    B -->|Authenticated| C{StudioAccessMiddleware}
    B -->|Unauthenticated| Z[401 Unauthorized]

    C -->|repo == nil| D[503 STUDIO_SERVICE_UNAVAILABLE]
    C -->|repo available| E{Is Owner?}

    E -->|Yes| H[Skip member checks]
    E -->|No| F[GetOrganizationMembership]

    F --> F1{Found in org_memberships?}
    F1 -->|Yes canonical userId| G1[Return membership]
    F1 -->|Yes legacy user field| G2[Return + fire-and-forget normalize]
    F1 -->|Not found| F2{Search legacy org.Members array}
    F2 -->|Found| G3[ConvertLegacyMember → return]
    F2 -->|Not found| G4[403 STUDIO_ACCESS_DENIED]

    G1 --> I{studioAccess == true?}
    G2 --> I
    G3 --> I
    I -->|No| J[403 STUDIO_ACCESS_DENIED]
    I -->|Yes| K{Credit checks}

    H --> K
    K -->|Exhausted| L[402 CREDITS_EXHAUSTED / HARD_LIMIT_REACHED]
    K -->|OK| M[Proceed to handler]

    M --> N{Credit increment needed?}
    N -->|Yes, not owner| O[IncrementMemberCreditsUsed]
    O --> O1{Found in org_memberships?}
    O1 -->|Yes| O2[Update membership doc]
    O1 -->|No match| O3[Fallback: update legacy org.Members array]
    O3 -->|No match| O4[Error: no active membership found]
```

---

## Migration Flow Diagram

```mermaid
flowchart TD
    A[POST /internal/migrate/members] --> B{repo == nil?}
    B -->|Yes| C[503 Service Unavailable]
    B -->|No| D[Create unique index on org_memberships]

    D --> E[Normalize legacy docs: rename user → userId]
    E --> F[Find all orgs with non-empty members array]

    F --> G[For each org]
    G --> H[Build membership docs from embedded members]
    H --> I[InsertMany ordered=false]

    I --> J{Duplicate key?}
    J -->|Yes| K[Count as skipped]
    J -->|No| L[Count as created]
    J -->|Other error| M[Count as error + log]

    K --> N[Next org]
    L --> N
    M --> N
    N -->|More orgs| G
    N -->|Done| O[Return MigrationResult]
```

---

## Prerequisites

1. **MongoDB** running locally or accessible via connection string
2. **Admin JWT token** (the migration endpoint requires `AuthenticateAdmin`)
3. **Test user accounts**: at least one org owner + one non-owner member
4. **Gateway service** running locally (`go run ./cmd/server`)

---

## Step-by-Step Testing Guide

### Phase 1: Run Unit Tests

```bash
cd gateway

# Run all new tests
go test ./internal/models/ -v -run TestConvertLegacyMember
go test ./internal/repository/ -v -run TestGetOrganizationMembership
go test ./internal/repository/ -v -run TestMembershipRawToMembership
go test ./internal/repository/ -v -run TestIncrementMemberCreditsUsed

# Run all tests together
go test ./internal/... -v
```

**Expected**: All tests pass. Verify output covers:
- Canonical schema lookup
- Legacy `user` field normalization
- Fallback to embedded `org.Members`
- Error propagation for DB failures
- Invalid ObjectID rejection

---

### Phase 2: Database Preparation

#### Step 1 — Verify existing data

```bash
# Connect to MongoDB
mongosh

# Switch to your database
use nitrocloud

# Check if organizations have embedded members
db.organizations.find({ "members.0": { $exists: true } }).count()

# Sample an org with members
db.organizations.findOne({ "members.0": { $exists: true } }, { name: 1, members: 1 })
```

#### Step 2 — Check the new collection (should be empty or non-existent pre-migration)

```bash
db.organization_memberships.countDocuments()
# Expected: 0 (or collection doesn't exist yet)
```

---

### Phase 3: Test Migration Endpoint

#### Step 3 — Run the migration

```bash
# Replace <ADMIN_TOKEN> with a valid admin JWT
curl -X POST http://localhost:3001/internal/migrate/members \
  -H "Authorization: Bearer <ADMIN_TOKEN>" \
  -H "Content-Type: application/json"
```

**Expected response:**
```json
{
  "success": true,
  "data": {
    "orgsProcessed": 5,
    "membersCreated": 12,
    "membersSkipped": 0,
    "legacyDocsNormalized": 0,
    "errors": 0
  }
}
```

#### Step 4 — Verify migration results

```bash
mongosh
use nitrocloud

# Count migrated memberships
db.organization_memberships.countDocuments()

# Verify a specific membership
db.organization_memberships.findOne({ organizationId: ObjectId("<ORG_ID>") })

# Verify canonical field names
db.organization_memberships.find({ "userId": { $exists: true } }).count()

# Check no legacy "user" field remains
db.organization_memberships.find({ "user": { $exists: true }, "userId": { $exists: false } }).count()
# Expected: 0
```

#### Step 5 — Run migration again (idempotency test)

```bash
curl -X POST http://localhost:3001/internal/migrate/members \
  -H "Authorization: Bearer <ADMIN_TOKEN>" \
  -H "Content-Type: application/json"
```

**Expected**: `membersCreated: 0`, `membersSkipped: <previous count>`, `errors: 0`

---

### Phase 4: Test Access Control (StudioAccessMiddleware)

#### Step 6 — Owner access (should always work)

```bash
# Use an owner's JWT token
curl -X POST http://localhost:3001/v1/studio/chat/completions \
  -H "Authorization: Bearer <OWNER_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o:free", "messages": [{"role": "user", "content": "Hello"}]}'
```

**Expected**: 200 OK (or proxied response from provider)

#### Step 7 — Member with studioAccess: true

```bash
curl -X POST http://localhost:3001/v1/studio/chat/completions \
  -H "Authorization: Bearer <MEMBER_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o:free", "messages": [{"role": "user", "content": "Hello"}]}'
```

**Expected**: 200 OK — membership resolved from `organization_memberships` collection

#### Step 8 — Member with studioAccess: false

```bash
# Set studioAccess to false for a test member
mongosh --eval 'db.organization_memberships.updateOne(
  { userId: ObjectId("<USER_ID>"), organizationId: ObjectId("<ORG_ID>") },
  { $set: { studioAccess: false } }
)'

curl -X POST http://localhost:3001/v1/studio/chat/completions \
  -H "Authorization: Bearer <MEMBER_NO_ACCESS_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o:free", "messages": [{"role": "user", "content": "Hello"}]}'
```

**Expected**: 403 with `STUDIO_ACCESS_DENIED`

#### Step 9 — Non-member (not in org at all)

```bash
curl -X POST http://localhost:3001/v1/studio/chat/completions \
  -H "Authorization: Bearer <OUTSIDER_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o:free", "messages": [{"role": "user", "content": "Hello"}]}'
```

**Expected**: 403 with `STUDIO_ACCESS_DENIED` — "You are not a member of this organization"

---

### Phase 5: Test Credit Operations

#### Step 10 — Verify credit increment updates the membership document

```bash
# Record before state
mongosh --eval 'db.organization_memberships.findOne(
  { userId: ObjectId("<USER_ID>"), organizationId: ObjectId("<ORG_ID>") },
  { studioCreditsUsed: 1 }
)'

# Make a request that triggers credit increment (non-owner)
curl -X POST http://localhost:3001/v1/studio/usage/report \
  -H "Authorization: Bearer <MEMBER_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o", "promptTokens": 100, "completionTokens": 50, "costCents": 5}'

# Record after state
mongosh --eval 'db.organization_memberships.findOne(
  { userId: ObjectId("<USER_ID>"), organizationId: ObjectId("<ORG_ID>") },
  { studioCreditsUsed: 1, updatedAt: 1 }
)'
```

**Expected**: `studioCreditsUsed` incremented by 5, `updatedAt` refreshed

#### Step 11 — Verify credits NOT updated on embedded array

```bash
mongosh --eval 'db.organizations.findOne(
  { _id: ObjectId("<ORG_ID>") },
  { "members.studioCreditsUsed": 1 }
)'
```

**Expected**: Embedded array `studioCreditsUsed` should remain unchanged (not incremented)

---

### Phase 6: Test GetStudioUsageStatus

#### Step 12 — Verify usage status returns correct limits from membership

```bash
curl -X GET http://localhost:3001/v1/studio/usage/status \
  -H "Authorization: Bearer <MEMBER_TOKEN>"
```

**Expected response includes:**
```json
{
  "success": true,
  "data": {
    "hasAccess": true,
    "personalLimit": 5000,
    "personalUsed": 505,
    "creditsRemaining": ...,
    "creditsTotal": ...
  }
}
```

---

### Phase 7: Test Degraded Mode (Nil Repo)

> These tests require modifying the server startup to pass `nil` as the MongoDB repo.
> Alternatively, stop MongoDB and restart the gateway (if it handles nil repo gracefully).

#### Step 13 — StudioAccessMiddleware returns 503 when repo is nil

```bash
curl -X POST http://localhost:3001/v1/studio/chat/completions \
  -H "Authorization: Bearer <ANY_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o:free", "messages": [{"role": "user", "content": "Hello"}]}'
```

**Expected**: 503 with `STUDIO_SERVICE_UNAVAILABLE`

#### Step 14 — LoggingMiddleware skips writes (check server logs)

**Expected**: Log line `[Logging] Skipping usage log — MongoDB repo is nil (degraded mode)`

#### Step 15 — UsageWorker skips aggregation (check server logs)

**Expected**: Log line `[UsageWorker] Skipping aggregate update — MongoDB repo is nil (degraded mode)`

#### Step 16 — MigrateMembers returns 503

```bash
curl -X POST http://localhost:3001/internal/migrate/members \
  -H "Authorization: Bearer <ADMIN_TOKEN>"
```

**Expected**: 503 with `"MongoDB is unavailable"`

---

### Phase 8: Test Legacy Fallback (Pre-Migration Orgs)

#### Step 17 — Remove membership doc, keep legacy embedded array

```bash
# Delete the membership doc for a test user
mongosh --eval 'db.organization_memberships.deleteOne({
  userId: ObjectId("<USER_ID>"),
  organizationId: ObjectId("<ORG_ID>")
})'

# Verify embedded member still exists in org
mongosh --eval 'db.organizations.findOne(
  { _id: ObjectId("<ORG_ID>"), "members.user": ObjectId("<USER_ID>") },
  { "members.$": 1 }
)'
```

#### Step 18 — Access should still work via legacy fallback

```bash
curl -X POST http://localhost:3001/v1/studio/chat/completions \
  -H "Authorization: Bearer <MEMBER_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o:free", "messages": [{"role": "user", "content": "Hello"}]}'
```

**Expected**: 200 OK, server logs show:
```
[GetOrganizationMembership] Using legacy org.Members fallback — userID: ..., orgID: ...
```

#### Step 19 — Credit increment falls back to legacy array

```bash
curl -X POST http://localhost:3001/v1/studio/usage/report \
  -H "Authorization: Bearer <MEMBER_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o", "promptTokens": 50, "completionTokens": 25, "costCents": 3}'
```

**Expected**: Server logs show:
```
[IncrementMemberCreditsUsed] Falling back to org.Members array — userID: ..., orgID: ...
```

---

### Phase 9: Test Legacy-Dump Normalization

#### Step 20 — Insert a legacy-dumped doc with `user` field

```bash
mongosh --eval '
db.organization_memberships.insertOne({
  user: ObjectId("<USER_ID>"),
  organizationId: ObjectId("<ORG_ID>"),
  studioAccess: true,
  studioCreditsUsed: NumberLong(0),
  isDeleted: false,
  createdAt: new Date(),
  updatedAt: new Date()
})'
```

#### Step 21 — Access request triggers fire-and-forget normalization

```bash
curl -X POST http://localhost:3001/v1/studio/chat/completions \
  -H "Authorization: Bearer <MEMBER_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o:free", "messages": [{"role": "user", "content": "Hello"}]}'
```

#### Step 22 — Verify normalization happened

```bash
# Wait 2-3 seconds for the fire-and-forget goroutine
sleep 3

mongosh --eval 'db.organization_memberships.findOne({
  organizationId: ObjectId("<ORG_ID>"),
  userId: ObjectId("<USER_ID>")
})'
```

**Expected**: Document now has `userId` field, `user` field removed

---

## Test Scenarios Matrix

### A. Data Model & Conversion

| # | Scenario | Input | Expected | Component |
|---|----------|-------|----------|-----------|
| A1 | ConvertLegacyMember — basic member | Legacy member with studioAccess: true | Returns OrganizationMembership with matching fields, zero ID/RoleID | `models/organization.go` |
| A2 | ConvertLegacyMember — nil credit limit | Legacy member with nil StudioCreditLimit | Membership has nil StudioCreditLimit (uses org default) | `models/organization.go` |
| A3 | ConvertLegacyMember — at credit limit | Used == Limit | Membership reflects exact credit usage | `models/organization.go` |
| A4 | ConvertLegacyMember — guest role | Guest role member | Membership preserves role metadata | `models/organization.go` |

### B. GetOrganizationMembership

| # | Scenario | Input | Expected | Component |
|---|----------|-------|----------|-----------|
| B1 | Found in memberships (canonical `userId`) | Valid orgID + userID, doc with `userId` | Returns membership, no fallback | `repository/mongo.go` |
| B2 | Found via legacy-dump (`user` field) | Doc with `user` not `userId` | Returns membership + triggers normalize goroutine | `repository/mongo.go` |
| B3 | Legacy-dump preserves credit limit | Doc with `user`, `studioCreditLimit` set | StudioCreditLimit carried through | `repository/mongo.go` |
| B4 | Legacy-dump defaults studioAccess false | Doc with `user`, no studioAccess field | StudioAccess == false | `repository/mongo.go` |
| B5 | Fallback to legacy org.Members | Not in collection, in embedded array | Returns ConvertLegacyMember result | `repository/mongo.go` |
| B6 | Correct member among multiple legacy | Multiple legacy members, match by userID | Returns correct member | `repository/mongo.go` |
| B7 | Not found anywhere | Not in collection, not in legacy | Returns mongo.ErrNoDocuments | `repository/mongo.go` |
| B8 | Empty legacy slice | Not in collection, empty []OrganizationMember | Returns mongo.ErrNoDocuments | `repository/mongo.go` |
| B9 | Nil legacy slice | Not in collection, nil slice | Returns mongo.ErrNoDocuments | `repository/mongo.go` |
| B10 | DB error propagated | DB error (not ErrNoDocuments) | Error returned, no fallback attempted | `repository/mongo.go` |
| B11 | Invalid orgID | "not-a-hex-id" | Returns error | `repository/mongo.go` |
| B12 | Invalid userID | "not-a-hex-id" | Returns error | `repository/mongo.go` |

### C. IncrementMemberCreditsUsed

| # | Scenario | Input | Expected | Component |
|---|----------|-------|----------|-----------|
| C1 | Found in memberships | Membership doc exists | studioCreditsUsed incremented, updatedAt set | `repository/mongo.go` |
| C2 | Not in memberships — legacy fallback | No doc, embedded member exists | Legacy array member.studioCreditsUsed incremented | `repository/mongo.go` |
| C3 | Not found anywhere | No doc, no embedded member | Error: "no active membership found..." | `repository/mongo.go` |
| C4 | DB error on memberships update | DB failure on first UpdateOne | Error propagated immediately | `repository/mongo.go` |
| C5 | DB error on legacy fallback | DB failure on legacy UpdateOne | Error propagated | `repository/mongo.go` |
| C6 | Invalid orgID | "bad-id" | Error returned | `repository/mongo.go` |
| C7 | Invalid userID | "bad-id" | Error returned | `repository/mongo.go` |

### D. StudioAccessMiddleware

| # | Scenario | Input | Expected | Component |
|---|----------|-------|----------|-----------|
| D1 | Repo is nil — fail closed | repo == nil | 503 STUDIO_SERVICE_UNAVAILABLE | `middleware/studio_access.go` |
| D2 | Owner bypasses member checks | Owner JWT | Proceeds to credit checks | `middleware/studio_access.go` |
| D3 | Member found in new collection | Member in org_memberships | Access granted if studioAccess true | `middleware/studio_access.go` |
| D4 | Member found via legacy fallback | Member in org.Members only | Access granted if studioAccess true | `middleware/studio_access.go` |
| D5 | Non-member denied | User not in org | 403 STUDIO_ACCESS_DENIED | `middleware/studio_access.go` |
| D6 | Member without studioAccess | studioAccess: false | 403 STUDIO_ACCESS_DENIED | `middleware/studio_access.go` |
| D7 | DB error during membership fetch | DB error ≠ ErrNoDocuments | 503 STUDIO_SERVICE_UNAVAILABLE | `middleware/studio_access.go` |
| D8 | Prefetched member passed to GetStudioUsageStatus | Member already fetched | No redundant DB call | `middleware/studio_access.go` |
| D9 | Usage status endpoint skips validation | GET /studio/usage/status | Passes through without member check | `middleware/studio_access.go` |

### E. GetStudioUsageStatus

| # | Scenario | Input | Expected | Component |
|---|----------|-------|----------|-----------|
| E1 | With prefetched member | prefetchedMember != nil | Uses provided member, no DB query | `repository/mongo.go` |
| E2 | Without prefetched member | prefetchedMember == nil, non-owner | Fetches via GetOrganizationMembership | `repository/mongo.go` |
| E3 | Owner has no personalLimit | Owner JWT | personalLimit is nil in response | `repository/mongo.go` |
| E4 | Member personalLimit and personalUsed | Member with credit limit | personalLimit and personalUsed populated | `repository/mongo.go` |
| E5 | Member not found | Non-member, no prefetch | Error: "user X is not a member of org Y" | `repository/mongo.go` |

### F. Migration

| # | Scenario | Input | Expected | Component |
|---|----------|-------|----------|-----------|
| F1 | First-time migration | Orgs with embedded members | membersCreated > 0 | `repository/migration.go` |
| F2 | Idempotent re-run | Already migrated | membersSkipped == previous created, membersCreated == 0 | `repository/migration.go` |
| F3 | Legacy docs normalized | Docs with `user` field | legacyDocsNormalized > 0, field renamed to `userId` | `repository/migration.go` |
| F4 | Org with empty members | org.members == [] | Org skipped, not counted | `repository/migration.go` |
| F5 | Duplicate userId in same org | Two members with same user in embedded array | One created, one skipped (duplicate key) | `repository/migration.go` |
| F6 | Repo is nil | handler.repo == nil | 503 "MongoDB is unavailable" | `handlers/studio.go` |
| F7 | Admin auth required | Non-admin JWT | 401/403 (depending on auth middleware) | `cmd/server/main.go` |

### G. Degraded Mode

| # | Scenario | Input | Expected | Component |
|---|----------|-------|----------|-----------|
| G1 | LoggingMiddleware — repo nil | Usage log with nil repo | Skipped, log warning printed | `middleware/logging.go` |
| G2 | UsageWorker — repo nil | Usage event with nil mongoRepo | Skipped, log warning printed | `repository/worker.go` |
| G3 | StudioAccess — repo nil | Any request with nil repo | 503 (not silently allowed) | `middleware/studio_access.go` |
| G4 | MigrateMembers — repo nil | POST migrate with nil repo | 503 | `handlers/studio.go` |

### H. membershipRaw.toMembership Normalization

| # | Scenario | Input | Expected | Component |
|---|----------|-------|----------|-----------|
| H1 | Canonical schema (userId set) | raw.UserID populated | Membership.UserID == raw.UserID | `repository/mongo.go` |
| H2 | Legacy schema (user set, userId zero) | raw.User populated, UserID zero | Membership.UserID == raw.User | `repository/mongo.go` |
| H3 | Both fields set | Both raw.UserID and raw.User | Prefers UserID (canonical) | `repository/mongo.go` |
| H4 | All studio fields preserved | All credit fields set | Membership has matching values | `repository/mongo.go` |

---

## Post-Testing Checklist

- [ ] All unit tests pass (`go test ./internal/... -v`)
- [ ] Migration endpoint returns correct counts
- [ ] Migration is idempotent (safe to re-run)
- [ ] Legacy `user` field normalized to `userId` after migration
- [ ] Owner access works without membership lookup
- [ ] Member access resolved from `organization_memberships` collection
- [ ] Member access falls back to `org.Members` when not migrated
- [ ] Non-member correctly denied with `STUDIO_ACCESS_DENIED`
- [ ] `studioAccess: false` correctly denied
- [ ] Credit increment updates `organization_memberships` document
- [ ] Credit increment does NOT touch legacy embedded array (when membership doc exists)
- [ ] `GetStudioUsageStatus` returns `personalLimit` and `personalUsed` from membership
- [ ] Prefetched member avoids redundant DB call
- [ ] 503 returned when MongoDB is unavailable (all fail-closed paths)
- [ ] Logging and worker gracefully skip writes in degraded mode
- [ ] Fire-and-forget normalization renames `user` → `userId` on legacy-dump docs
- [ ] Unique partial index `idx_org_userId_unique_active` exists on collection
- [ ] Legacy index `idx_org_user_legacy_deleted` exists for backward compat queries
