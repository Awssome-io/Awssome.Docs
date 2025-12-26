# AWS Partner Central API Reference (English)

API reference for Partner Central Selling and Account SDKs. This document categorizes and describes all endpoints at the business level.

> 📘 **Vietnamese version**: [aws-partner-central-api-reference.md](./aws-partner-central-api-reference.md)

---

## Overview

AWS Partner Central provides 2 main SDKs:

| SDK | Purpose | API Count |
|-----|---------|-----------|
| **Partner Central Selling** | Manage sales opportunities, engagements, and co-sell with AWS | 42 |
| **Partner Central Account** | Manage partner profile, verification, and connections with other partners | 29 |

---

## Partner Central Selling API

SDK for partners to manage sales pipeline and co-sell collaboration with AWS.

---

### 1. Opportunity Management

**Purpose:** Manage the entire lifecycle of sales opportunities - from identifying prospects to closing deals.

**Benefits:**
- 🎯 **Higher win rate**: Co-selling with AWS can increase deal win probability by 2-3x
- 📊 **Visibility**: AWS sales team can see and support your deals
- 🔄 **Data sync**: Synchronize pipeline between internal CRM and AWS Partner Central
- 💰 **Access funding**: Qualifying opportunities may receive AWS funding/incentives

| API | Description |
|-----|-------------|
| `ListOpportunities` | Retrieve opportunities with filters by customer, stage, date |
| `GetOpportunity` | Get details of a specific opportunity |
| `CreateOpportunity` | Create new opportunity from partner side |
| `UpdateOpportunity` | Update opportunity information (stage, customer info, etc.) |
| `SubmitOpportunity` | Submit opportunity for AWS review to start co-sell |
| `AssignOpportunity` | Assign opportunity to a specific sales rep |
| `AssociateOpportunity` | Link opportunity with Solutions/AwsProducts/Offers |
| `DisassociateOpportunity` | Unlink opportunity from entities |
| `GetAwsOpportunitySummary` | View AWS-side summary of an opportunity |

> **📖 Glossary:**
> - **Opportunity**: Sales opportunity - a potential deal with a specific customer
> - **Stage**: Phase in sales pipeline (Prospect → Qualified → Committed → Closed)
> - **Co-sell**: Collaborative selling model - Partner and AWS sales work together on deals
> - **Involvement Type**: Level of AWS involvement (Co-Sell = need support, For Visibility Only = notification only)
> - **Submit**: Send opportunity for AWS review to begin co-sell process

**Opportunity Lifecycle Stages:**
```
Prospect → Qualified → Technical Validation → Business Validation → Committed → Launched → Closed (Won/Lost)
```

---

### 2. Solution Management

**Purpose:** Manage partner solutions that have been validated and published by AWS.

**Benefits:**
- 🏆 **Credibility**: AWS-validated solutions increase customer trust
- 🔗 **Link to Opportunities**: Connect solutions to opportunities to demonstrate expertise
- 📈 **Marketplace visibility**: Solutions can appear in AWS Marketplace

| API | Description |
|-----|-------------|
| `ListSolutions` | Retrieve list of registered solutions |

> **📖 Glossary:**
> - **Solution**: Partner technology solution that has been reviewed and approved by AWS
> - **AWS Competency**: Certification from AWS for expertise in a specific domain

---

### 3. Engagement Management

**Purpose:** Create and manage collaborative workspace between partner, AWS, and potentially other partners.

**Benefits:**
- 🤝 **Collaboration hub**: Single place for all stakeholders to work together
- 📁 **Centralized info**: Consolidate deal, customer, and project information
- 👥 **Multi-party**: Support multiple partners working on the same deal

| API | Description |
|-----|-------------|
| `ListEngagements` | Retrieve list of engagements |
| `GetEngagement` | Get engagement details |
| `CreateEngagement` | Create new engagement (collaboration workspace) |
| `ListEngagementMembers` | View members participating in engagement |
| `ListEngagementResourceAssociations` | View resources linked to engagement |

> **📖 Glossary:**
> - **Engagement**: Virtual workspace for collaboration - where partner and AWS work together on deals
> - **Member**: Participant in engagement (AWS account or partner account)
> - **Resource Association**: Link between engagement and resources (opportunity, solution)

---

### 4. Engagement Context

**Purpose:** Add and manage detailed customer project information within engagement.

**Benefits:**
- 📋 **Rich context**: AWS sales has full context to provide effective support
- 🎯 **Better matching**: AWS can match the right resources/experts for the deal
- 📝 **Documentation**: Store project information for audit and tracking

| API | Description |
|-----|-------------|
| `CreateEngagementContext` | Add context (customer project info) to engagement |
| `UpdateEngagementContext` | Update context data |

> **📖 Glossary:**
> - **Context**: Additional information about customer project (industry, business problem, timeline)
> - **Customer Project**: Specific customer project that the deal is targeting

---

### 5. Engagement Invitations

**Purpose:** Invite stakeholders into engagement for collaboration.

**Benefits:**
- 🚀 **Quick onboarding**: Rapidly bring AWS or other partners into the deal
- 🔐 **Controlled access**: Control who can participate in engagement
- 📧 **Formal process**: Clear invitation process with tracking

| API | Description |
|-----|-------------|
| `ListEngagementInvitations` | Retrieve list of invitations (sent/received) |
| `GetEngagementInvitation` | View invitation details |
| `CreateEngagementInvitation` | Invite partner/AWS account to engagement |
| `AcceptEngagementInvitation` | Accept invitation |
| `RejectEngagementInvitation` | Reject invitation |

> **📖 Glossary:**
> - **Invitation**: Request to join an engagement
> - **Sender**: Person sending the invitation
> - **Receiver**: Person receiving invitation (AWS account ID or partner)

---

### 6. Resource Snapshots

**Purpose:** Create immutable (frozen) copies of resources at a specific point in time.

**Benefits:**
- 📸 **Point-in-time record**: Save deal state at important moments
- 🔍 **Audit trail**: Evidence of deal state for audit/compliance
- 📊 **Historical tracking**: Track deal evolution over time
- 🤝 **Share state**: Consistently share deal state with AWS/partners

| API | Description |
|-----|-------------|
| `ListResourceSnapshots` | Retrieve list of snapshots in engagement |
| `GetResourceSnapshot` | View snapshot details |
| `CreateResourceSnapshot` | Create snapshot of opportunity at current point in time |

> **📖 Glossary:**
> - **Snapshot**: Data copy at a point in time, immutable after creation
> - **Revision**: Version number of snapshot (increments with each new creation)
> - **Immutable**: Cannot be changed - ensures data integrity

---

### 7. Resource Snapshot Jobs

**Purpose:** Automate snapshot creation based on schedule or triggers.

**Benefits:**
- ⏰ **Automation**: Automatically capture state without manual intervention
- 📈 **Consistent tracking**: Ensure snapshots are created regularly
- 🔄 **Continuous sync**: Keep AWS and partners updated with latest data

| API | Description |
|-----|-------------|
| `ListResourceSnapshotJobs` | Retrieve list of jobs |
| `GetResourceSnapshotJob` | View job details |
| `CreateResourceSnapshotJob` | Create auto-snapshot job |
| `StartResourceSnapshotJob` | Start job |
| `StopResourceSnapshotJob` | Stop job |
| `DeleteResourceSnapshotJob` | Delete job |

> **📖 Glossary:**
> - **Job**: Background task that performs work on a schedule
> - **Schedule**: Job execution schedule (e.g., daily, weekly)
> - **IAM Role**: AWS role that gives job permission to perform operations

---

### 8. Async Task Operations

**Purpose:** Execute multiple complex operations in a single API call.

**Benefits:**
- 🚀 **Simplified workflow**: One API call instead of 5-6 separate calls
- ⚡ **Faster execution**: AWS orchestrates steps optimally
- 🛡️ **Error handling**: AWS handles errors and rollback if needed
- 📦 **Atomic operations**: Ensures all steps complete or none do

| API | Description |
|-----|-------------|
| `ListEngagementByAcceptingInvitationTasks` | List tasks for accepting invitations |
| `StartEngagementByAcceptingInvitationTask` | Accept invitation and join engagement |
| `ListEngagementFromOpportunityTasks` | List tasks creating engagement from opportunity |
| `StartEngagementFromOpportunityTask` | **Create full engagement workflow from opportunity** |
| `ListOpportunityFromEngagementTasks` | List tasks creating opportunity from engagement |
| `StartOpportunityFromEngagementTask` | Create opportunity from engagement |

> **📖 Glossary:**
> - **Task**: Unit of work running asynchronously (non-blocking)
> - **Orchestration**: Coordinating multiple operations in correct sequence
> - **Async**: Asynchronous - API returns immediately, task runs in background
> - **ClientToken**: Token ensuring idempotency (multiple calls create only 1 task)

**StartEngagementFromOpportunityTask automatically executes:**
```
GetOpportunity → CreateEngagement → CreateResourceSnapshot → CreateResourceSnapshotJob → CreateEngagementInvitation → SubmitOpportunity
```

---

### 9. Selling System Settings

**Purpose:** Configure system-level settings for selling operations.

**Benefits:**
- ⚙️ **Centralized config**: Single place to manage all settings
- 🔐 **IAM integration**: Configure roles for automated operations

| API | Description |
|-----|-------------|
| `GetSellingSystemSettings` | Get settings (IAM role for snapshot jobs) |
| `PutSellingSystemSettings` | Update settings |

> **📖 Glossary:**
> - **System Settings**: Configuration applying to entire account
> - **IAM Role ARN**: Amazon Resource Name of IAM role

---

### 10. Resource Tagging (Selling)

**Purpose:** Attach tags (labels) to resources for organization and tracking.

**Benefits:**
- 🏷️ **Organization**: Categorize resources by project, team, customer
- 💰 **Cost allocation**: Track costs by tags
- 🔍 **Filtering**: Easily filter resources by tags

| API | Description |
|-----|-------------|
| `ListTagsForResource` | Get resource tags |
| `TagResource` | Add tags |
| `UntagResource` | Remove tags |

> **📖 Glossary:**
> - **Tag**: Key-value pair attached to resource (e.g., "Environment": "Production")
> - **ARN**: Amazon Resource Name - unique identifier of resource in AWS

---

## Partner Central Account API

SDK for partners to manage account, profile, and connections with other partners.

---

### 1. Profile & Visibility

**Purpose:** Manage profile information and visibility level of partner to AWS and other partners.

**Benefits:**
- 🌟 **Discoverability**: Visible profile helps AWS and customers find you
- 🎯 **Targeted leads**: AWS can refer deals matching your expertise
- 🔒 **Privacy control**: Control who sees what information about your company

| API | Description |
|-----|-------------|
| `GetProfileVisibility` | View current visibility level |
| `PutProfileVisibility` | Set visibility (Full/Limited) |
| `StartProfileUpdateTask` | Start profile update task |
| `GetProfileUpdateTask` | View update task status |
| `CancelProfileUpdateTask` | Cancel update task |

> **📖 Glossary:**
> - **Profile**: Public information about partner (company info, competencies, solutions)
> - **Visibility**: Visibility level - Full (complete) or Limited (restricted)
> - **Profile Update Task**: Async task to update profile information

---

### 2. Verification

**Purpose:** Verify partner identity and legitimacy with AWS.

**Benefits:**
- ✅ **Trust**: Verified partners are more trusted by AWS and customers
- 🔓 **Access features**: Some features only available to verified partners
- 🏆 **Program eligibility**: Verification required to join partner programs

| API | Description |
|-----|-------------|
| `GetVerification` | View verification status |
| `StartVerification` | Begin verification process |
| `SendEmailVerificationCode` | Send email verification code |

> **📖 Glossary:**
> - **Registrant Verification**: Verify the registrant (individual representing company)
> - **Business Verification**: Verify the business (legitimate company)
> - **Verification Code**: OTP code sent via email for confirmation

---

### 3. Partner Management

**Purpose:** Manage partner entities within AWS Partner Network ecosystem.

**Benefits:**
- 📋 **Centralized records**: Manage all partner info in one place
- 🔄 **Programmatic access**: Integrate with other systems via API

| API | Description |
|-----|-------------|
| `GetPartner` | Get partner information |
| `CreatePartner` | Create new partner |
| `ListPartners` | Retrieve list of partners |

> **📖 Glossary:**
> - **Partner**: Company registered to participate in AWS Partner Network
> - **AWS Partner Network (APN)**: Official AWS partner program

---

### 4. Connection Management

**Purpose:** Manage connections (formal relationships) with other partners.

**Benefits:**
- 🤝 **Formal partnerships**: Establish official relationships with other partners
- 🔄 **Deal sharing**: Can share deals and co-sell with connected partners
- 📊 **Network effect**: Expand reach through partner network

| API | Description |
|-----|-------------|
| `ListConnections` | Retrieve list of connections |
| `GetConnection` | View connection details |
| `CancelConnection` | Cancel connection |
| `GetConnectionPreferences` | Get preferences settings |
| `UpdateConnectionPreferences` | Update preferences |

> **📖 Glossary:**
> - **Connection**: Formal relationship between two partners
> - **COSELL Connection**: Connection for co-selling deals together
> - **PARTNER Connection**: General partner relationship connection
> - **Preferences**: Settings for how to receive and handle connections

---

### 5. Connection Invitations

**Purpose:** Send and manage connection invitations with other partners.

**Benefits:**
- 📧 **Formal process**: Official invitation process with tracking
- 🔐 **Consent-based**: Both parties must agree to establish connection
- 📊 **Audit trail**: Invitation history for review

| API | Description |
|-----|-------------|
| `ListConnectionInvitations` | Retrieve list of invitations |
| `CreateConnectionInvitation` | Invite partner to connect |
| `GetConnectionInvitation` | View invitation details |
| `AcceptConnectionInvitation` | Accept connection |
| `RejectConnectionInvitation` | Reject connection |
| `CancelConnectionInvitation` | Cancel sent invitation |

> **📖 Glossary:**
> - **Invitation**: Request to establish connection
> - **Receiver Identifier**: AWS Account ID of invited partner

---

### 6. Alliance & Training

**Purpose:** Manage alliance lead contact and link with AWS Training/Certification.

**Benefits:**
- 👤 **Clear contact**: AWS knows who is the primary contact for alliance matters
- 🎓 **Training benefits**: Link email domain to receive training benefits
- 📜 **Certification tracking**: Track AWS certifications of employees

| API | Description |
|-----|-------------|
| `GetAllianceLeadContact` | Get alliance lead information |
| `PutAllianceLeadContact` | Update alliance lead contact |
| `AssociateAwsTrainingCertificationEmailDomain` | Link email domain for AWS Training |
| `DisassociateAwsTrainingCertificationEmailDomain` | Unlink email domain |

> **📖 Glossary:**
> - **Alliance Lead**: Primary representative of partner for AWS relationship
> - **Email Domain**: Company email domain (e.g., @yourcompany.com)
> - **AWS Training & Certification**: AWS training and certification program

---

### 7. Resource Tagging (Account)

**Purpose:** Attach tags to account resources for organization and tracking.

**Benefits:**
- 🏷️ **Organization**: Categorize resources by purpose
- 📊 **Reporting**: Filter and report by tags

| API | Description |
|-----|-------------|
| `ListTagsForResource` | Get tags |
| `TagResource` | Add tags |
| `UntagResource` | Remove tags |

---

## Common Workflows

### Workflow 1: Co-Sell with AWS (End-to-End)

Complete process to bring a deal into co-sell with AWS:

```
1. CreateOpportunity
   └── Create opportunity with customer info, stage = Prospect

2. UpdateOpportunity (multiple times)
   └── Update information as deal progresses through stages

3. AssociateOpportunity
   └── Link with Solutions/AwsProducts if applicable

4. SubmitOpportunity (InvolvementType: Co-Sell)
   └── Submit for AWS review
   └── Opportunity transitions to under review status

5. [AWS Reviews] → Approved or Action Required

6. If Approved:
   └── AWS sales will engage and support the deal
```

### Workflow 2: Create Engagement from Opportunity (Automated)

```
StartEngagementFromOpportunityTask
└── Automatically executes:
    ├── GetOpportunity
    ├── CreateEngagement
    ├── CreateResourceSnapshot
    ├── CreateResourceSnapshotJob
    ├── CreateEngagementInvitation (invite AWS)
    └── SubmitOpportunity
```

### Workflow 3: Partner-to-Partner Connection

```
1. CreateConnectionInvitation
   └── Send invitation with ConnectionType (COSELL/PARTNER)

2. [Partner B receives invitation]

3. AcceptConnectionInvitation (Partner B)
   └── Connection established

4. ListConnections
   └── Verify existing connections
```

### Workflow 4: Sync Opportunities (Automation)

```
1. Store LastModifiedDate from previous sync

2. ListOpportunities
   └── Filter: AfterLastModifiedDate = LastModifiedDate
   └── Sort: LastModifiedDate Descending

3. Iterate with NextToken for pagination

4. Process each opportunity:
   └── GetOpportunity for full details
   └── Sync to local database

5. Update LastModifiedDate = newest modified date
```

### Workflow 5: Account Setup and Verification

```
1. CreatePartner (if not exists)

2. SendEmailVerificationCode
   └── Send code to email

3. StartVerification
   └── Begin verification with code

4. GetVerification
   └── Check status (REGISTRANT/BUSINESS)

5. PutProfileVisibility
   └── Set visibility level for profile

6. PutAllianceLeadContact
   └── Set alliance lead contact info
```

### Workflow 6: Engagement Collaboration

```
1. CreateEngagement
   └── Create engagement workspace

2. CreateEngagementContext
   └── Add customer project info

3. CreateEngagementInvitation
   └── Invite partners to engagement

4. [Partners Accept]

5. CreateResourceSnapshot
   └── Snapshot opportunity state

6. ListEngagementMembers
   └── View participants

7. UpdateEngagementContext
   └── Update progress
```

---

## Environment Catalogs

| Catalog | Description |
|---------|-------------|
| `AWS` | Production environment - real opportunities |
| `Sandbox` | Testing environment - safe to test |

---

## Error Handling

| Error | Description |
|-------|-------------|
| `ResourceNotFoundException` | Resource does not exist |
| `ValidationException` | Invalid input |
| `ConflictException` | Conflict state (e.g., opportunity under review) |
| `ThrottlingException` | Rate limit exceeded |
| `AccessDeniedException` | Permission denied |

---

## Quick Reference

- **Opportunity ID format**: `O` + 1-19 digits
- **ClientToken**: Used to ensure idempotency for create operations
- **Pagination**: Use `NextToken` to iterate through multiple pages
- **Catalog**: `AWS` (production) or `Sandbox` (testing)
