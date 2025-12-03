# Tenant Protection Program Automation - Fixed Price Proposal

**Prepared by:** Cristian Stoichin

**Date:** December 2, 2025

**Project:** Automated Tenant Enrollment/Unenrollment Script for Cubby Storage

---

## Why I'm the Right Fit for This Project

**Live Production Platforms - See My Work:**

**1. Simple Booking SaaS Platform (Scheduled Automation):**
- **Client Store:** https://finance-dev.simple-booking.org
- **Admin Dashboard:** https://admin-dev.simple-booking.org
- **Already Built:** Automated scheduling, API integrations, AWS Lambda deployments, DynamoDB, email/SMS notifications via Twilio & AWS SNS

**2. Document Processing Platform:**
- **Live Platform:** https://admin.short-ly.net/login
- **Demo Credentials:** demo.user@short-ly.net / Pass123$
- **Already Built:** Python/Lambda backend, API Gateway, scheduled processing jobs, error handling, audit logging

**What This Means for Your Project:** I've built and deployed production Python automation scripts that run on schedules, integrate with external APIs, and handle time-sensitive business logic. The patterns you need (API integration, conditional logic based on timestamps, error handling, scheduled execution) are exactly what I do regularly.

---

## I Understand Your Requirements

Your automation has three core workflows that need reliable, auditable execution:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    AUTO-ENROLLMENT LOGIC                                 │
├─────────────────────────────────────────────────────────────────────────┤
│  NEW TENANTS                        │  EXISTING TENANTS                  │
│  - Opted for own insurance          │  - Previously had valid policy     │
│  - No documentation within 48hrs    │  - Policy lapsed/expired           │
│  → Auto-enroll in protection        │  → Auto-enroll in protection       │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    AUTO-UNENROLLMENT LOGIC                               │
├─────────────────────────────────────────────────────────────────────────┤
│  Account 30+ days past due → Unenroll from protection program           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    RE-ENROLLMENT LOGIC                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  Previously unenrolled (past due) + Account now current → Re-enroll     │
│  Document lapse via API flags/notes for compliance                       │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Technical Considerations:**
- Time-sensitive rules require precise timestamp handling
- API rate limits need to be respected
- All actions must be logged for auditing
- Failures need immediate notification
- Secure credential management (no hardcoded API keys)

---

## Technical Approach

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                   AWS Lambda Function                            │
│                   (Python 3.11+)                                 │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Fetcher   │  │  Processor  │  │        Updater          │  │
│  │ (API Query) │→ │  (Rules)    │→ │ (Enroll/Unenroll API)   │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────────┐
│  Cubby Storage  │ │ AWS CloudWatch  │ │   Email/Slack Alerts    │
│      API        │ │     Logs        │ │   (Failures Only)       │
└─────────────────┘ └─────────────────┘ └─────────────────────────┘
```

### Script Logic Flow

```python
# Pseudocode for main execution flow

def handler(event, context):
    """Daily scheduled execution"""

    # 1. Initialize secure API client
    api = CubbyStorageClient(get_secret("CUBBY_API_KEY"))

    # 2. Fetch all relevant tenant data
    tenants = api.get_tenants(include=["insurance", "balance", "enrollment"])

    # 3. Process each tenant through rules engine
    actions = []
    for tenant in tenants:
        action = evaluate_tenant(tenant)
        if action:
            actions.append(action)

    # 4. Execute enrollment/unenrollment actions
    results = execute_actions(api, actions)

    # 5. Log all actions for audit trail
    log_results(results)

    # 6. Send notifications on failures
    if results.failures:
        send_alert(results.failures)

    return results.summary()
```

### Rules Engine

```python
def evaluate_tenant(tenant):
    """
    Evaluate tenant against all business rules
    Returns: EnrollAction, UnenrollAction, ReenrollAction, or None
    """

    # Rule 1: New tenant, opted for own insurance, no docs after 48hrs
    if is_new_tenant(tenant) and \
       opted_own_insurance(tenant) and \
       hours_since_movein(tenant) > 48 and \
       not has_valid_documentation(tenant):
        return EnrollAction(tenant, reason="48hr_no_docs")

    # Rule 2: Existing tenant with lapsed policy
    if has_lapsed_policy(tenant) and not is_enrolled(tenant):
        return EnrollAction(tenant, reason="policy_lapsed")

    # Rule 3: Account 30+ days past due
    if days_past_due(tenant) >= 30 and is_enrolled(tenant):
        return UnenrollAction(tenant, reason="30_days_past_due")

    # Rule 4: Previously unenrolled, now current
    if was_unenrolled_for_nonpayment(tenant) and \
       is_account_current(tenant):
        return ReenrollAction(tenant, reason="account_current")

    return None
```

---

## Deliverables

### Phase 1 - Core Script Development
- Python script with clean, modular architecture
- Cubby Storage API integration:
  - Secure authentication using provided API keys
  - Tenant data queries (move-in dates, insurance details, balances)
  - Enrollment/unenrollment API calls
- Rules engine implementing all four business rules:
  - 48-hour documentation window for new tenants
  - Lapsed policy detection for existing tenants
  - 30-day past due unenrollment
  - Re-enrollment on account current
- Comprehensive error handling with retry logic

### Phase 2 - Logging & Notifications
- CloudWatch logging for full audit trail:
  - Every action logged with tenant ID, action type, reason, timestamp
  - API response logging for debugging
  - Daily summary logs
- Email notifications on failures (configurable recipients)
- Optional: Slack webhook integration for real-time alerts
- Dry-run mode for testing without making changes

### Phase 3 - Deployment & Documentation
- AWS Lambda deployment with:
  - EventBridge (CloudWatch Events) for daily scheduling
  - Environment variables for configuration
  - AWS Secrets Manager for API key storage
  - IAM roles with least-privilege permissions
- Infrastructure as Code (Terraform or CloudFormation)
- Comprehensive documentation:
  - Setup guide
  - Configuration options
  - Business rule documentation
  - Troubleshooting guide
- Unit tests for rules engine
- Integration tests with mock API responses

### What's Included:
- Complete GitHub repository with all code
- Infrastructure as Code for AWS deployment
- Comprehensive README with setup instructions
- Unit and integration tests
- 7 days post-deployment support
- 1-hour handoff/training session

### What's NOT Included:
- AWS monthly infrastructure costs (~$5-10/month for this workload)
- Cubby Storage API access (you provide)
- Ongoing maintenance (separate contract available)

---

## Timeline: 2-3 Weeks

| Week | Phase | Milestone |
|------|-------|-----------|
| **Week 1** | Phase 1 | API integration, rules engine, core script functionality |
| **Week 2** | Phase 2-3 | Logging, notifications, testing, deployment, documentation |
| **Week 3** | Buffer | Final testing, edge case handling, handoff |

**Availability:** I can start within 3-5 days of project approval.

### Communication Plan:
- Daily Slack/email updates
- Shared GitHub repository with PR-based development
- Weekly video check-in (or as needed)
- Documentation in repository README

---

## AWS Monthly Cost Estimate

| Service | Configuration | Monthly Cost |
|---------|---------------|--------------|
| **Lambda** | Daily execution (~30 invocations/month) | $0 (free tier) |
| **CloudWatch Logs** | Log storage & retention | $2 |
| **Secrets Manager** | 1-2 secrets | $1 |
| **SNS** | Email notifications | $0 (free tier) |
| **EventBridge** | Scheduled rule | $0 |
| | | |
| **TOTAL** | | **~$3-5/month** |

This is an extremely cost-efficient solution that scales with your needs.

---

## Relevant Experience

### Python & AWS Expertise:
- **AWS Certified Solutions Architect Professional**
- Extensive experience with Python scripting and automation
- Multiple production Lambda functions deployed and running
- API integrations with external services (Stripe, Twilio, various SaaS APIs)

### Live Production Platforms:

**1. Simple Booking SaaS Platform:**
- **Live:** https://admin-dev.simple-booking.org
- **Stack:** Python/Lambda, API Gateway, DynamoDB
- **Relevance:** Scheduled automation, payment processing, API integrations

**2. Document Processing Platform:**
- **Live:** https://admin.short-ly.net/login
- **Stack:** Python/Lambda, S3, DynamoDB
- **Relevance:** File processing automation, error handling, logging patterns

### Development Approach:
I use **Claude Code Max** with parallel AI agents for accelerated development:
- Comprehensive test coverage
- Clean, well-documented code
- Faster iteration cycles

---

## Questions About the API

Before starting, I'd like to clarify:

1. **API Documentation:** Do you have Cubby Storage API documentation, or will we need to discover endpoints?
2. **Tenant Status Fields:** What specific fields indicate:
   - Tenant opted for own insurance vs. protection plan?
   - Insurance documentation submitted (declaration page, policy number, expiration)?
   - Current enrollment status in protection program?
3. **API Rate Limits:** Are there any rate limits on the Cubby Storage API?
4. **Test Environment:** Is there a sandbox/test environment for development?
5. **Historical Data:** Should the script process all existing tenants on first run, or only new events?
6. **Timezone:** What timezone should 48-hour windows and delinquency calculations use?

---

## Why Choose Me

| Factor | My Advantage |
|--------|--------------|
| **Proven Platforms** | Test my live systems before we talk |
| **AWS Expertise** | Certified Solutions Architect with Lambda experience |
| **Python Experience** | Production automation scripts running daily |
| **Fast Delivery** | 2-3 week timeline with thorough testing |
| **Clean Code** | Well-documented, maintainable, testable |
| **Post-Delivery Support** | 7 days included + ongoing maintenance available |

---

## Fixed Price: $3,500

This includes:
- Complete script development
- AWS Lambda deployment
- Infrastructure as Code
- Full documentation
- Unit and integration tests
- 7 days post-deployment support
- 1-hour handoff session

**Payment Terms:** 50% upfront, 50% upon delivery

---

## Next Steps

1. **This Week:** 30-minute call to review Cubby Storage API details and clarify requirements
2. **Upon Agreement:** Kickoff within 3-5 days
3. **Week 2-3:** Your automation is deployed and running

---

I'm confident I can deliver a reliable, well-tested automation script that handles your tenant protection enrollment logic. My experience with scheduled Python scripts on AWS Lambda, combined with proper error handling and logging, will give you a system you can trust to run daily without intervention.

**Let's automate this and save your team hours of manual work.**

Best regards,

**Cristian Stoichin**

P.S. Visit my live platforms before we talk - the reliability and quality you see there is exactly what you'll get for this project.
