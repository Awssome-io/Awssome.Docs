# Freemium Solution 1: Dedicated Freemium DB with Schema Isolation

## Overview

Single RDS SQL Server instance serving all freemium users with dedicated schemas per tenant. Simple migration path to dedicated infrastructure on upgrade.

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
│                    MANAGEMENT PORTAL                            │
│           (mgmt.awssome.io - Central DB + WebApp)               │
│  • Customer metadata, AWS roles, tenant registry                │
│  • Tenant tier tracking (free vs paid)                          │
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
│ + HubSpot Sync UI    │  │ + HubSpot Sync   │  │ + HubSpot Sync   │
└──────────┬───────────┘  └────────┬─────────┘  └────────┬─────────┘
           │                       │                     │
           ▼                       ▼                     ▼
┌──────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ FREEMIUM RDS SQL     │  │ DEDICATED RDS    │  │ DEDICATED RDS    │
│ Server DB            │  │ SQL Server       │  │ SQL Server       │
├──────────────────────┤  │                  │  │                  │
│ Schema: user_1       │  │ All features     │  │ All features     │
│  ├─ DealReg tables   │  │ + DealReg        │  │ + DealReg        │
│  ├─ HubSpotTokens    │  │                  │  │                  │
│  ├─ HubSpotMappings  │  │                  │  │                  │
│ Schema: user_2       │  └──────────────────┘  └──────────────────┘
│  ├─ DealReg tables   │
│  ├─ HubSpotTokens    │
│ Schema: user_N       │
│  ├─ DealReg tables   │
│  ├─ HubSpotTokens    │
│ (1000+ schemas)      │
│                      │
│ Tier-aware sync:     │
│ • Free: 10 deals max │
│ • Paid: Unlimited    │
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

Each freemium user receives:
- **Dedicated schema** in shared SQL Server database (e.g., `user_abc123`)
- **Deal Registration tables** for storing AWS Marketplace deals
- **HubSpot integration tables**:
  - OAuth tokens (access/refresh tokens, HubSpot portal ID)
  - Deal mappings (links HubSpot deals to Awssome deal registrations)
  - Sync history (tracks import success/failures)
- **Schema-level permissions** ensuring user can only access their own data

### Data Access Pattern

- Application uses **schema-qualified queries** (e.g., `SELECT * FROM [user_abc123].[DealRegistrations]`)
- Connection string includes tenant schema context
- Database enforces namespace isolation via SQL Server schema permissions
- No cross-schema access possible without explicit permissions

### Migration on Upgrade

**Process:**
1. **Export schema** - Use native SQL Server tools (bcp utility or SSMS) to extract all tenant data
2. **Provision dedicated RDS** - Create new SQL Server instance via AWS API/Terraform
3. **Import data** - Restore tenant data to new dedicated database
4. **Update registry** - Management Portal points to new DB host
5. **Cleanup** - Delete schema from freemium DB after 30-day retention period

**Benefits:**
- Schema export/import is native SQL Server operation (simple, reliable)
- HubSpot OAuth tokens automatically migrate (no app reinstall)
- Zero downtime possible with dual-run strategy
- Rollback available during retention period

## HubSpot Integration

### Overview

Single HubSpot app for all tiers (freemium & paid). Tier-aware backend applies deal limits based on tenant subscription.

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

### OAuth Flow

**Process:**
1. User clicks "Connect HubSpot" in Awssome portal
2. Redirected to HubSpot OAuth authorization page
3. User approves connection (grants `crm.objects.deals.read` permission)
4. HubSpot redirects back with authorization code
5. Backend exchanges code for access/refresh tokens
6. Tokens stored in tenant's schema (HubSpotTokens table)
7. User redirected back to Awssome with "Connected" status

**Key Benefits:**
- OAuth app installs on **customer's HubSpot account** (uses their API quota)
- Single app for all tiers (freemium & paid)
- No app reinstall needed when upgrading

### Tier-Aware Deal Sync

**Sync Process:**
1. User clicks "Sync Deals from HubSpot" button (manual trigger)
2. Backend checks tenant tier from Management Portal
3. **Free tier**: Fetch maximum 10 deals
4. **Paid tier**: Fetch unlimited deals
5. Use HubSpot Batch API (100 deals per request)
6. Import deals as Awssome Deal Registrations
7. Create mappings (links HubSpot deal ID → Awssome deal ID)
8. Log sync history (success/failure, deal counts)

**Rate Limiting:**
- Respects customer's HubSpot API limits (not yours)
- Batch requests reduce API call count
- Manual trigger = low API usage (no polling/webhooks)

### User Experience

**Connection Flow:**
- "Connect HubSpot" button in settings
- OAuth consent screen (HubSpot-hosted)
- "✓ HubSpot Connected" confirmation
- Display HubSpot portal name/ID

**Sync Flow:**
- "Sync Deals" button (disabled while syncing)
- Progress indicator during import
- Success message: "Imported X of Y deals"
- Free tier shows "(Free tier: 10 deals max)" warning

### Migration with HubSpot Data

**Upgrade Process:**
1. Export tenant's schema (includes all HubSpot tables)
2. Import to new dedicated database
3. HubSpot OAuth tokens automatically migrate
4. No app reinstall required
5. Sync resumes with unlimited deal limit (paid tier)

### HubSpot Integration Benefits

✅ **Single OAuth flow**: Free & paid users install same HubSpot app
✅ **No reinstall on upgrade**: Tokens migrate with data
✅ **Customer's rate limits**: Uses their HubSpot API quota
✅ **Tier-aware**: Backend enforces 10-deal limit for free users
✅ **Manual trigger**: Low API usage, no webhook complexity
✅ **Simple architecture**: One codebase, one marketplace listing

## Pros

✅ **Cost effective**: 1 RDS for 1000+ users vs 1000+ RDS instances
✅ **True isolation**: SQL Server schemas prevent cross-tenant data leaks
✅ **Simple migration**: Schema export/import is native SQL Server operation
✅ **Scalable**: SQL Server handles 1000s of schemas efficiently
✅ **Perception**: Can market as "dedicated schema" (sounds isolated)
✅ **Rollback friendly**: Keep schema X days after upgrade for verification
✅ **Standard SQL pattern**: Well-understood approach, easy to maintain
✅ **Backup simplicity**: Single backup covers all freemium users
✅ **HubSpot integration**: Single OAuth app for all tiers, no reinstall on upgrade
✅ **Tier-aware sync**: Automatically enforce 10-deal limit for free users

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

### Phase 1: Core Freemium MVP (2-3 weeks)
- [ ] Design schema template for Deal Registration
- [ ] Build schema provisioning automation
- [ ] Implement signup flow with CloudFormation runner
- [ ] Create schema on successful CFN deployment
- [ ] Store tenant → schema mapping in Management Portal
- [ ] Update freemium webapp to route queries to correct schema

### Phase 2: HubSpot Integration (2-3 weeks)
- [ ] Create HubSpot app in developer portal (OAuth scopes: crm.objects.deals.read)
- [ ] Implement OAuth flow (authorization, callback, token storage)
- [ ] Add HubSpot tables to schema template (tokens, mappings, sync history)
- [ ] Build tier-aware sync service (10 deals for free, unlimited for paid)
- [ ] Implement rate limiting and batch API calls
- [ ] Create "Connect HubSpot" UI in Awssome portal
- [ ] Build manual sync trigger with progress indicator
- [ ] Test OAuth flow for freemium & paid tenants

### Phase 3: Migration Flow (1-2 weeks)
- [ ] Build upgrade trigger in Management Portal
- [ ] Automate RDS provisioning via AWS API/Terraform
- [ ] Implement schema export/import pipeline (include HubSpot tables)
- [ ] Create DNS update automation
- [ ] Build tenant registry update logic
- [ ] Add schema cleanup job (30-day delay)
- [ ] Test HubSpot integration continues working post-migration

### Phase 4: Monitoring & Operations (1 week)
- [ ] Set up CloudWatch metrics per schema (query count, data size)
- [ ] Implement alerting for shared resource contention
- [ ] Create operational runbook for schema management
- [ ] Build admin dashboard showing schema stats
- [ ] Add HubSpot sync monitoring (success rate, API usage)
- [ ] Set up alerts for OAuth token expiration

## Operational Runbook

### Common Tasks

**Add new freemium user:**
1. User completes signup on freemium.awssome.io
2. Run CloudFormation template in their AWS account
3. Capture IAM role ARN for AWS Marketplace access
4. Automatically provision schema using SQL scripts
5. Store tenant → schema mapping in Management Portal
6. User can now access their portal and create deals

**Upgrade user to paid:**
1. Trigger upgrade from Management Portal
2. Provision new dedicated RDS instance (Terraform/AWS API)
3. Export tenant's schema using native SQL Server tools
4. Import data to new dedicated database
5. Update Management Portal registry (point to new DB)
6. Update DNS (userX.awssome.io → new infrastructure)
7. Schedule schema cleanup after 30-day retention

**Monitor freemium DB health:**
- **Schema count**: Track total freemium users
- **Top resource consumers**: Identify heavy users (row counts, storage)
- **Query performance**: Monitor slow queries per schema
- **HubSpot sync metrics**: Success rate, API usage per tenant
- **Storage growth**: Forecast when to upgrade instance size

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
