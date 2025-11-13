# Freemium Solution 2: Multi-Tier Pooled Architecture (RECOMMENDED)

## Overview

Multiple smaller RDS SQL Server pools (250 users each) with multi-tenant table design using Row-Level Security. Provides better isolation, smaller blast radius, and flexibility to separate competitors.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT PORTAL                              │
│  • Tenant registry with "tier" field: free_pool_1, free_pool_2  │
│  • Upgrade triggers pool → dedicated migration                   │
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
│ Shared EB Instance   │  │ Shared EB        │  └────────┬─────────┘
└──────────┬───────────┘  └────────┬─────────┘           │
           │                       │                      ▼
           ▼                       ▼          ┌──────────────────────┐
┌──────────────────────┐  ┌──────────────────┐│ DEDICATED RDS       │
│ POOL 1 RDS SQL SVR   │  │ POOL 2 RDS SQL   ││ Full isolation      │
├──────────────────────┤  ├──────────────────┤└──────────────────────┘
│ DB: freemium_pool_1  │  │ DB: freemium_p_2 │
│ Table: DealRegs      │  │ Table: DealRegs  │
│  tenant_id (PK part) │  │  tenant_id (PK)  │
│  user_id             │  │  user_id         │
│  deal_data...        │  │  deal_data...    │
│                      │  │                  │
│ Row-Level Security:  │  │ Row-Level Sec:   │
│ WHERE tenant_id =    │  │ WHERE tenant_id= │
│   @session_tenant    │  │   @session_tenant│
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

```sql
-- Multi-tenant table design
CREATE TABLE DealRegistrations (
    id BIGINT IDENTITY PRIMARY KEY,
    tenant_id VARCHAR(50) NOT NULL,  -- Freemium user identifier
    deal_name NVARCHAR(255) NOT NULL,
    aws_role_arn VARCHAR(500),
    product_id VARCHAR(100),
    customer_name NVARCHAR(255),
    deal_value DECIMAL(18, 2),
    status VARCHAR(50) DEFAULT 'draft',
    created_at DATETIME2 DEFAULT GETDATE(),
    updated_at DATETIME2 DEFAULT GETDATE(),

    -- Composite key for tenant isolation
    CONSTRAINT PK_DealRegs PRIMARY KEY CLUSTERED (tenant_id, id),
    INDEX idx_tenant_status (tenant_id, status),
    INDEX idx_tenant_created (tenant_id, created_at DESC)
);

-- Supporting tables
CREATE TABLE TenantMetadata (
    tenant_id VARCHAR(50) PRIMARY KEY,
    pool_name VARCHAR(50) NOT NULL,  -- 'pool_1', 'pool_2', etc.
    aws_role_arn VARCHAR(500),
    deal_limit INT DEFAULT 10,
    storage_used_mb DECIMAL(10, 2) DEFAULT 0,
    created_at DATETIME2 DEFAULT GETDATE(),
    last_active DATETIME2,

    INDEX idx_pool (pool_name)
);
```

### Row-Level Security (SQL Server 2016+)

```sql
-- Create security predicate function
CREATE FUNCTION dbo.fn_TenantAccessPredicate(@tenant_id VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS fn_securitypredicate_result
WHERE @tenant_id = CAST(SESSION_CONTEXT(N'TenantID') AS VARCHAR(50))
    OR IS_MEMBER('db_owner') = 1;  -- Allow DBAs to see all
GO

-- Apply security policy to table
CREATE SECURITY POLICY TenantSecurityPolicy
ADD FILTER PREDICATE dbo.fn_TenantAccessPredicate(tenant_id)
    ON dbo.DealRegistrations,
ADD BLOCK PREDICATE dbo.fn_TenantAccessPredicate(tenant_id)
    ON dbo.DealRegistrations AFTER INSERT
WITH (STATE = ON);
GO
```

### Application Integration

```csharp
// Set tenant context at beginning of request
public async Task<IActionResult> OnActionExecuting(ActionExecutingContext context)
{
    var tenantId = GetTenantIdFromRequest(); // From JWT, subdomain, etc.

    using (var connection = new SqlConnection(connectionString))
    {
        await connection.OpenAsync();

        // Set session context for RLS
        using (var cmd = connection.CreateCommand())
        {
            cmd.CommandText = "EXEC sp_set_session_context @key, @value";
            cmd.Parameters.AddWithValue("@key", "TenantID");
            cmd.Parameters.AddWithValue("@value", tenantId);
            await cmd.ExecuteNonQueryAsync();
        }

        // All subsequent queries automatically filtered by RLS
        var deals = await _context.DealRegistrations
            .Where(d => d.Status == "active")
            .ToListAsync();  // Only returns deals for this tenant
    }
}
```

### Pool Assignment Logic

```csharp
public class PoolAssignmentService
{
    public async Task<string> AssignUserToPool(string newTenantId, string companyName)
    {
        // 1. Check if competitor exists in any pool
        var competitorPools = await _db.TenantMetadata
            .Where(t => IsCompetitor(t.CompanyName, companyName))
            .Select(t => t.PoolName)
            .Distinct()
            .ToListAsync();

        // 2. Find pool with capacity that doesn't have competitor
        var availablePool = await _db.TenantMetadata
            .GroupBy(t => t.PoolName)
            .Select(g => new {
                Pool = g.Key,
                Count = g.Count(),
                HasCompetitor = competitorPools.Contains(g.Key)
            })
            .Where(p => p.Count < 250 && !p.HasCompetitor)
            .OrderBy(p => p.Count)  // Fill pools evenly
            .FirstOrDefaultAsync();

        // 3. Create new pool if needed
        if (availablePool == null)
        {
            var nextPoolNum = await GetNextPoolNumber();
            var newPool = $"pool_{nextPoolNum}";
            await ProvisionNewPool(newPool);
            return newPool;
        }

        return availablePool.Pool;
    }

    private bool IsCompetitor(string company1, string company2)
    {
        // Simple keyword matching (enhance with ML/manual tagging)
        var keywords1 = company1.ToLower().Split(' ');
        var keywords2 = company2.ToLower().Split(' ');
        return keywords1.Intersect(keywords2).Any();
    }
}
```

### Migration on Upgrade

```sql
-- Step 1: Identify tenant data in pool
SELECT * FROM freemium_pool_1.dbo.DealRegistrations
WHERE tenant_id = 'user_abc123';

-- Step 2: Create new dedicated database
-- (Done via AWS RDS API/Terraform)

-- Step 3: Bulk copy tenant data to new DB
INSERT INTO customer_abc123_db.dbo.DealRegistrations
SELECT
    id,
    deal_name,
    aws_role_arn,
    product_id,
    customer_name,
    deal_value,
    status,
    created_at,
    updated_at
FROM freemium_pool_1.dbo.DealRegistrations
WHERE tenant_id = 'user_abc123';

-- Note: New DB doesn't need tenant_id column (single tenant)

-- Step 4: Update Management Portal registry
UPDATE CustomerRegistry
SET
    db_host = 'customer-abc123-db.xxxxx.us-east-1.rds.amazonaws.com',
    db_name = 'customer_abc123_db',
    pool_name = NULL,
    tier = 'paid',
    migrated_at = GETDATE()
WHERE tenant_id = 'user_abc123';

-- Step 5: Soft delete from pool (wait 30 days before hard delete)
UPDATE freemium_pool_1.dbo.DealRegistrations
SET status = 'migrated', updated_at = GETDATE()
WHERE tenant_id = 'user_abc123';

-- Later: Hard delete
DELETE FROM freemium_pool_1.dbo.DealRegistrations
WHERE tenant_id = 'user_abc123'
    AND status = 'migrated'
    AND updated_at < DATEADD(day, -30, GETDATE());
```

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

### Phase 3: Migration Flow (1-2 weeks)
- [ ] Build upgrade trigger in Management Portal
- [ ] Automate dedicated RDS provisioning
- [ ] Implement tenant data migration pipeline
- [ ] Create DNS/routing update automation
- [ ] Build soft delete + cleanup job (30-day retention)
- [ ] Test migration rollback procedure

### Phase 4: Operations & Monitoring (1 week)
- [ ] CloudWatch metrics per pool (query rate, data size, tenant count)
- [ ] Alerting for pool health (high CPU, storage full, slow queries)
- [ ] Admin dashboard showing pool distribution
- [ ] Operational runbook for pool management

## Operational Runbook

### Add New Freemium User

```bash
# 1. User completes signup → Receives tenant_id: user_abc123
# 2. Assign to pool
pool_name=$(./scripts/assign_pool.sh user_abc123 "Acme Corp")

# 3. Insert into TenantMetadata
sqlcmd -S ${pool_name}-db -Q "
INSERT INTO TenantMetadata (tenant_id, pool_name, aws_role_arn, deal_limit)
VALUES ('user_abc123', '${pool_name}', 'arn:aws:iam::123:role/Pub', 10)
"

# 4. User can now create deals (RLS automatically filters to their tenant_id)
```

### Provision New Pool

```bash
# When existing pools hit 80% capacity
./scripts/provision_pool.sh pool_5

# Terraform creates:
# - RDS SQL Server instance: freemium-pool-5-db
# - Security groups, parameter groups
# - CloudWatch alarms
# - Backup policies

# Run schema migrations
flyway migrate -url=jdbc:sqlserver://freemium-pool-5-db \
    -schemas=dbo -locations=filesystem:./migrations
```

### Upgrade User to Paid

```bash
# Trigger from Management Portal
./scripts/upgrade_tenant.sh user_abc123

# Script does:
# 1. terraform apply -var="tenant_id=user_abc123" (new RDS)
# 2. Migrate data: WHERE tenant_id = 'user_abc123'
# 3. Update CustomerRegistry (pool_name → NULL, tier → paid)
# 4. Update Route53: user_abc123.awssome.io → new EB endpoint
# 5. Soft delete from pool (status='migrated')
# 6. Schedule cleanup job (30 days)
```

### Monitor Pool Health

```sql
-- Tenant count per pool
SELECT pool_name, COUNT(*) as tenant_count
FROM TenantMetadata
GROUP BY pool_name
ORDER BY tenant_count DESC;

-- Top storage consumers per pool
SELECT TOP 10
    pool_name,
    tenant_id,
    storage_used_mb,
    last_active
FROM TenantMetadata
ORDER BY storage_used_mb DESC;

-- Query performance by tenant
SELECT
    tenant_id,
    COUNT(*) as query_count,
    AVG(duration_ms) as avg_duration,
    MAX(duration_ms) as max_duration
FROM QueryLog
WHERE timestamp > DATEADD(hour, -1, GETDATE())
GROUP BY tenant_id
ORDER BY query_count DESC;
```

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
