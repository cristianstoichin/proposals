# NFC Emergency Profile SaaS - Backend Development Proposal

**Prepared by:** Cristian Stoichin

**Date:** December 1, 2025

**Project:** Multi-Role Dashboard Backend with NFC-Linked Emergency Profiles

---

## Why I'm the Right Fit for This Project

**Live Multi-Tenant SaaS Platforms - See My Work:**

**1. Simple Booking SaaS Platform (Multi-Role Dashboard):**
- **Client Store:** https://finance-dev.simple-booking.org/services
- **Admin Dashboard:** https://admin-dev.simple-booking.org/calendar
- **Already Built:** Role-based access control, user management, CRUD operations, audit logging, Prisma ORM, PostgreSQL

**2. Document Processing Platform:**
- **Live Platform:** https://admin.short-ly.net/login
- **Demo Credentials:** demo.user@short-ly.net / Pass123$
- **Already Built:** JWT Authentication, S3 file storage, Lambda backend, API Gateway, DynamoDB

**What This Means for Your Project:** I've built multi-tenant SaaS platforms with role-based permissions from scratch. The core patterns you need (RBAC, CRUD, audit logs, secure data separation) are already in my toolkit. I understand the difference between public-facing endpoints and authenticated internal routes - exactly what your NFC tag to Emergency Profile flow requires.

---

## I Understand Your Architecture

Your system has two distinct access patterns that need to be kept separate:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AUTHENTICATED DASHBOARD                          │
│  (Owner, Supervisor, Employee, Master Admin)                             │
├─────────────────────────────────────────────────────────────────────────┤
│  Worker Management │ Certifications │ Compliance │ Incident Reports     │
│  Full CRUD         │ Full CRUD      │ Audit Logs │ Internal Data        │
└────────────────────────────────────────┬────────────────────────────────┘
                                         │
                                         │ Workers linked via Organization
                                         │
┌────────────────────────────────────────┴────────────────────────────────┐
│                      WORKER'S EMERGENCY PROFILE                          │
│  (Controlled by Worker - Voluntary fields)                               │
├─────────────────────────────────────────────────────────────────────────┤
│  Medical Info │ Emergency Contacts │ Allergies │ Blood Type │ DNR       │
│  Worker decides what to share - First responders only see this          │
└────────────────────────────────────────┬────────────────────────────────┘
                                         │
                                         │ public_key=KCozyjQn2XDVuyKHzfeGe6d6
                                         │
┌────────────────────────────────────────┴────────────────────────────────┐
│                         NFC TAG (Physical)                               │
├─────────────────────────────────────────────────────────────────────────┤
│  Pre-encoded URL: yourplatform.com/e/{public_key}                        │
│  Never rewritten - Virtual updates only                                  │
│  Tap → Instant load → No app, no login                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Insight:** The NFC tag is a one-time bridge. Once linked, all updates happen in your database. The public_key in the URL is cryptographically random and only exposes emergency-designated fields. This is smart design - I'll build the backend to enforce this separation at every layer.

---

## My Approach to Your Requirements

### Authentication & Authorization

**Current Auth.js Integration:**
- I'll work with your existing Auth.js setup
- Extend session to include role and organization context
- JWT tokens will carry role claims for API authorization

**RBAC Implementation:**

```typescript
// Role hierarchy with permission inheritance
const ROLES = {
  MASTER_ADMIN: ['*'],  // Full system access
  OWNER: ['org:*', 'workers:*', 'reports:*', 'tags:*'],
  SUPERVISOR: ['org:read', 'workers:read', 'workers:update', 'reports:*'],
  EMPLOYEE: ['profile:own', 'emergency:own']
}

// Middleware enforces permissions on every route
// Organization-scoped queries prevent cross-tenant data access
```

### Data Access Control Strategy

**1. Organization Isolation:**
- Every query includes `organizationId` filter
- Supervisors/Owners only see their organization's workers
- Master Admin has cross-org visibility with audit logging

**2. Emergency Profile Separation:**
- Emergency data stored in separate table with different access rules
- Worker controls which fields are populated (voluntary)
- Public endpoint only returns non-null emergency fields
- No compliance/certification data exposed via public key

**3. Database Schema Approach (Prisma):**

```prisma
model Worker {
  id              String   @id @default(cuid())
  organizationId  String
  organization    Organization @relation(fields: [organizationId])

  // Internal compliance data - Dashboard only
  certifications  Certification[]
  complianceData  ComplianceRecord[]

  // Emergency profile - Worker controlled
  emergencyProfile EmergencyProfile?

  // NFC Tag link
  nfcTag          NfcTag?
}

model EmergencyProfile {
  id              String   @id @default(cuid())
  workerId        String   @unique
  publicKey       String   @unique @default(cuid()) // Random token for NFC URL

  // Voluntary fields - Worker decides what to share
  medicalConditions String?
  allergies         String?
  bloodType         String?
  emergencyContact1 Json?
  emergencyContact2 Json?
  medications       String?
  specialInstructions String?

  // Encryption at field level for sensitive data
  encryptedData     Bytes?

  isActive          Boolean @default(true)
  lastAccessed      DateTime? // Audit: when was profile viewed
}

model NfcTag {
  id              String   @id @default(cuid())
  publicKey       String   @unique // Matches EmergencyProfile.publicKey
  workerId        String   @unique

  // Tag is linked once, never needs rewrite
  linkedAt        DateTime @default(now())
  isActive        Boolean  @default(true)
}
```

### Session/User Management

- Auth.js session extended with organization and role context
- Refresh token rotation for security
- Session invalidation on role change
- Audit log on every login/logout

### Protection of Sensitive Data

**1. Encryption Strategy:**
- PostgreSQL: Encryption at rest (AWS RDS default)
- Sensitive fields: Application-level encryption with AWS KMS
- Emergency profile: Additional encryption layer for medical data
- Secrets: AWS Secrets Manager (no hardcoded keys)

**2. Public Emergency Endpoint Security:**
```typescript
// GET /api/emergency/{publicKey}
// Rate limited, no auth required, minimal data exposure
const getEmergencyProfile = async (publicKey: string) => {
  // Only return fields worker chose to populate
  // Log access for audit trail (IP, timestamp, user-agent)
  // No internal compliance data exposed
}
```

**3. Audit Logging:**
- Every data access logged to CloudWatch
- Dashboard actions tied to user identity
- Emergency profile views logged with metadata
- Exportable for compliance reporting

---

## Architecture Simplification Ideas

Your current stack is solid. Here are optional simplifications:

| Current | Simplified Option | Trade-off |
|---------|-------------------|-----------|
| PostgreSQL + DynamoDB | DynamoDB only | Full serverless stack, pay-per-request, scales to zero, no connection pooling headaches |
| ECS Fargate | Lambda + API Gateway | Lower cost at low traffic, scales to zero, no container management |
| Lambda@Edge | CloudFront + standard Lambda | Edge is overkill unless sub-50ms global latency is required |
| Turborepo | Keep it | Good choice for monorepo, no change needed |

**My Recommendation:** Go full serverless with **DynamoDB + Lambda**. Here's why:

1. **DynamoDB fits your access patterns perfectly:**
   - Worker lookup by organization (GSI on organizationId)
   - Emergency profile lookup by publicKey (instant, single-digit ms)
   - Tag-to-worker linking (simple key-value)
   - Audit logs (time-series writes, perfect for DynamoDB)

2. **True serverless cost model:** Pay only when used. $0 when idle. No minimum monthly cost.
3. **No connection pooling:** Lambda + RDS = connection pool nightmares. Lambda + DynamoDB = no problem
4. **Emergency profile endpoint:** DynamoDB's single-digit millisecond reads + CloudFront caching = instant loads

### Database Options Comparison

| Option | Monthly Cost (MVP) | Serverless | Prisma Support | Best For |
|--------|-------------------|------------|----------------|----------|
| **DynamoDB** | ~$5-20 (pay per request) | Yes - true $0 idle | No | Simple access patterns, max cost efficiency |
| **Aurora PostgreSQL Serverless v2** | ~$50-100 minimum | Scales down but not to zero | Yes | Need Prisma + some serverless benefits |
| **RDS PostgreSQL** | ~$15-30 (always running) | No | Yes | Traditional setup, predictable costs |

**If you want Prisma:** Aurora PostgreSQL Serverless v2 is the best middle ground. It scales with demand and avoids Lambda connection pool issues via Data API. However, it has a minimum cost (~$50/month) even at low usage - it doesn't scale to true zero like DynamoDB.

**If you want maximum cost efficiency:** DynamoDB + Lambda is unbeatable. You literally pay nothing when no one is using the system. The trade-off is no Prisma - you'd use the AWS SDK directly or a library like ElectroDB for type-safe DynamoDB access.

**My honest take:** For an MVP, DynamoDB saves money and complexity. But if your team is already comfortable with Prisma and SQL, Aurora Serverless v2 is a solid choice - just know you're paying ~$50/month minimum for that convenience.

---

## Deliverables

### Backend Logic & API Routes
- Next.js API routes with clean separation
- Type-safe endpoints with Zod validation
- Error handling with consistent response format

### RBAC & Permission Handling
- Role-based middleware for all protected routes
- Organization-scoped queries
- Permission checks at service layer (defense in depth)

### CRUD Endpoints
- Workers: Full CRUD with role-based filtering
- Certifications: CRUD with expiration tracking
- Tags: Link/unlink, status management
- Incident Reports: Create, update, resolve workflow
- Organizations: Owner/Admin management

### Public Emergency Profile Endpoint
- Instant load (<200ms target)
- No authentication required
- Rate limited (prevent enumeration attacks)
- Audit logged
- CORS configured for NFC-triggered browser access

### Database Schema (Prisma)
- Complete schema with secure relations
- Migrations ready for deployment
- Indexes optimized for common queries
- Seed data for development/testing

### Clean, Maintainable Structure
```
apps/
  dashboard/          # Next.js frontend (your existing UI)
  api/                # API routes
packages/
  database/           # Prisma schema, client, migrations
  auth/               # Auth.js configuration, RBAC utils
  types/              # Shared TypeScript types
  utils/              # Common utilities
```

---

## Relevant Experience - Live Production Platforms

### 1. Simple Booking - Multi-Tenant SaaS Platform (Most Relevant)
- **Client Store:** https://finance-dev.simple-booking.org/services
- **Admin Dashboard:** https://admin-dev.simple-booking.org/calendar
- **Demo Credentials:** finance-dev@simple-booking.org / Pass123$
- **Stack:** Vue.js frontend, Python/Lambda backend, API Gateway, DynamoDB
- **Why It's Relevant:**
  - Multi-role dashboard with user management
  - CRUD operations for bookings, services, clients
  - Real-time calendar with role-based visibility
  - Stripe payment integration (service payments + SaaS subscriptions)
  - Twilio & AWS SNS for automated messaging
  - Production-ready handling real money transactions

### 2. Short-ly Document Processing Platform
- **Live Platform:** https://admin.short-ly.net/login
- **Demo Credentials:** demo.user@short-ly.net / Pass123$
- **Stack:** Vue.js frontend, Python/Lambda backend, S3, DynamoDB, AWS Bedrock
- **Why It's Relevant:**
  - JWT Authentication with refresh tokens (same pattern you'll need)
  - File upload system with progress tracking
  - S3 integration with presigned URLs and lifecycle policies
  - API Gateway with CORS, rate limiting, security configured
  - Production monitoring and error handling
  - Infrastructure as Code (Terraform) - fully scripted AWS deployment
  - CI/CD with GitHub Actions

### Certifications & Experience:
- **AWS Certified Solutions Architect Professional**
- Extensive experience in automotive industry with large corporation
- Deep understanding of compliance, reporting, and role-based security

### Development Approach:
I use **Claude Code Max** with parallel AI agents for accelerated development:
- Parallel task execution across features
- Automated testing and code review
- Comprehensive documentation generated alongside code

---

## What I Need From You

1. **Figma/Design Access:** To understand the dashboard UI structure and data requirements
2. **Auth.js Configuration:** Current setup to extend (or I can configure from scratch)
3. **NFC Tag URL Format:** Confirm the URL structure you're using (e.g., `yourdomain.com/e/{publicKey}`)
4. **Priority Features:** Which CRUD operations are most critical for MVP?
5. **AWS Account Access:** For deployment and infrastructure setup

---

## Next Steps

1. **This Week:** 30-minute call to review your Figma designs and clarify RBAC requirements
2. **Upon Agreement:** Kickoff within 3-5 days
3. **Week 1:** Core RBAC, authentication extension, worker CRUD
4. **Week 2:** Emergency profile endpoint, tag linking, certifications
5. **Week 3:** Audit logs, deployment, testing, documentation

---

## Why This Partnership Works

You mentioned you're looking for someone who values culture and building something meaningful. I'm not here for a quick project flip. I've built SaaS platforms that are running in production today. 

I communicate directly, I don't over-engineer, and I ship working code. If I see a simpler path that maintains security, I'll propose it. If something will take longer than expected, you'll know immediately.

**Let's build something that works when it matters most.**

---

Best regards,

**Cristian Stoichin**

P.S. Try my live platforms before we talk. The Simple Booking admin dashboard and the Document Processing system both demonstrate the patterns this project needs: authentication, role-based access, CRUD operations, and clean API design.
