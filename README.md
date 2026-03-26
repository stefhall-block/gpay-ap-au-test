# Google Pay Gateway - Australia Expansion Implementation Plan

**Project:** Enable Google Pay Gateway for Australia Paylater  
**Epic:** Multi-Region Support for Google Pay Gateway  
**Priority:** High  
**Target Release:** Q2 2026  
**Owner:** Commerce Payments/ Merchant Services 
**Created:** March 25, 2026

---

## 📋 Executive Summary

Enable Google Pay Gateway integration to support Australian merchants using Afterpay Paylater. Currently, the integration is hardcoded to use `Region.US`, preventing AU merchants from completing checkout flows.

**Business Impact:**
- Unlock AU market for Google Pay integration
- Enable Paylater product in Australia via Google Pay
- Foundation for future multi-region expansion (GB, ES)

**Technical Scope:**
- Update merchant data model to include region
- Modify checkout flow to use merchant-specific region
- Add validation and testing for multi-region support

---

## 🎯 Goals & Success Criteria

### Goals
1. ✅ Enable AU merchants to complete Google Pay checkout flows
2. ✅ Maintain backward compatibility with existing US merchants
3. ✅ Create scalable foundation for additional regions
4. ✅ Zero downtime deployment

### Success Criteria
- [ ] AU merchant can complete end-to-end checkout
- [ ] Virtual card retrieval succeeds for AU merchants
- [ ] Existing US merchants experience no disruption
- [ ] <1% error rate increase during rollout
- [ ] 100% test coverage for region routing logic

### Non-Goals
- GB/ES region support (future phase)
- Currency conversion logic
- Region-specific product features

---

## 📊 Project Phases

### Phase 1: Quick Win - Feature Flag Implementation
**Timeline:** 2-3 days  
**Goal:** Enable AU testing in sandbox

### Phase 2: Production Solution - Merchant Region Support
**Timeline:** 2 weeks  
**Goal:** Production-ready multi-region support

### Phase 3: Rollout & Monitoring
**Timeline:** 1 week  
**Goal:** Safe production deployment

---


## Phase 1: Quick Win (Sprint 1)

### Story 1: Add Feature Flag for AU Region Testing

**Story Points:** 2  
**Priority:** High  
**Labels:** `quick-win`, `feature-flag`, `australia`

**Description:**
As a developer, I want to add a feature flag to enable AU region testing so that we can validate the integration works before making database changes.

**Acceptance Criteria:**
- [ ] Feature flag `gpay-au-region-enabled` created in LaunchDarkly
- [ ] Flag defaults to `false` (US region)
- [ ] When enabled, uses `Region.AU` instead of `Region.US`
- [ ] Flag can be toggled per merchant ID
- [ ] Logging added to track which region is used

**Technical Details:**
```java
// In CheckoutResourceManager.java line ~320
Region region = launchDarklyGateway.boolVariation(
    "gpay-au-region-enabled", 
    buildFeatureFlagContext(orderDto.gpayMerchantId()),
    false
) ? Region.AU : Region.US;

GetCheckoutResponse getCheckoutResponse = afterpayApiGateway.getCheckoutByToken(
    BEARER + accessToken,
    USER_AGENT,
    region,  // Use feature flag determined region
    orderDto.checkoutToken()
);
```

**Files to Modify:**
- `gpay/server/src/main/java/com/afterpay/gateway/gpay/server/resources/v1/CheckoutResourceManager.java`

**Testing:**
- Unit test with flag enabled/disabled
- Integration test with AU merchant in sandbox
- Verify US merchants unaffected

---

### Story 2: Sandbox Testing with AU Merchant

**Story Points:** 3  
**Priority:** High  
**Labels:** `testing`, `australia`, `sandbox`

**Description:**
As a QA engineer, I want to test the complete checkout flow with an AU merchant in sandbox to validate the integration works end-to-end.
We will do this with a Gpay Test merchant similar to how we have this constructed in admin portal in Production-US. There is one singular MID, that serves as an umbrella MID for all corresponding merchants onboarded by GPay.

**Acceptance Criteria:**
- [ ] AU test merchant created in sandbox
- [ ] Feature flag enabled for test merchant
- [ ] Complete checkout flow tested:
  - [ ] Create checkout
  - [ ] User completes Afterpay flow
  - [ ] Read card (virtual card retrieval)
  - [ ] Verify correct AU API endpoint used
- [ ] OAuth flow tested (if applicable)
- [ ] Error scenarios tested
- [ ] Test results documented

**Test Scenarios:**
1. Happy path: AU merchant checkout → virtual card
2. US merchant still works (flag disabled)
3. Invalid region handling
4. API timeout scenarios
5. OAuth token refresh (AU)

**Dependencies:**
- [GPAY-001] Feature flag implementation

---

### Story 3: [GPAY-003] Document Feature Flag Approach & Limitations

**Story Points:** 1  
**Priority:** Medium  
**Labels:** `documentation`

**Description:**
As a tech lead, I want to document the feature flag approach and its limitations so the team understands this is a temporary solution.

**Acceptance Criteria:**
- [ ] README updated with feature flag usage
- [ ] Limitations documented (not merchant-specific)
- [ ] Runbook created for toggling flag
- [ ] Rollback procedure documented
- [ ] Link to Phase 2 implementation plan

**Deliverables:**
- `docs/gpay-au-feature-flag.md`
- Updated `README.md`
- Runbook in Confluence/Wiki

---

## Phase 2: Production Solution (Sprint 2-3)

### Story 4: Add Region Field to Merchant Data Model

**Story Points:** 5  
**Priority:** High  
**Labels:** `database`, `data-model`, `breaking-change`

**Description:**
As a backend engineer, I want to add a region field to the merchant data model so we can properly route requests to the correct Afterpay API.

**Acceptance Criteria:**
- [ ] `region` field added to `MerchantDto`
- [ ] Database migration script created
- [ ] Migration tested in dev environment
- [ ] Rollback script created
- [ ] Default value set to "US" for existing merchants
- [ ] Region validation added (only US, AU, GB allowed)

**Database Changes:**
```sql
-- Migration: V1_XX__add_region_to_merchants.sql
ALTER TABLE gpay_merchants 
ADD COLUMN region VARCHAR(2) NOT NULL DEFAULT 'US';

CREATE INDEX idx_merchants_region ON gpay_merchants(region);

-- Add constraint
ALTER TABLE gpay_merchants 
ADD CONSTRAINT chk_region CHECK (region IN ('US', 'AU', 'GB', 'ES'));
```

**Code Changes:**
```java
// MerchantDto.java
public record MerchantDto(
    String gpayMerchantId,
    String gpayMerchantReference,
    String originalGoogleMerchantReference,
    String merchantDisplayName,
    String businessLocation,
    String organizationName,
    String fullAddress,
    String corporateWebsite,
    String customerSupportUrl,
    String customerSupportEmail,
    String customerSupportPhone,
    String mcc,
    Long afterpayMerchantId,
    String region  // NEW FIELD
) {
    // Add validation
    public MerchantDto {
        if (region == null || region.isEmpty()) {
            throw new IllegalArgumentException("Region cannot be null or empty");
        }
        if (!List.of("US", "AU", "GB", "ES").contains(region)) {
            throw new IllegalArgumentException("Invalid region: " + region);
        }
    }
}
```

**Files to Modify:**
- `gpay/dao/src/main/java/com/afterpay/gateway/gpay/dao/models/MerchantDto.java`
- `gpay/dao/src/main/java/com/afterpay/gateway/gpay/dao/MerchantDao.java`
- `gpay/persistence/src/main/resources/db/migration/V1_XX__add_region_to_merchants.sql`

**Testing:**
- Unit tests for validation
- Migration test in dev/staging
- Rollback test
- Performance test (ensure index works)

**Dependencies:**
- DBA review required
- Migration approval needed

---

### Story 5: Update Merchant Onboarding to Require Region

**Story Points:** 3  
**Priority:** High  
**Labels:** `onboarding`, `validation`

**Description:**
As a backend engineer, I want to update merchant onboarding to require and validate region so new merchants are properly configured.

**Acceptance Criteria:**
- [ ] Region field required in merchant creation API
- [ ] Region validation added
- [ ] Error messages clear and actionable
- [ ] Admin endpoint updated to set/update region
- [ ] Audit logging for region changes

**API Changes:**
```json
// POST /gpay/v1/admin/merchants
{
  "gpayMerchantId": "merchant_123",
  "merchantDisplayName": "AU Test Merchant",
  "region": "AU",  // NEW REQUIRED FIELD
  // ... other fields
}
```

**Validation Rules:**
- Region must be one of: US, AU, GB, ES
- Region must match Afterpay merchant account region
- Cannot change region after merchant is created (immutable)

**Files to Modify:**
- `gpay/server/src/main/java/com/afterpay/gateway/gpay/server/resources/AdminResource.java`
- `gpay/server/src/main/java/com/afterpay/gateway/gpay/server/resources/AdminResourceManager.java`
- `gpay/persistence/src/main/java/com/afterpay/gateway/gpay/persistence/MerchantPersistence.java`

**Testing:**
- Unit tests for validation
- Integration tests for admin API
- Test invalid region rejection
- Test region immutability

---

### Story 6:  Update CheckoutResourceManager to Use Merchant Region

**Story Points:** 3  
**Priority:** High  
**Labels:** `checkout`, `core-logic`

**Description:**
As a backend engineer, I want to update the checkout flow to use the merchant's region instead of hardcoded US so AU merchants can complete checkouts.

**Acceptance Criteria:**
- [ ] Remove hardcoded `Region.US`
- [ ] Fetch merchant region from database
- [ ] Use merchant region in API calls
- [ ] Add error handling for missing/invalid region
- [ ] Add logging for region routing
- [ ] Metrics added for region usage

**Code Changes:**
```java
// CheckoutResourceManager.java
private ReadCardResponse processCardConfirmation(
    ConfirmSingleUseCardCheckoutRequest confirmSingleUseCardCheckoutRequest,
    GpayOrderDto orderDto,
    ReadCardRequest readCardRequest
) {
    ConfirmSingleUseCardCheckoutResponse confirmSingleUseCardCheckoutResponse = 
        apiPlusGateway.confirmSingleUseCardCheckout(confirmSingleUseCardCheckoutRequest);

    // NEW: Fetch merchant to get region
    MerchantDto merchant = merchantService.getMerchant(orderDto.gpayMerchantId());
    Region region = Region.valueOf(merchant.region());
    
    LOGGER.info("Using region {} for merchant {}", 
        region, orderDto.gpayMerchantId());

    String accessToken = doormanGateway.getToken(
        gPayMerchantApipConfiguration.gpayMerchantId()
    );
    
    GetCheckoutResponse getCheckoutResponse = afterpayApiGateway.getCheckoutByToken(
        BEARER + accessToken,
        USER_AGENT,
        region,  // Use merchant's region
        orderDto.checkoutToken()
    );

    // ... rest of method
}
```

**Files to Modify:**
- `gpay/server/src/main/java/com/afterpay/gateway/gpay/server/resources/v1/CheckoutResourceManager.java`

**Metrics to Add:**
- `gpay.checkout.region.us` (counter)
- `gpay.checkout.region.au` (counter)
- `gpay.checkout.region.error` (counter)

**Testing:**
- Unit tests with US merchant
- Unit tests with AU merchant
- Integration tests for both regions
- Error handling tests

**Dependencies:**
- [GPAY-004] Merchant region field
- [GPAY-005] Merchant onboarding

---

### Story 7: Add Region-Specific Integration Tests

**Story Points:** 5  
**Priority:** High  
**Labels:** `testing`, `integration-tests`

**Description:**
As a QA engineer, I want comprehensive integration tests for multi-region support so we can confidently deploy to production.

**Acceptance Criteria:**
- [ ] Integration tests for US merchant checkout
- [ ] Integration tests for AU merchant checkout
- [ ] Tests for region routing logic
- [ ] Tests for error scenarios
- [ ] Tests for OAuth flow (both regions)
- [ ] Tests for account linking (both regions)
- [ ] Performance tests (no regression)

**Test Coverage:**
```
✓ US Merchant
  ✓ Create checkout
  ✓ Complete checkout
  ✓ Read card (virtual card)
  ✓ OAuth flow
  ✓ Account linking

✓ AU Merchant
  ✓ Create checkout
  ✓ Complete checkout
  ✓ Read card (virtual card)
  ✓ OAuth flow
  ✓ Account linking

✓ Error Scenarios
  ✓ Invalid region
  ✓ Missing region
  ✓ API timeout (US)
  ✓ API timeout (AU)
  ✓ Unauthorized (both regions)

✓ Edge Cases
  ✓ Region mismatch (merchant vs checkout)
  ✓ Feature flag + database region conflict
  ✓ Migration rollback scenario
```

**Files to Create:**
- `gpay/server/src/integrationTest/java/com/afterpay/gateway/gpay/server/MultiRegionCheckoutTest.java`
- `gpay/server/src/integrationTest/java/com/afterpay/gateway/gpay/server/AuMerchantCheckoutTest.java`

**Testing:**
- Run in CI/CD pipeline
- Run against sandbox environment
- Performance benchmarks

---

### Story 8: Backfill Existing Merchants with Region Data

**Story Points:** 3  
**Priority:** Medium  
**Labels:** `data-migration`, `ops`

**Description:**
As a data engineer, I want to backfill existing merchants with correct region data so the system works correctly after migration.

**Acceptance Criteria:**
- [ ] Script created to identify merchant regions
- [ ] Dry-run executed and validated
- [ ] Backfill script executed in staging
- [ ] Results validated
- [ ] Production backfill plan approved
- [ ] Rollback plan documented

**Backfill Strategy:**
1. Default all existing merchants to "US"
2. Identify AU merchants (if any exist)
3. Manual verification of region assignments
4. Execute backfill with monitoring

**Script:**
```sql
-- Identify potential AU merchants (manual review needed)
SELECT 
    gpay_merchant_id,
    merchant_display_name,
    business_location,
    full_address
FROM gpay_merchants
WHERE business_location LIKE '%AU%' 
   OR business_location LIKE '%Australia%'
   OR full_address LIKE '%Australia%';

-- Backfill (after manual verification)
UPDATE gpay_merchants 
SET region = 'AU' 
WHERE gpay_merchant_id IN (
    -- List of verified AU merchants
    'merchant_au_1',
    'merchant_au_2'
);

-- Verify
SELECT region, COUNT(*) 
FROM gpay_merchants 
GROUP BY region;
```

**Deliverables:**
- Backfill script
- Verification queries
- Execution runbook
- Rollback procedure

**Dependencies:**
- [GPAY-004] Database migration
- Business team to identify AU merchants

---

### Story 9: [GPAY-009] Update Documentation & Runbooks

**Story Points:** 2  
**Priority:** Medium  
**Labels:** `documentation`

**Description:**
As a tech writer, I want to update all documentation to reflect multi-region support so teams know how to use and troubleshoot the feature.

**Acceptance Criteria:**
- [ ] README updated with region support
- [ ] API documentation updated
- [ ] Merchant onboarding guide updated
- [ ] Troubleshooting runbook created
- [ ] Architecture diagrams updated
- [ ] Release notes drafted

**Documentation Updates:**

**README.md:**
```markdown
## Multi-Region Support

Google Pay Gateway now supports multiple regions:
- **US**: United States and Canada
- **AU**: Australia and New Zealand
- **GB**: United Kingdom (coming soon)
- **ES**: Europe (coming soon)

### Configuration
Merchants are assigned a region during onboarding. The region determines:
- Which Afterpay API endpoint is used
- OAuth configuration
- Currency handling

### Adding a New Merchant
```bash
POST /gpay/v1/admin/merchants
{
  "region": "AU",  // Required
  ...
}
```
```

**Runbook Topics:**
- How to identify merchant region
- How to troubleshoot region routing issues
- How to add support for new region
- Common error scenarios

**Files to Update:**
- `README.md`
- `docs/merchant-onboarding.md`
- `docs/troubleshooting.md`
- `docs/architecture.md`

---

## 🚀 Phase 3: Rollout & Monitoring (Sprint 4)

### Story 10: [GPAY-010] Deploy to Staging & Conduct UAT

**Story Points:** 3  
**Priority:** High  
**Labels:** `deployment`, `uat`, `staging`

**Description:**
As a release manager, I want to deploy the multi-region changes to staging and conduct UAT so we can validate everything works before production.

**Acceptance Criteria:**
- [ ] Database migration executed in staging
- [ ] Code deployed to staging
- [ ] Feature flag configured
- [ ] UAT test plan executed
- [ ] Performance testing completed
- [ ] No critical bugs found
- [ ] Sign-off from QA and Product

**UAT Test Plan:**
1. **US Merchant Testing**
   - Create 5 test checkouts
   - Verify all succeed
   - Check logs for correct region

2. **AU Merchant Testing**
   - Create 5 test checkouts
   - Verify all succeed
   - Check logs for correct region

3. **Performance Testing**
   - Load test with 100 concurrent checkouts
   - Verify <500ms p95 latency
   - No error rate increase

4. **Monitoring Validation**
   - Verify metrics are being collected
   - Verify alerts are configured
   - Test alert firing

**Success Criteria:**
- 100% test pass rate
- No performance regression
- All monitoring working

**Dependencies:**
- All Phase 2 stories completed
- Staging environment ready

---

### Story 11: [GPAY-011] Production Deployment Plan & Rollback Procedure

**Story Points:** 2  
**Priority:** High  
**Labels:** `deployment`, `production`, `runbook`

**Description:**
As a release manager, I want a detailed production deployment plan and rollback procedure so we can safely deploy with minimal risk.

**Acceptance Criteria:**
- [ ] Deployment runbook created
- [ ] Rollback procedure documented
- [ ] Communication plan created
- [ ] On-call schedule confirmed
- [ ] Monitoring dashboard created
- [ ] Incident response plan ready

**Deployment Runbook:**

**Pre-Deployment Checklist:**
- [ ] All tests passing in CI/CD
- [ ] UAT sign-off received
- [ ] Database migration tested in staging
- [ ] Rollback plan reviewed
- [ ] On-call team notified
- [ ] Feature flag configured (disabled)

**Deployment Steps:**
1. **Database Migration** (Maintenance window: 2 hours)
   - Execute migration script
   - Verify migration success
   - Run validation queries
   - Checkpoint: Can rollback here

2. **Code Deployment** (Blue-Green deployment)
   - Deploy to 10% of traffic
   - Monitor for 30 minutes
   - Deploy to 50% of traffic
   - Monitor for 30 minutes
   - Deploy to 100% of traffic

3. **Feature Flag Rollout**
   - Enable for 1 test merchant
   - Monitor for 1 hour
   - Enable for all AU merchants
   - Monitor for 24 hours

**Monitoring During Rollout:**
- Error rate: <1% increase
- Latency: <10% increase
- Success rate: >99%
- Region routing: Verify correct distribution

**Rollback Triggers:**
- Error rate >5%
- Latency >2x baseline
- Critical bug discovered
- Data corruption detected

**Rollback Procedure:**
1. Disable feature flag (immediate)
2. Rollback code deployment (if needed)
3. Rollback database migration (if needed)
4. Verify system stability
5. Post-mortem

**Communication Plan:**
- Pre-deployment: Email to stakeholders
- During deployment: Slack updates every 30 min
- Post-deployment: Summary email
- If issues: Immediate incident notification

---

### Story 12: [GPAY-012] Post-Deployment Monitoring & Validation

**Story Points:** 2  
**Priority:** High  
**Labels:** `monitoring`, `validation`, `production`

**Description:**
As an SRE, I want to monitor the production deployment for 72 hours to ensure stability and catch any issues early.

**Acceptance Criteria:**
- [ ] Monitoring dashboard created
- [ ] Alerts configured
- [ ] Daily health checks for 3 days
- [ ] Metrics analyzed and documented
- [ ] No critical issues found
- [ ] Post-deployment report created

**Alerts to Configure:**
- Error rate >5% (P1)
- Latency >1000ms (P2)
- Region routing failure (P1)
- Database connection issues (P1)

**Daily Health Checks:**
- Review error logs
- Check success rates
- Analyze latency trends
- Verify region distribution
- Check for anomalies

**Post-Deployment Report:**
- Summary of deployment
- Metrics comparison (before/after)
- Issues encountered and resolved
- Lessons learned
- Recommendations

---

## Success Metrics

### Technical Metrics
| Metric | Target | Measurement |
|--------|--------|-------------|
| AU Checkout Success Rate | >99% | DataDog |
| US Checkout Success Rate | >99% (no regression) | DataDog |
| API Latency (p95) | <500ms | DataDog |
| Error Rate | <1% | DataDog |
| Test Coverage | >90% | SonarQube |

### Business Metrics
| Metric | Target | Measurement |
|--------|--------|-------------|
| AU Merchant Onboarding | 5+ in first month | Internal dashboard |
| AU Transaction Volume | Track baseline | Analytics |
| Support Tickets | <5 region-related | Zendesk |

---

## Risks & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **Breaking US merchants** | Medium | High | Feature flag rollout, extensive testing, blue-green deployment |
| **AU API differences** | Low | Medium | Thorough sandbox testing, API documentation review |
| **Database migration failure** | Low | High | Test in staging, have rollback script ready, maintenance window |
| **Region mismatch errors** | Medium | Medium | Validation at onboarding, clear error messages, monitoring |
| **Performance degradation** | Low | Medium | Load testing, gradual rollout, performance monitoring |
| **Data corruption** | Very Low | Critical | Database backups, transaction safety, rollback plan |

---

## 🔗 Dependencies

### Internal Dependencies
- **DBA Team**: Database migration review and execution
- **QA Team**: Integration testing and UAT
- **Product Team**: AU merchant identification and onboarding
- **SRE Team**: Monitoring and deployment support

### External Dependencies
- **Afterpay API**: AU endpoint availability and stability
- **LaunchDarkly**: Feature flag infrastructure
- **Google**: No changes needed (gateway-side only)

### Blocking Dependencies
- None - all dependencies can be worked in parallel

---

## Timeline & Milestones

```
Week 1-2: Phase 1 - Quick Win
├─ Day 1-2: [GPAY-001] Feature flag implementation
├─ Day 3-4: [GPAY-002] Sandbox testing
└─ Day 5: [GPAY-003] Documentation

Week 3-4: Phase 2 - Core Implementation
├─ Week 3:
│  ├─ [GPAY-004] Database schema changes
│  ├─ [GPAY-005] Merchant onboarding updates
│  └─ [GPAY-006] Checkout flow updates
└─ Week 4:
   ├─ [GPAY-007] Integration tests
   ├─ [GPAY-008] Data backfill
   └─ [GPAY-009] Documentation

Week 5: Phase 3 - Rollout
├─ Day 1-3: [GPAY-010] Staging deployment & UAT
├─ Day 4: [GPAY-011] Production deployment
└─ Day 5-7: [GPAY-012] Monitoring & validation

Week 6: Post-Launch
└─ Monitoring, bug fixes, optimization
```

**Key Milestones:**
- ✅ **M1**: Feature flag working in sandbox (End of Week 1)
- ✅ **M2**: Database migration complete in staging (End of Week 3)
- ✅ **M3**: All integration tests passing (End of Week 4)
- ✅ **M4**: Production deployment complete (End of Week 5)
- ✅ **M5**: 72-hour stability confirmed (End of Week 6)

---

## 👥 Team & Responsibilities

| Role | Responsibility | Team Member |
|------|---------------|-------------|
| **Tech Lead** | Overall technical direction, code review | [Name] |
| **Backend Engineer** | Core implementation (GPAY-004, 005, 006) | [Name] |
| **QA Engineer** | Testing strategy, UAT (GPAY-002, 007, 010) | [Name] |
| **Data Engineer** | Database migration, backfill (GPAY-004, 008) | [Name] |
| **SRE** | Deployment, monitoring (GPAY-011, 012) | [Name] |
| **Product Manager** | Requirements, AU merchant coordination | [Name] |
| **Tech Writer** | Documentation (GPAY-003, 009) | [Name] |

---

## 📚 Reference Documentation

### Technical Docs
- [Google Pay Gateway Architecture](link)
- [Afterpay API Documentation - AU](https://developers.afterpay.com/afterpay-online/reference/api-reference-v2)
- [Multi-Region Support Analysis](/Users/stefanhall/google-pay-au-paylater-analysis.md)

### Related Projects
- [APIP-XXX] API Plus Multi-Region Support
- [AUTH-XXX] OAuth Multi-Region Configuration

### Code References
- `CheckoutResourceManager.java:322` - Current hardcoded region
- `DefaultAfterpayApiGateway.java:48-60` - Region client mapping
- `MerchantDto.java` - Merchant data model

---

## 🔄 Post-Launch Activities

### Week 1 Post-Launch
- [ ] Daily monitoring reviews
- [ ] Bug triage and fixes
- [ ] Performance optimization
- [ ] Gather feedback from AU merchants

### Week 2-4 Post-Launch
- [ ] Analyze usage patterns
- [ ] Optimize based on metrics
- [ ] Plan for GB/ES regions
- [ ] Knowledge transfer sessions

### Ongoing
- [ ] Monthly health checks
- [ ] Quarterly performance reviews
- [ ] Continuous improvement
- [ ] Region expansion planning

---

## 📞 Communication Plan

### Stakeholder Updates
- **Weekly**: Status email to stakeholders
- **Bi-weekly**: Demo to product team
- **Sprint Review**: Progress presentation
- **Launch**: Go-live announcement

### Team Communication
- **Daily**: Standup updates
- **Weekly**: Technical sync
- **Ad-hoc**: Slack #gpay-au-project channel

### Escalation Path
1. Tech Lead → Engineering Manager
2. Engineering Manager → Director of Engineering
3. Director → VP Engineering

---

## ✅ Definition of Done

### Story-Level DoD
- [ ] Code complete and reviewed
- [ ] Unit tests written and passing (>80% coverage)
- [ ] Integration tests written and passing
- [ ] Documentation updated
- [ ] QA sign-off received
- [ ] No critical bugs
- [ ] Deployed to staging

### Epic-Level DoD
- [ ] All stories completed
- [ ] UAT passed
- [ ] Performance benchmarks met
- [ ] Monitoring in place
- [ ] Deployed to production
- [ ] 72-hour stability confirmed
- [ ] Post-deployment report completed
- [ ] Knowledge transfer complete

---

## 📝 Notes & Assumptions

### Assumptions
1. Afterpay AU API has feature parity with US API
2. No currency conversion needed (handled elsewhere)
3. OAuth configuration already supports AU
4. API Plus gateway is truly region-agnostic
5. Existing US merchants will not be affected

### Open Questions
- [ ] Are there AU-specific compliance requirements?
- [ ] Do we need separate monitoring for AU?
- [ ] Should we have AU-specific error messages?
- [ ] What's the expected AU merchant volume?
- [ ] Are there AU-specific product features needed?

### Decisions Made
- ✅ Use merchant-level region (not transaction-level)
- ✅ Region is immutable after merchant creation
- ✅ Default existing merchants to US
- ✅ Use feature flag for initial testing
- ✅ Blue-green deployment strategy

---

## 🎉 Success Celebration

Once all milestones are complete:
- Team celebration lunch
- Demo to wider engineering team
- Blog post about multi-region architecture
- Update team roadmap with GB/ES support

---

**Document Version:** 1.0  
**Last Updated:** March 25, 2026  
**Next Review:** After Phase 1 completion
