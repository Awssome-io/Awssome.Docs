# Freemium Solution 2: Multi-Tier Pooled Architecture (RECOMMENDED)

## Overview

Multiple smaller RDS SQL Server pools (250 users each) with multi-tenant table design using Row-Level Security. Provides better isolation, smaller blast radius, and flexibility to separate competitors.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│           SINGLE HUBSPOT APP (Marketplace)                       │
│  • OAuth scopes: crm.objects.deals.read                         │
│  • Installs on customer's HubSpot account                       │
└─────────────────────────────────────────────────────────────────┘
                      │ OAuth flow (all tiers)
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT PORTAL                              │
│  • Tenant registry with "tier" field: free_pool_1, free_pool_2  │
│  • Upgrade triggers pool → dedicated migration                   │
│  • Tenant tier tracking (free vs paid)                          │
└─────────────────────────────────────────────────────────────────┘
                                ▲
                                │
                ┌───────────────┼────────────────┐
                │               │                │
                ▼               ▼                ▼
┌──────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  FREEMIUM TENANTS    │  │  FREEMIUM TENANTS│  │  PAID TENANT 1   │
│  Pool 1 (250 users)  │  │  Pool 2 (250 usr)│  │ customer1.aws... │
│ freemium.awssome.io  │  │ freemium.aws...  │  ├──────────────────┤
├──────────────────────┤  ├──────────────────┤  │ Dedicated EB     │
│ Shared EB Instance   │  │ Shared EB        │  │ + HubSpot Sync   │
│ + HubSpot Sync UI    │  │ + HubSpot Sync   │  └────────┬─────────┘
└──────────┬───────────┘  └────────┬─────────┘           │
           │                       │                      ▼
           ▼                       ▼          ┌──────────────────────┐
┌──────────────────────┐  ┌──────────────────┐│ DEDICATED RDS       │
│ POOL 1 RDS SQL SVR   │  │ POOL 2 RDS SQL   ││ Full isolation      │
├──────────────────────┤  ├──────────────────┤│ + HubSpot (full)    │
│ DB: freemium_pool_1  │  │ DB: freemium_p_2 │└──────────────────────┘
│ Table: DealRegs      │  │ Table: DealRegs  │
│  tenant_id (PK part) │  │  tenant_id (PK)  │
│                      │  │                  │
│ Row-Level Security:  │  │ Row-Level Sec:   │
│ WHERE tenant_id =    │  │ WHERE tenant_id= │
│   @session_tenant    │  │   @session_tenant│
│                      │  │                  │
│ Tier-aware sync:     │  │ Tier-aware:      │
│ • Free: 10 deals max │  │ • Free: 10 deals │
│ • Paid: Unlimited    │  │ • Paid: Unlimited│
└──────────────────────┘  └──────────────────┘
```

## Pool Management Strategy

```
Pool Lifecycle:
• Start: Pool 1 (0-250 users)
• Scale: When Pool 1 > 250, create Pool 2
• Balance: Distribute new signups across pools
• Isolate: If Company X & Y are competitors, assign to different pools
• Upgrade: Move high-usage tenants to dedicated infrastructure
```

## Upgrade Flow

```
user_X in pool_1 → Triggers:
  1. Spin up dedicated RDS + Elastic Beanstalk
  2. Migrate WHERE tenant_id = 'user_X' → new DB
  3. Update Management Portal registry
  4. Point userX.awssome.io → new infrastructure
  5. DELETE FROM pool_1 WHERE tenant_id = 'user_X'
```

## Implementation Details

### Database Schema

**Multi-Tenant Table Design:**

All tables include `tenant_id` column as part of primary key for partition and isolation:

**Core Tables:**
- **DealRegistrations**: AWS Marketplace deals
  - Composite key: (tenant_id, id)
  - Indexes on tenant_id + status, tenant_id + created_at
- **TenantMetadata**: Pool assignments, deal limits, storage tracking
  - Tracks which pool each tenant belongs to
  - Records AWS role ARN for Marketplace access

**HubSpot Tables:**
- **HubSpotTokens**: OAuth access/refresh tokens per tenant
  - One connection per tenant (unique constraint on tenant_id)
  - Stores HubSpot portal ID and granted scopes
- **HubSpotDealMappings**: Links HubSpot deals → Awssome deal registrations
  - Composite unique key: (tenant_id, hubspot_deal_id)
  - Foreign key to DealRegistrations
- **HubSpotSyncHistory**: Audit log of sync operations
  - Tracks fetched vs imported deal counts
  - Records errors and sync duration

**Key Design Principles:**
- Every table has `tenant_id` column
- All queries filtered by `tenant_id`
- Composite keys ensure tenant isolation
- Indexes optimized for tenant-specific queries

### Row-Level Security (RLS)

**What is RLS?**
SQL Server 2016+ feature that automatically filters rows based on session context. Enforced at database engine level (impossible to bypass in queries).

**How It Works:**
1. Create security predicate function that checks session context
2. Apply filter policy to all tables
3. Application sets `tenant_id` in session context at request start
4. All subsequent queries automatically filtered to show only that tenant's data
5. Cannot query other tenants' data even with malicious SQL

**Applied To:**
- DealRegistrations
- HubSpotTokens
- HubSpotDealMappings
- HubSpotSyncHistory
- All multi-tenant tables

**Benefits:**
- Engine-level enforcement (not application-level)
- Impossible to accidentally query wrong tenant
- Prevents SQL injection cross-tenant attacks
- DBA access still possible for support/debugging

### Application Integration

**Request Flow:**
1. User makes request to freemium.awssome.io
2. Backend extracts `tenant_id` from JWT/subdomain/session
3. Opens database connection to assigned pool
4. Executes: `SET SESSION_CONTEXT 'TenantID' = 'user_abc123'`
5. All queries automatically filtered by RLS
6. Returns only that tenant's data

**Example Query:**
- App code: `SELECT * FROM DealRegistrations WHERE status = 'active'`
- RLS adds: `... AND tenant_id = 'user_abc123'`
- Result: Only user_abc123's active deals

### Pool Assignment Logic

**Signup Process:**
1. New user signs up with company name
2. System checks if competitor already exists in any pool
3. Finds pool with capacity (<250 users) that doesn't have competitor
4. If all pools full or have competitor, provisions new pool
5. Assigns user to selected pool
6. Stores mapping in TenantMetadata table

**Competitor Detection Methods:**
- Keyword matching (e.g., "Acme Corp" vs "Acme Inc")
- Manual tagging via Management Portal
- Machine learning classification (future enhancement)
- Industry/vertical categorization

**Load Balancing:**
- Fill pools evenly (distribute 250 users across pools)
- Can manually move high-usage tenants to dedicated pool
- Monitor per-pool resource usage

### Migration on Upgrade

**Process:**
1. **Provision dedicated RDS** - Create new SQL Server instance (Terraform/AWS API)
2. **Copy tenant data** - Bulk copy all rows WHERE tenant_id = 'user_abc123':
   - DealRegistrations
   - HubSpotTokens
   - HubSpotDealMappings
   - HubSpotSyncHistory
3. **Update Management Portal** - Point tenant to new DB host, mark as 'paid' tier
4. **Update DNS** - Route userX.awssome.io → new infrastructure
5. **Soft delete** - Mark pool data as 'migrated' (retain 30 days for rollback)
6. **Hard delete** - Remove from pool after 30-day retention period

**Key Points:**
- Dedicated DB doesn't need `tenant_id` column (single tenant)
- HubSpot OAuth tokens automatically migrate (no app reinstall)
- Sync resumes with unlimited deal limit (paid tier)
- Zero downtime possible with dual-run strategy
- 30-day retention allows rollback if issues found

## HubSpot Integration

### Overview

Single HubSpot app for all tiers with tier-aware backend. Same implementation as Solution 1, but with multi-tenant table design and Row-Level Security.

### HubSpot API Constraints

**Rate Limits (per customer's HubSpot account):**
- **Free/Starter**: 100 req/10sec, 250k daily
- **Professional**: 190 req/10sec, 625k daily
- **Enterprise**: 190 req/10sec, 1M daily
- **Burst limit**: 110 req/10sec per OAuth app install

**Key Points:**
- Rate limits apply to **customer's HubSpot account**, not your app
- Manual sync trigger = lower API usage
- Batch API recommended (100 deals per request)

### Tier-Aware Sync Implementation

**Sync Process:**
1. User clicks "Sync Deals from HubSpot"
2. Backend gets tenant info from Management Portal (pool, tier)
3. Opens connection to assigned pool database
4. **Sets RLS context**: Executes `SET SESSION_CONTEXT 'TenantID' = 'user_abc123'`
5. Queries HubSpotTokens table (RLS automatically filters to current tenant)
6. Checks tier: Free = 10 deals max, Paid = unlimited
7. Fetches deals from HubSpot using Batch API
8. Imports deals as Awssome Deal Registrations
9. Creates HubSpotDealMappings (links HubSpot deal ID → Awssome deal ID)
10. Logs sync history with success/failure metrics

**RLS Enforcement:**
- All queries automatically include `WHERE tenant_id = 'user_abc123'`
- Cannot accidentally query other tenants' data
- Cannot insert data for other tenants (BLOCK predicate)
- Engine-level enforcement (not application-level)

### Row-Level Security for HubSpot

**How RLS Protects HubSpot Data:**

**Session Context:**
- Application sets tenant_id once at request start
- All subsequent queries filtered automatically
- No need to add `WHERE tenant_id = X` to every query

**Automatic Filtering:**
- Query: `SELECT * FROM HubSpotTokens`
- RLS adds: `WHERE tenant_id = 'user_abc123'`
- Returns: Only current tenant's OAuth tokens

**Cross-Tenant Protection:**
- Query: `SELECT * FROM HubSpotTokens WHERE tenant_id = 'other_user'`
- RLS blocks: Returns empty result
- Engine refuses to return other tenants' data

**Benefits:**
- Impossible to access competitor's HubSpot OAuth tokens
- Automatic protection against SQL injection attacks
- No manual WHERE clause maintenance
- Works across all HubSpot tables (Tokens, Mappings, SyncHistory)

### HubSpot Integration Benefits for Pooled Architecture

✅ **RLS protection**: Automatic tenant isolation, impossible to access other tenants' HubSpot data
✅ **Pool flexibility**: Can move high-usage HubSpot syncs to dedicated pool
✅ **Competitor isolation**: Competing companies' HubSpot data in separate pools
✅ **Single OAuth app**: Same app for freemium & paid tiers
✅ **Seamless upgrade**: HubSpot tokens migrate with data, no reinstall needed
✅ **Tier-aware**: Backend enforces 10-deal limit for free users

## Pros

✅ **Flexible scaling**: Add pools as needed, don't over-provision
✅ **Competitor isolation**: Manually assign sensitive companies to separate pools
✅ **Smaller blast radius**: Pool failure affects 250 users, not 1000+
✅ **Industry standard**: Multi-tenant table pattern is proven at scale (Salesforce, HubSpot)
✅ **Cost effective**: Still pool resources but with more control
✅ **Performance isolation**: Can upgrade high-usage pools to larger instances
✅ **A/B testing friendly**: Can test features on Pool 2 before Pool 1
✅ **Gradual rollout**: Start with 1 pool, add more as you grow
✅ **Load balancing**: Distribute new users evenly across pools
✅ **Operational visibility**: Monitor per-pool metrics separately
✅ **HubSpot RLS protection**: Automatic tenant isolation for HubSpot data
✅ **HubSpot pool flexibility**: Move high-usage syncs to dedicated pool

## Cons

❌ **More complex management**: Need pool assignment logic and competitor detection
❌ **Still shared infrastructure**: Multiple tenants per DB (perception issue remains)
❌ **Migration overhead**: Need to track which pool each tenant is in
❌ **Requires RLS**: Must implement Row-Level Security correctly (security risk if bugs)
❌ **Connection string management**: Need to route queries to correct pool
❌ **Higher base cost**: 2-4 pools = $200-400/month vs $100 for single DB
❌ **Schema design**: Must include tenant_id in all tables (can't forget)
❌ **Testing complexity**: Need to test RLS policies thoroughly

## Cost Analysis

### Freemium Tier (1000 users across 4 pools)

**Per Pool:**
- **RDS SQL Server Web Edition** (db.t3.small): ~$50/month
  - 2 vCPU, 2GB RAM
  - 50GB storage
  - 250 users per pool

**Total Cost:**
- 4 pools × $50/month = **$200/month** for 1000 users
- Cost per user: **$0.20/month**

### Scaling Strategy

| User Count | Pools | Instance Size | Monthly Cost | Per User |
|------------|-------|---------------|--------------|----------|
| 0-250 | 1 | db.t3.small | $50 | $0.20 |
| 251-500 | 2 | db.t3.small | $100 | $0.20 |
| 501-1000 | 4 | db.t3.small | $200 | $0.20 |
| 1001-2500 | 10 | db.t3.small | $500 | $0.20 |
| High usage | 1 | db.t3.medium | $100 | - |

### Comparison to Solution 1

| Metric | Solution 1 (Single DB) | Solution 2 (Pooled) |
|--------|------------------------|---------------------|
| Base cost | $100/month | $200/month |
| Per user | $0.10 | $0.20 |
| Blast radius | 1000 users | 250 users |
| Flexibility | Low | High |

**Tradeoff:** 2x cost for 4x reduction in blast radius + competitor isolation

## Security Considerations

### Strengths
- **Row-Level Security**: SQL Server enforces tenant isolation at DB engine level
- **Smaller attack surface**: 250 users per pool vs 1000+
- **Pool isolation**: Competitor data physically separated on different RDS instances
- **Audit trail**: Can track all RLS policy violations
- **Principle of least privilege**: App user only sees own tenant data

### Weaknesses
- **RLS bugs**: If predicate function has bug, could leak data across tenants
- **Session context**: Must properly set tenant context on every request
- **DBA access**: Database admin can still bypass RLS (IS_MEMBER check)
- **Shared compute**: CPU/memory/disk still shared within pool
- **Metadata exposure**: Can infer tenant count, data sizes from metrics

### Mitigations

```sql
-- Test RLS policy thoroughly
-- Create test users in different tenants
CREATE USER tenant_a_user WITHOUT LOGIN;
EXEC sp_set_session_context @key = N'TenantID', @value = 'tenant_a';

-- Try to access tenant B data (should return empty)
SELECT * FROM DealRegistrations WHERE tenant_id = 'tenant_b';
-- Expected: 0 rows (blocked by RLS)

-- Add safety checks in application
public async Task ValidateTenantAccess(string tenantId, string dealId)
{
    var deal = await _db.DealRegistrations.FindAsync(dealId);
    if (deal == null || deal.TenantId != tenantId)
    {
        throw new UnauthorizedAccessException("Cross-tenant access denied");
    }
}
```

## Implementation Timeline

### Phase 1: Single Pool MVP (2-3 weeks)
- [ ] Design multi-tenant schema with tenant_id
- [ ] Implement Row-Level Security policies
- [ ] Build pool provisioning automation (Terraform)
- [ ] Create signup flow with pool assignment
- [ ] Integrate RLS with application (set session context)
- [ ] Test RLS policies thoroughly (cross-tenant access prevention)

### Phase 2: Multi-Pool Support (1-2 weeks)
- [ ] Build pool assignment logic (competitor detection)
- [ ] Implement load balancing across pools
- [ ] Create pool monitoring dashboard
- [ ] Add pool capacity alerting (>80% full)
- [ ] Automate new pool provisioning

### Phase 3: HubSpot Integration (2-3 weeks)
- [ ] Create HubSpot app in developer portal (OAuth scopes: crm.objects.deals.read)
- [ ] Implement OAuth flow (authorization, callback, token storage)
- [ ] Add HubSpot tables to pool schemas with RLS policies
- [ ] Build tier-aware sync service (10 deals for free, unlimited for paid)
- [ ] Implement rate limiting and batch API calls
- [ ] Create "Connect HubSpot" UI in Awssome portal
- [ ] Build manual sync trigger with progress indicator
- [ ] Test RLS policies for HubSpot tables thoroughly
- [ ] Test OAuth flow for freemium & paid tenants

### Phase 4: Migration Flow (1-2 weeks)
- [ ] Build upgrade trigger in Management Portal
- [ ] Automate dedicated RDS provisioning
- [ ] Implement tenant data migration pipeline (include HubSpot tables)
- [ ] Create DNS/routing update automation
- [ ] Build soft delete + cleanup job (30-day retention)
- [ ] Test migration rollback procedure
- [ ] Test HubSpot integration continues working post-migration

### Phase 5: Operations & Monitoring (1 week)
- [ ] CloudWatch metrics per pool (query rate, data size, tenant count)
- [ ] Alerting for pool health (high CPU, storage full, slow queries)
- [ ] Admin dashboard showing pool distribution
- [ ] Operational runbook for pool management
- [ ] Add HubSpot sync monitoring (success rate, API usage per pool)
- [ ] Set up alerts for OAuth token expiration

## Operational Runbook

### Add New Freemium User

**Process:**
1. User completes signup on freemium.awssome.io
2. Run automated pool assignment script (checks competitors, capacity)
3. Assign user to appropriate pool (pool_1, pool_2, etc.)
4. Insert tenant metadata (pool name, AWS role ARN, deal limit)
5. User can now access their portal and create deals
6. RLS automatically filters all queries to their tenant_id

### Provision New Pool

**Trigger:** When existing pools hit 80% capacity (200+ users)

**Automated Steps:**
1. Terraform provisions new RDS SQL Server instance
2. Creates security groups and parameter groups
3. Sets up CloudWatch alarms and backup policies
4. Runs database schema migrations (creates all tables)
5. Updates Management Portal with new pool availability
6. Ready to accept new tenant assignments

**Infrastructure Created:**
- RDS SQL Server instance (freemium-pool-X-db)
- VPC security groups
- CloudWatch metrics and alarms
- Automated backup schedule (daily)
- Parameter groups with optimal settings

### Upgrade User to Paid

**Process:**
1. Trigger upgrade from Management Portal
2. Provision new dedicated RDS instance (Terraform)
3. Copy all tenant data WHERE tenant_id = 'user_abc123'
4. Update CustomerRegistry (mark as 'paid', point to new DB)
5. Update Route53 DNS (userX.awssome.io → new infrastructure)
6. Soft delete from pool (mark as 'migrated', retain 30 days)
7. Schedule permanent cleanup after retention period

### Monitor Pool Health

**Key Metrics:**
- **Tenant count per pool** - Track distribution across pools
- **Top storage consumers** - Identify heavy users in each pool
- **Query performance** - Monitor slow queries by tenant
- **HubSpot sync metrics** - Success rate, API usage per pool
- **Resource utilization** - CPU, memory, disk per pool
- **Error rates** - Track RLS policy violations, failed syncs

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| RLS policy bug leaks data | Low | Critical | Extensive testing, code review, audit logging |
| Pool outage affects 250 users | Medium | Medium | Multi-AZ deployment, automated failover, quick recovery |
| Session context not set | Low | Critical | Middleware enforces context, integration tests |
| Competitor detection fails | Medium | Low | Manual override in Management Portal |
| Pool capacity exhausted | Low | Medium | Alerting at 80%, auto-provision new pool |
| Migration data loss | Low | Critical | Test migration, 30-day retention, rollback plan |

## Success Metrics

- **Cost efficiency**: <$0.25 per freemium user per month
- **Isolation**: 0 cross-tenant data leaks
- **Availability**: 99.5% uptime per pool
- **Signup time**: <5 minutes registration to usable portal
- **Migration time**: <15 minutes downtime during upgrade
- **Competitor separation**: 100% of identified competitors in different pools
- **Conversion rate**: Track freemium → paid by pool

## Recommended Hybrid Approach

For maximum security and conversion optimization:

```
FREEMIUM TIER (Evaluation Only):
├─ Use pooled architecture (Solution 2)
├─ Clear messaging: "For testing, not production deals"
├─ Limit: 10 deals per freemium user
├─ No SLA guarantee
└─ Automatic data deletion after 90 days of inactivity

PAID TIER (Production):
├─ Dedicated RDS from day 1
├─ Full feature access
├─ 99.9% SLA
├─ SOC2/HIPAA compliant
└─ Marketing: "Your competitors can't access your data"

UPGRADE PATH:
├─ 1-click migration from pool → dedicated
├─ Data verification window (7 days dual-run)
├─ Automatic cutover with DNS update
└─ Data retention in pool for 30 days (rollback safety)
```

## Why This Solution is Recommended

1. **Balances cost and isolation**: 2x cost of Solution 1, but 4x better isolation
2. **Competitor-aware**: Can separate competing companies across pools
3. **Flexible growth**: Start with 1 pool, add more as needed (no over-provisioning)
4. **Industry standard**: Multi-tenant table design is battle-tested
5. **Clear upgrade path**: Simple data migration (WHERE tenant_id = X)
6. **Operational flexibility**: Can upgrade high-usage pools independently
7. **Risk mitigation**: Smaller blast radius (250 vs 1000 users)

## When to Use This Solution

**Choose Solution 2 if:**
- Competitor data isolation is important
- Want flexibility to scale gradually
- Can accept 2x cost for better isolation
- Have operational capacity for multi-pool management
- Need ability to separate specific customers

**Avoid Solution 2 if:**
- Cost is absolute priority (use Solution 1)
- Don't care about competitor isolation
- Team is small (can't manage multiple pools)
- Prefer simplicity over flexibility
