# Freemium Solution 1: Dedicated Freemium DB with Schema Isolation

## Overview

Single RDS SQL Server instance serving all freemium users with dedicated schemas per tenant. Simple migration path to dedicated infrastructure on upgrade.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT PORTAL                            │
│           (mgmt.awssome.io - Central DB + WebApp)               │
│  • Customer metadata, AWS roles, tenant registry                │
└─────────────────────────────────────────────────────────────────┘
                                ▲
                                │ Centralized API
                ┌───────────────┼───────────────┐
                │               │               │
                ▼               ▼               ▼
┌──────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  FREEMIUM TENANT     │  │  PAID TENANT 1   │  │  PAID TENANT 2   │
│ freemium.awssome.io  │  │ customer1.aws... │  │ customer2.aws... │
├──────────────────────┤  ├──────────────────┤  ├──────────────────┤
│ Elastic Beanstalk    │  │ Elastic Beanstalk│  │ Elastic Beanstalk│
│ (Shared Instance)    │  │ (Shared Instance)│  │ (Shared Instance)│
└──────────┬───────────┘  └────────┬─────────┘  └────────┬─────────┘
           │                       │                     │
           ▼                       ▼                     ▼
┌──────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ FREEMIUM RDS SQL     │  │ DEDICATED RDS    │  │ DEDICATED RDS    │
│ Server DB            │  │ SQL Server       │  │ SQL Server       │
├──────────────────────┤  │                  │  │                  │
│ Schema: user_1       │  │ All features     │  │ All features     │
│  ├─ DealReg tables   │  │ + DealReg        │  │ + DealReg        │
│ Schema: user_2       │  │                  │  │                  │
│  ├─ DealReg tables   │  └──────────────────┘  └──────────────────┘
│ Schema: user_N       │
│  ├─ DealReg tables   │
│ (1000+ schemas)      │
└──────────────────────┘
```

## Upgrade Flow

```
user_X in freemium → Triggers:
  1. Spin up new dedicated RDS for customer
  2. Migrate schema user_X data → new DB
  3. Update tenant registry in Management Portal
  4. Update DNS: userX.awssome.io → new infrastructure
  5. Drop schema user_X from freemium DB
```

## Implementation Details

### Schema Creation on Signup

```sql
-- Create dedicated schema for new freemium user
CREATE SCHEMA [user_abc123];

-- Create Deal Registration table
CREATE TABLE [user_abc123].[DealRegistrations] (
    id BIGINT IDENTITY PRIMARY KEY,
    deal_name NVARCHAR(255) NOT NULL,
    aws_role_arn VARCHAR(500),
    product_id VARCHAR(100),
    customer_name NVARCHAR(255),
    deal_value DECIMAL(18, 2),
    status VARCHAR(50) DEFAULT 'draft',
    created_at DATETIME2 DEFAULT GETDATE(),
    updated_at DATETIME2 DEFAULT GETDATE()
);

-- Create dedicated application user for this schema
CREATE USER [user_abc123_app] FOR LOGIN [freemium_app];

-- Grant permissions only to this schema
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::[user_abc123] TO [user_abc123_app];
```

### Application Query Pattern

```sql
-- Application uses schema-qualified queries
-- Connection string contains tenant_schema context

-- Example: Fetch deals for tenant
SELECT * FROM [user_abc123].[DealRegistrations]
WHERE status = 'active';

-- Example: Create new deal
INSERT INTO [user_abc123].[DealRegistrations]
(deal_name, aws_role_arn, product_id, customer_name, deal_value)
VALUES
('Deal ABC', 'arn:aws:iam::123456789012:role/Publisher', 'prod-123', 'Acme Corp', 50000.00);
```

### Migration on Upgrade

```sql
-- Step 1: Export schema data
-- Use SQL Server Management Studio or bcp utility
bcp [freemium_db].[user_abc123].[DealRegistrations] out deals.dat -N -S server -U user -P pass

-- Step 2: Create new dedicated RDS instance (via AWS API/Terraform)
-- New instance: customer-abc123-db.xxxxx.us-east-1.rds.amazonaws.com

-- Step 3: Import to new database
bcp [customer_db].[dbo].[DealRegistrations] in deals.dat -N -S new-server -U user -P pass

-- Step 4: Update Management Portal registry
UPDATE CustomerRegistry
SET db_host = 'customer-abc123-db.xxxxx.us-east-1.rds.amazonaws.com',
    db_name = 'customer_db',
    tier = 'paid',
    migrated_at = GETDATE()
WHERE tenant_id = 'user_abc123';

-- Step 5: Clean up freemium DB after verification (wait 30 days)
DROP SCHEMA [user_abc123] CASCADE;
DROP USER [user_abc123_app];
```

## Pros

✅ **Cost effective**: 1 RDS for 1000+ users vs 1000+ RDS instances
✅ **True isolation**: SQL Server schemas prevent cross-tenant data leaks
✅ **Simple migration**: Schema export/import is native SQL Server operation
✅ **Scalable**: SQL Server handles 1000s of schemas efficiently
✅ **Perception**: Can market as "dedicated schema" (sounds isolated)
✅ **Rollback friendly**: Keep schema X days after upgrade for verification
✅ **Standard SQL pattern**: Well-understood approach, easy to maintain
✅ **Backup simplicity**: Single backup covers all freemium users

## Cons

❌ **Shared infrastructure risk**: All freemium users on 1 DB instance
   - Noisy neighbor: One tenant's heavy queries affect all others
   - DDoS vulnerability: Attack on one tenant impacts entire freemium tier
   - Single point of failure

❌ **Perception problem**: "Shared database" may scare enterprise customers
   - Despite schema isolation, security teams may reject
   - Competitors knowing they share infrastructure creates trust issues

❌ **Single point of failure**: If freemium DB goes down, all freemium users affected

❌ **Scaling limits**: Eventually hit SQL Server practical limits
   - ~32,000 schema limit (theoretical)
   - Performance degradation with 5000+ schemas
   - Query plan cache bloat

❌ **Backup/restore complexity**: Can't restore single tenant easily
   - Must restore entire DB to recover one tenant's data
   - Point-in-time recovery affects all tenants

❌ **Schema management overhead**: Need automated schema provisioning
   - Schema naming conventions
   - Permission management
   - Schema lifecycle tracking

## Cost Analysis

### Freemium Tier (1000 users)
- **RDS SQL Server Web Edition** (db.t3.medium): ~$100/month
  - 2 vCPU, 4GB RAM
  - 100GB storage
  - Sufficient for 1000 light users
- **Cost per user**: $0.10/month

### Paid Tier (per customer)
- **RDS SQL Server Web Edition** (db.t3.small): ~$50/month
  - 2 vCPU, 2GB RAM
  - 50GB storage
  - Dedicated instance

### Savings Calculation
- **Without pooling**: 1000 users × $50 = $50,000/month
- **With pooling**: 1 instance × $100 = $100/month
- **Monthly savings**: $49,900
- **Annual savings**: $598,800

### Scaling Costs
- **2000 users**: Upgrade to db.t3.large (~$200/month) = $0.10/user
- **5000 users**: Upgrade to db.r5.large (~$500/month) = $0.10/user
- **10,000 users**: Consider splitting to 2 instances (~$1,000/month) = $0.10/user

## Security Considerations

### Strengths
- **Namespace isolation**: SQL Server schemas provide logical separation
- **Permission-based access**: Each app user can only access own schema
- **Audit logging**: Can track all cross-schema access attempts
- **SQL injection protection**: Schema-qualified queries prevent cross-tenant attacks

### Weaknesses
- **DBA access**: Database admin can access all schemas
- **Shared resources**: CPU/memory/disk contention
- **Metadata exposure**: Can see schema names, sizes, query patterns
- **Compliance gaps**: May not meet SOC2/HIPAA requirements for multi-tenant

### Mitigations
- Document that freemium is "evaluation only, not for production workloads"
- Implement strict DBA access controls and audit logging
- Use transparent data encryption (TDE) for data at rest
- Network isolation using VPC security groups
- Rate limiting per tenant to prevent abuse

## Implementation Timeline

### Phase 1: MVP (2-3 weeks)
- [ ] Design schema template for Deal Registration
- [ ] Build schema provisioning automation
- [ ] Implement signup flow with CloudFormation runner
- [ ] Create schema on successful CFN deployment
- [ ] Store tenant → schema mapping in Management Portal
- [ ] Update freemium webapp to route queries to correct schema

### Phase 2: Migration Flow (1-2 weeks)
- [ ] Build upgrade trigger in Management Portal
- [ ] Automate RDS provisioning via AWS API/Terraform
- [ ] Implement schema export/import pipeline
- [ ] Create DNS update automation
- [ ] Build tenant registry update logic
- [ ] Add schema cleanup job (30-day delay)

### Phase 3: Monitoring & Operations (1 week)
- [ ] Set up CloudWatch metrics per schema (query count, data size)
- [ ] Implement alerting for shared resource contention
- [ ] Create operational runbook for schema management
- [ ] Build admin dashboard showing schema stats

## Operational Runbook

### Common Tasks

**Add new freemium user:**
```bash
# 1. User completes signup
# 2. Run CloudFormation in their AWS account
# 3. Capture IAM role ARN
# 4. Provision schema:
sqlcmd -S freemium-db -i provision_schema.sql -v tenant_id="user_abc123"
```

**Upgrade user to paid:**
```bash
# 1. Trigger upgrade from Management Portal
# 2. Provision new RDS instance
terraform apply -var="tenant_id=user_abc123"

# 3. Export schema
./scripts/export_schema.sh user_abc123 freemium-db

# 4. Import to new DB
./scripts/import_schema.sh user_abc123 customer-abc123-db

# 5. Update registry and DNS
./scripts/cutover_tenant.sh user_abc123

# 6. Schedule cleanup
./scripts/schedule_cleanup.sh user_abc123 30
```

**Monitor freemium DB health:**
```sql
-- Check schema count
SELECT COUNT(*) FROM sys.schemas WHERE name LIKE 'user_%';

-- Check top resource consumers
SELECT TOP 10
    SCHEMA_NAME(t.schema_id) as schema_name,
    SUM(p.rows) as row_count,
    SUM(a.total_pages) * 8 / 1024 as size_mb
FROM sys.tables t
INNER JOIN sys.partitions p ON t.object_id = p.object_id
INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
WHERE SCHEMA_NAME(t.schema_id) LIKE 'user_%'
GROUP BY SCHEMA_NAME(t.schema_id)
ORDER BY size_mb DESC;
```

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| DB outage affects all freemium | Medium | High | Multi-AZ deployment, automated failover |
| Competitor data concerns | High | High | Clear messaging: "evaluation only" |
| Performance degradation at scale | Medium | Medium | Monitoring, upgrade instance size proactively |
| Schema limit reached | Low | Medium | Split to multiple DBs at 3000 schemas |
| Migration data loss | Low | Critical | Test migration flow, keep source 30 days |
| Security breach via shared DB | Low | Critical | Encryption, audit logging, minimal DBA access |

## Success Metrics

- **Cost efficiency**: <$0.15 per freemium user per month
- **Signup time**: <5 minutes from registration to usable portal
- **Migration time**: <15 minutes downtime during upgrade
- **Reliability**: 99.5% uptime for freemium tier
- **Conversion rate**: Track freemium → paid conversion
- **Data integrity**: Zero data loss during migrations

## Recommendation

**Use this solution if:**
- Cost optimization is highest priority
- Freemium is positioned as "evaluation/testing only"
- Can accept shared infrastructure trade-offs
- Have operational expertise with SQL Server schemas

**Avoid this solution if:**
- Enterprise customers require physical isolation
- Compliance needs (HIPAA/SOC2) for freemium tier
- Competitors sharing DB creates trust issues
- Need to guarantee SLAs for free tier
