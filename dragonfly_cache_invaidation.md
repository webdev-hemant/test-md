# Dragonfly Cache Invalidation Plan

> **Context:** The NitroStudio gateway (Go) introduced a DragonflyDB cache layer for
> organization and member state (PR #58). This document describes the changes needed
> in the NitroCloud backend (NestJS) to keep that cache fresh by invalidating stale
> keys whenever the underlying MongoDB data is mutated.

---

## Architecture Overview

The gateway caches two types of state in DragonflyDB:

| Key pattern | Data cached |
|-------------|-------------|
| `gway:org:{orgId}` | Org credits, plan, model config, overage settings, free-credit state |
| `gway:member:{orgId}:{userId}` | Studio access flag, credit limit, credits used, additional limit |

On a cache miss the gateway **seeds on demand** from MongoDB via a `singleflight`
group (deduplicates concurrent seeds for the same key). Cache entries expire after a
configurable TTL (default 7 days).

### Invalidation Flow

```
NitroCloud Backend                 DragonflyDB                Studio Gateway
       │                               │                          │
       │  1. Mutate MongoDB             │                          │
       │──────────────────►             │                          │
       │                               │                          │
       │  2. DEL gway:org:{orgId}      │                          │
       │──────────────────────────────► │                          │
       │                               │                          │
       │                               │  3. HGETALL (cache miss) │
       │                               │ ◄─────────────────────── │
       │                               │                          │
       │                               │  4. Seed from MongoDB    │
       │                               │     HSET ...             │
       │                               │ ─────────────────────► ──│
       │                               │                          │
```

**Strategy:** Delete-on-write. The backend DELs the cache key immediately after
mutating MongoDB. The gateway re-seeds from MongoDB on the next request that hits
that key. This approach is simple, reliable, and requires zero coordination protocol
between the two services — they only share the DragonflyDB instance.

---

## Mutation Points

### Org State (`gway:org:{orgId}`) — 14 invalidation points

| # | File | Method | Trigger |
|---|------|--------|---------|
| 1 | `organizations/organizations.service.ts` | `initializeFreeCreditFields` | New org created (free credit defaults set) |
| 2 | `organizations/organizations.service.ts` | `updateStudioConfig` | Admin changes model config (allowedModels, paidModelsEnabled, etc.) |
| 3 | `billing/billing.service.ts` | `createSubscription` | New subscription created |
| 4 | `billing/billing.service.ts` | `updateSubscriptionPlan` | Plan upgrade or downgrade |
| 5 | `billing/billing.service.ts` | `cancelSubscription` | Subscription cancelled |
| 6 | `billing/billing.service.ts` | `syncSubscriptionFromCheckout` | Stripe checkout session completed |
| 7 | `billing/billing.service.ts` | `toggleUsageBasedBilling` | Overage billing toggled on/off |
| 8 | `billing/billing.service.ts` | `updateUsageSettings` | Usage limits or auto-topup settings changed |
| 9 | `billing/billing.service.ts` | `topupStudioCredits` | Studio credit top-up (Stripe invoice flow) |
| 10 | `billing/stripe-webhook.controller.ts` | `handleInvoicePaid` | Stripe `invoice.paid` event |
| 11 | `billing/stripe-webhook.controller.ts` | `handlePaymentIntentSucceeded` | Stripe `payment_intent.succeeded` (top-up safety net) |
| 12 | `billing/stripe-webhook.controller.ts` | `handleChargeRefunded` | Stripe `charge.refunded` (credits decremented) |
| 13 | `billing/stripe-webhook.controller.ts` | `updateLocalSubscription` | Stripe `customer.subscription.updated` |
| 14 | `billing/stripe-webhook.controller.ts` | `handleSubscriptionDeleted` | Stripe `customer.subscription.deleted` |

### Member State (`gway:member:{orgId}:{userId}`) — 5 invalidation points

| # | File | Method | Trigger |
|---|------|--------|---------|
| 15 | `organizations/organizations.service.ts` | `addMember` | Member added directly |
| 16 | `organizations/organizations.service.ts` | `addMemberViaInvitation` | Member added via invitation |
| 17 | `organizations/organizations.service.ts` | `removeMember` | Member soft-deleted |
| 18 | `organizations/organizations.service.ts` | `updateMemberStudioAccess` | Studio access toggled |
| 19 | `organizations/organizations.service.ts` | `updateMemberStudioSettings` | Credit limit or additional usage limit changed |

> **Note:** `studio-request.service.ts` `processRequest` calls `updateMemberStudioSettings`
> internally, so it is covered transitively — no separate invalidation call needed.

**Total: 19 invalidation call sites across 4 files.**

---

## Implementation Steps

### Step 1: Extend `DragonflyService`

**File:** `billing/services/dragonfly.service.ts`

Add three methods to the existing service (which already connects to the same
DragonflyDB instance via ioredis):

```typescript
private readonly GATEWAY_ORG_PREFIX = 'gway:org:';
private readonly GATEWAY_MEMBER_PREFIX = 'gway:member:';

/**
 * Delete the gateway's cached org state, forcing a re-seed on next request.
 */
async invalidateGatewayOrgState(orgId: string): Promise<void> {
  try {
    await this.client.del(`${this.GATEWAY_ORG_PREFIX}${orgId}`);
    this.logger.log(`Invalidated gateway org cache for ${orgId}`);
  } catch (err: any) {
    this.logger.error(`Failed to invalidate gateway org cache for ${orgId}: ${err.message}`);
  }
}

/**
 * Delete the gateway's cached member state.
 */
async invalidateGatewayMemberState(orgId: string, userId: string): Promise<void> {
  try {
    await this.client.del(`${this.GATEWAY_MEMBER_PREFIX}${orgId}:${userId}`);
    this.logger.log(`Invalidated gateway member cache for ${orgId}/${userId}`);
  } catch (err: any) {
    this.logger.error(`Failed to invalidate gateway member cache for ${orgId}/${userId}: ${err.message}`);
  }
}

/**
 * Delete all gateway member keys for an org (SCAN + batch DEL).
 * Useful after bulk operations that affect all members.
 */
async invalidateAllGatewayMemberStates(orgId: string): Promise<void> {
  try {
    const pattern = `${this.GATEWAY_MEMBER_PREFIX}${orgId}:*`;
    let cursor = '0';
    do {
      const [nextCursor, keys] = await this.client.scan(cursor, 'MATCH', pattern, 'COUNT', 200);
      cursor = nextCursor;
      if (keys.length > 0) {
        await this.client.del(...keys);
      }
    } while (cursor !== '0');
    this.logger.log(`Invalidated all gateway member caches for org ${orgId}`);
  } catch (err: any) {
    this.logger.error(`Failed to invalidate gateway member caches for org ${orgId}: ${err.message}`);
  }
}
```

All methods are fire-and-forget with try/catch — cache invalidation failure must
never break the primary mutation flow.

### Step 2: Extract `DragonflyService` into a Shared Module

The existing `DragonflyService` lives inside `BillingModule` and is not exported.
Since `OrganizationsService` also needs it, we must avoid a circular dependency.

**Create:** `dragonfly/dragonfly.module.ts`

```typescript
import { Module, Global } from '@nestjs/common';
import { DragonflyService } from './dragonfly.service';

@Global()
@Module({
  providers: [DragonflyService],
  exports: [DragonflyService],
})
export class DragonflyModule {}
```

**Move:** `billing/services/dragonfly.service.ts` → `dragonfly/dragonfly.service.ts`

**Update imports in:**
- `billing/billing.module.ts` — remove local `DragonflyService` provider, import `DragonflyModule`
- `billing/billing.service.ts` — update import path
- `billing/stripe-webhook.controller.ts` — update import path
- `billing/services/plan-enforcement.service.ts` — update import path
- `billing/services/billing-scheduler.service.ts` — update import path
- `organizations/organizations.module.ts` — no change needed (`@Global`)
- `organizations/organizations.service.ts` — add `DragonflyService` injection

### Step 3: Add Invalidation Calls

#### `billing/billing.service.ts` (7 calls)

```typescript
// After each mutation, add:
await this.dragonflyService.invalidateGatewayOrgState(organizationId);
```

Methods: `createSubscription`, `updateSubscriptionPlan`, `cancelSubscription`,
`syncSubscriptionFromCheckout`, `toggleUsageBasedBilling`, `updateUsageSettings`,
`topupStudioCredits`.

#### `billing/stripe-webhook.controller.ts` (5 calls)

```typescript
await this.dragonflyService.invalidateGatewayOrgState(organizationId);
```

Methods: `handleInvoicePaid`, `handlePaymentIntentSucceeded`, `handleChargeRefunded`,
`updateLocalSubscription`, `handleSubscriptionDeleted`.

#### `organizations/organizations.service.ts` (8 calls)

Org invalidation:
```typescript
await this.dragonflyService.invalidateGatewayOrgState(orgId);
```

Methods: `initializeFreeCreditFields`, `updateStudioConfig`.

Member invalidation:
```typescript
await this.dragonflyService.invalidateGatewayMemberState(orgId, userId);
```

Methods: `addMember`, `addMemberViaInvitation`, `removeMember`,
`updateMemberStudioAccess`, `updateMemberStudioSettings`.

---

## Out of Scope

These are explicitly **not** covered by this plan (by design):

| Item | Reason |
|------|--------|
| `freeTokensUsed` increments | Gateway-only writes (Go `UsageWorker`); no NitroCloud mutation exists |
| `membership.studioCreditsUsed` increments | Gateway-only writes; the gateway updates both Dragonfly and MongoDB |
| Billing period reset script | Manual/cron operation (`scripts/reset-studio-credits.ts`); can be added separately |
| Admin panel write endpoints | Currently read-only for studio credits; add invalidation if write endpoints are introduced |
| Rate limit keys (`gway:rl:*`) | Self-expiring per-minute counters; no invalidation needed |

---

## Gateway Cache Key Reference

For completeness, here are all key patterns the gateway uses in DragonflyDB:

| Pattern | Type | TTL | Purpose |
|---------|------|-----|---------|
| `gway:org:{orgId}` | Hash | 7d (configurable) | Org credits, plan, model config, overage |
| `gway:member:{orgId}:{userId}` | Hash | 7d (configurable) | Member access, credit limits |
| `gway:rl:{orgId}:{userId}:{minute}` | String (counter) | 120s | Per-user per-minute rate limit |

---

## Environment Variables

The `DragonflyService` in NitroCloud connects via:

| Variable | Default | Description |
|----------|---------|-------------|
| `DRAGONFLY_URL` | `redis://localhost:6379` | DragonflyDB connection URL |
| `DRAGONFLY_PASSWORD` | (none) | Optional auth password |

Both backends (NitroCloud and the Studio gateway) must point to the **same**
DragonflyDB instance for invalidation to work.

---

## Files Changed Summary

| Action | File | Change |
|--------|------|--------|
| Create | `dragonfly/dragonfly.module.ts` | New shared global module |
| Move | `billing/services/dragonfly.service.ts` → `dragonfly/dragonfly.service.ts` | Relocate + add 3 invalidation methods |
| Modify | `billing/billing.module.ts` | Import `DragonflyModule`, remove local provider |
| Modify | `billing/billing.service.ts` | Update import path, add 7 invalidation calls |
| Modify | `billing/stripe-webhook.controller.ts` | Update import path, add 5 invalidation calls |
| Modify | `billing/services/plan-enforcement.service.ts` | Update import path |
| Modify | `billing/services/billing-scheduler.service.ts` | Update import path |
| Modify | `organizations/organizations.service.ts` | Inject `DragonflyService`, add 8 invalidation calls |
| Modify | `app.module.ts` | Import `DragonflyModule` (if not using `@Global`) |
