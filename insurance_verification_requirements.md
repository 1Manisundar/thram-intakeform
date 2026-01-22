# Insurance Verification System - Requirements Documentation

## Project Overview

**Goal:** Automate Insurance Eligibility Verification to remove manual admin work in healthcare workflow.

**Current Problem:** Admins manually review insurance documents and verify eligibility before contacting patients.

**Solution:** Automated verification system using n8n workflow automation, OCR, and clearinghouse API integration.

---

## User Journey

### 1. User Submission
- User fills form with personal details:
  - Name
  - Date of Birth (DOB)
  - Member ID
  - Payer Name
  - Email
  - Phone Number
- User uploads insurance card images (front/back)
- Form submission triggers workflow

### 2. Immediate User Feedback
- **Popup Message:** "Our agent will review and get back to you"
- User does not wait for processing
- Processing happens asynchronously in background

### 3. Automated Background Verification
- System verifies data automatically
- No admin involvement during verification
- Multiple validation steps with error handling

### 4. Admin Portal Interaction
- Admin sees verification results
- **If verified:** Admin contacts user to discuss next steps (appointments, visits)
- **If issues found:** Admin requests corrected data or handles edge cases
- **If insurance invalid:** Admin discusses cash payment or alternatives

---

## System Architecture

### Technology Stack
- **Workflow Automation:** n8n (self-hosted)
- **OCR Service:** Mindee
- **Clearinghouse API:** Stedi
- **Database:** PostgreSQL
- **Frontend:** Angular + TypeScript
- **Backend:** Node.js + Express + TypeScript (for Admin Portal APIs)

### Core Components
1. User-facing form (Angular frontend)
2. n8n workflow (automation engine)
3. PostgreSQL database (data storage)
4. Admin Portal (Angular frontend + Node.js/Express backend)
5. External integrations (Mindee OCR, Stedi API)

---

## Database Schema

### Table: `insurance_verifications`

```sql
CREATE TABLE insurance_verifications (
    id SERIAL PRIMARY KEY,
    submission_id VARCHAR(100) UNIQUE NOT NULL, -- UUID for idempotency
    
    -- User Data
    user_name VARCHAR(255) NOT NULL,
    date_of_birth DATE NOT NULL,
    member_id_submitted VARCHAR(100) NOT NULL,
    payer_name VARCHAR(255) NOT NULL,
    user_email VARCHAR(255),
    user_phone VARCHAR(20),
    
    -- Document URLs
    insurance_card_front_url TEXT,
    insurance_card_back_url TEXT,
    
    -- OCR Results
    member_id_extracted VARCHAR(100),
    payer_name_extracted VARCHAR(255),
    ocr_confidence_score DECIMAL(5,2),
    ocr_attempt_count INT DEFAULT 0,
    ocr_last_attempt_at TIMESTAMP,
    
    -- Stedi Verification Results
    stedi_eligibility_status VARCHAR(50), -- active, inactive, unknown
    stedi_plan_name VARCHAR(255),
    stedi_coverage_amount DECIMAL(10,2),
    stedi_copay DECIMAL(10,2),
    stedi_deductible DECIMAL(10,2),
    stedi_response_json JSONB, -- Full Stedi response for reference
    stedi_attempt_count INT DEFAULT 0,
    stedi_last_attempt_at TIMESTAMP,
    
    -- Status Management
    status VARCHAR(50) NOT NULL DEFAULT 'need_to_be_verified',
    verification_notes TEXT,
    requires_user_action BOOLEAN DEFAULT FALSE,
    user_action_message TEXT,
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    verified_at TIMESTAMP,
    admin_contacted_at TIMESTAMP,
    
    -- Admin tracking
    assigned_admin_id INT,
    admin_priority VARCHAR(20) DEFAULT 'normal' -- urgent, high, normal, low
);

CREATE INDEX idx_status ON insurance_verifications(status);
CREATE INDEX idx_requires_user_action ON insurance_verifications(requires_user_action);
CREATE INDEX idx_created_at ON insurance_verifications(created_at);
```

---

## Status State Machine

### Initial State
- **need_to_be_verified** - Form just submitted, awaiting processing

### Processing States
- **ocr_in_progress** - OCR extraction started
- **stedi_in_progress** - API call in progress
- **stedi_retrying** - Retrying failed API call

### Error States (User Action Required)
- **ocr_failed** - Need clearer images → `requires_user_action = true`
- **data_mismatch** - OCR data ≠ Form data → `requires_user_action = true`
- **awaiting_user_resubmission** - User notified, waiting for new data

### Error States (Admin Action Required)
- **api_error_manual_review** - Stedi API failed after retries
- **partial_verification** - Incomplete data from Stedi

### Admin Discussion Required
- **insurance_invalid** - Insurance not active (cash payment discussion needed)

### Success State
- **data_verified** - Everything passed → Admin ready to contact user

### Post-Admin States
- **admin_contacted** - Admin has reached out to user
- **appointment_scheduled** - Next steps arranged
- **closed** - Process complete

---

## n8n Workflow Overview

### Phase 1: Immediate Response (< 3 seconds)
1. **Webhook Trigger** - Receive form submission
2. **Database Insert** - Store data with status `need_to_be_verified`
3. **Respond to Webhook** - Return success message immediately
   - Message: "Our agent will review and get back to you"
   - Include `submission_id` for tracking

### Phase 2: Automated Verification (Async)
4. **Update Status** - Set to `ocr_in_progress`
5. **OCR Processing (Mindee)** - Extract data from insurance card
   - Extract: Member ID, Payer Name
   - Get confidence score
6. **OCR Validation**
   - Check confidence score ≥ 70%
   - **If failed:** Status = `ocr_failed`, notify user for clearer images
7. **Data Comparison**
   - Compare OCR Member ID vs User-submitted Member ID
   - **If mismatch:** Status = `data_mismatch`, notify user to reverify
8. **Update Status** - Set to `stedi_in_progress`
9. **Stedi API Call (with Retry Logic)**
   - Send eligibility verification request
   - Retry strategy: 30s → 5min → 15min (max 3 retries)
   - **If failed after retries:** Status = `api_error_manual_review`
10. **Parse Stedi Response**
    - Check eligibility status (active/inactive)
    - Extract: plan name, coverage amount, copay, deductible
    - **If incomplete data:** Status = `partial_verification`
    - **If insurance invalid:** Status = `insurance_invalid`
    - **If success:** Status = `data_verified`
11. **Final Database Update** - Store all verification results
12. **Admin Notification** - Alert admin dashboard for verified records

---

## Error Handling Strategy

### OCR Failures
**Scenario:** OCR can't read insurance card (blurry, poor quality, confidence < 70%)

**Action:**
- Status: `ocr_failed`
- Set `requires_user_action = true`
- Message: "Please upload clearer images of your insurance card"
- Send email/SMS notification to user
- Workflow STOPS

### Data Mismatch
**Scenario:** OCR-extracted Member ID ≠ User-entered Member ID

**Action:**
- Status: `data_mismatch`
- Set `requires_user_action = true`
- Message: "Data mismatch detected. Please verify: You entered [X], Card shows [Y]"
- Send notification to user
- Workflow STOPS

### Stedi API Issues
**Scenario:** API timeout, 500 errors, rate limits

**Action (Retry Strategy):**
1. **Attempt 1:** Immediate call
2. **Attempt 2:** Retry after 30 seconds (if failed)
3. **Attempt 3:** Retry after 5 minutes (if failed)
4. **Attempt 4:** Final retry after 15 minutes (if failed)
5. **After 3 failures:**
   - Status: `api_error_manual_review`
   - Admin manually triggers verification or checks directly
   - Priority: `high`

**Non-retryable errors (401, 403):**
- Immediate status: `api_error_manual_review`
- No retries

**404 (Member not found):**
- Status: `insurance_invalid`
- Admin handles cash payment discussion

### Insurance Not Active/Valid
**Scenario:** Stedi returns eligibility status = "inactive"

**Action:**
- Status: `insurance_invalid`
- Set `requires_user_action = false` (admin will handle)
- Admin contacts user to discuss cash payment or alternatives

### Partial Data from Stedi
**Scenario:** Stedi returns incomplete information (missing coverage amount, plan details, etc.)

**Action:**
- Status: `partial_verification`
- Store available data
- Admin reviews and may need to call insurance directly

---

## Data Validation Logic

### Member ID Comparison (n8n Code Node)
```javascript
// Normalize both IDs (remove spaces, special chars, uppercase)
const normalizeId = (id) => {
  return id.toString().trim().replace(/[^a-zA-Z0-9]/g, '').toUpperCase();
};

const ocrMemberId = normalizeId(ocrData.member_id);
const userMemberId = normalizeId(userData.member_id_submitted);

if (ocrMemberId === userMemberId) {
  // Data matches - proceed to Stedi verification
  return { validation_passed: true };
} else {
  // Mismatch - notify user
  return {
    validation_passed: false,
    status: 'data_mismatch',
    requires_user_action: true,
    user_action_message: `Data mismatch: Submitted [${userData.member_id_submitted}] vs Card [${ocrData.member_id}]`
  };
}
```

### OCR Confidence Check
```javascript
const minConfidence = 70; // 70% minimum

if (!ocrResult.member_id || ocrResult.confidence_score < minConfidence) {
  return {
    ocr_success: false,
    status: 'ocr_failed',
    requires_user_action: true,
    user_action_message: 'Image quality too low. Please upload clearer images.'
  };
}
```

### Stedi Response Parser
```javascript
const requiredFields = ['eligibility_status', 'plan_name', 'coverage_amount'];
const hasAllFields = requiredFields.every(field => stediResponse[field]);

if (stediResponse.eligibility_status === 'active') {
  if (hasAllFields) {
    status = 'data_verified';
  } else {
    status = 'partial_verification';
  }
} else if (stediResponse.eligibility_status === 'inactive') {
  status = 'insurance_invalid';
} else {
  status = 'api_error_manual_review';
}
```

---

## Admin Portal Requirements

### Dashboard Features
1. **Filtering**
   - By status (data_verified, insurance_invalid, api_error_manual_review, etc.)
   - By date range
   - By priority
   - By requires_user_action flag

2. **Sorting**
   - By created date
   - By priority
   - By status

3. **Search**
   - By user name
   - By member ID
   - By submission ID

### Individual Record View
**User Information Section:**
- Name, DOB, Contact Info
- Submitted Member ID
- Submitted Payer Name

**Verification Results Section:**
- Status Badge (color-coded by status)
- OCR Extracted Data (Member ID, Payer, confidence score)
- Stedi Verification Results:
  - Eligibility Status
  - Plan Name
  - Coverage Amount
  - Copay
  - Deductible
  - Full JSON Response (collapsible)
- Verification Notes
- Timestamps (submitted, verified, contacted)

**Documents Section:**
- Insurance Card Images (viewable inline)
- Download option

**Admin Actions:**
- Mark as "Contacted"
- Request New Data from User (trigger notification)
- Manual Re-trigger Verification
- Add Admin Notes
- Assign to Admin
- Change Priority

### Status Color Coding
- **Green:** data_verified
- **Blue:** Processing states (ocr_in_progress, stedi_in_progress)
- **Yellow:** partial_verification, insurance_invalid
- **Red:** ocr_failed, data_mismatch, api_error_manual_review
- **Orange:** stedi_retrying
- **Gray:** awaiting_user_resubmission, closed

---

## User Notification System

### Triggers for User Notifications

**OCR Failed:**
- Email/SMS: "We couldn't read your insurance card. Please upload clearer images."
- Include link to resubmit

**Data Mismatch:**
- Email/SMS: "Data mismatch detected. You entered [X], but card shows [Y]. Please verify and resubmit."
- Include both values for comparison

**Verification Complete (Optional):**
- Email: "Your insurance has been verified. Our team will contact you shortly."

### Notification Channels
- Email (primary)
- SMS (optional, for urgent cases)
- In-app notification (if user has account)

---

## API Integration Details

### Mindee OCR API
**Endpoint:** [Mindee API Documentation]
**Request:** Upload insurance card image
**Response:** Extracted fields with confidence scores
**Error Handling:** Retry once on timeout, fail otherwise

### Stedi Clearinghouse API
**Endpoint:** [Stedi API Documentation]
**Request:** Eligibility verification with member details
**Response:** Eligibility status, plan details, coverage info
**Error Handling:** Exponential backoff retry (3 attempts)

---

## HIPAA Compliance Requirements

### Infrastructure
- ✅ **Self-host n8n** (do NOT use n8n Cloud)
- ✅ Encrypt database at rest (PostgreSQL encryption)
- ✅ Use HTTPS for all webhooks and API calls
- ✅ SSL/TLS certificates for all endpoints

### Access Control
- ✅ Role-based access control for Admin Portal
- ✅ Audit logging (track who accessed what data, when)
- ✅ Multi-factor authentication for admin users
- ✅ Automatic session timeout

### Data Handling
- ✅ Encrypt PHI (Protected Health Information) in transit and at rest
- ✅ Regular automated backups with encryption
- ✅ Data retention policy (auto-delete after X months)
- ✅ Secure file storage for insurance card images

### Third-Party Compliance
- ✅ Sign BAA (Business Associate Agreement) with Mindee
- ✅ Sign BAA with Stedi
- ✅ Verify both vendors are HIPAA compliant

### Monitoring
- ✅ Log all data access events
- ✅ Alert on unauthorized access attempts
- ✅ Regular security audits

---

## Performance & Scalability

### Response Time Requirements
- **Webhook response:** < 3 seconds (immediate acknowledgment)
- **OCR processing:** < 30 seconds
- **Stedi API call:** < 10 seconds (excluding retries)
- **Total verification time:** < 2 minutes (happy path)

### Concurrent Processing
- n8n should handle multiple submissions simultaneously
- Database should support concurrent reads/writes
- Use database connection pooling

### Retry Strategy Summary
| Component | Initial Wait | Retry 1 | Retry 2 | Max Retries |
|-----------|--------------|---------|---------|-------------|
| OCR (Mindee) | N/A | No retry | N/A | 0 (fail immediately) |
| Stedi API | 30 seconds | 5 minutes | 15 minutes | 3 |

---

## Future Enhancements (Out of Scope for MVP)

- Multi-language support
- Batch processing for multiple users
- Analytics dashboard for admin (verification success rates, common errors)
- Integration with scheduling system for automatic appointment booking
- Patient portal for users to check verification status
- Support for additional insurance providers beyond Stedi
- Machine learning to improve OCR accuracy over time

---

## Success Metrics

### Key Performance Indicators (KPIs)
- **Automation Rate:** % of verifications completed without admin intervention
- **Verification Success Rate:** % of submissions that reach `data_verified` status
- **Average Processing Time:** Time from submission to verification
- **Error Rate by Type:** Track OCR failures, API errors, data mismatches
- **Admin Time Saved:** Hours saved vs manual verification process

### Target Goals
- **Automation Rate:** > 80%
- **Verification Success Rate:** > 90%
- **Average Processing Time:** < 2 minutes
- **OCR Failure Rate:** < 5%
- **Admin Time Saved:** 70% reduction

---

## Risk Assessment

### Technical Risks
1. **OCR Accuracy:** Poor image quality leads to high failure rate
   - **Mitigation:** Clear instructions for users on uploading images, image quality validation before submission

2. **API Dependency:** Stedi API downtime blocks verification
   - **Mitigation:** Retry logic, fallback to manual verification, monitor API status

3. **Data Privacy:** HIPAA violation due to improper handling
   - **Mitigation:** Comprehensive security audit, compliance review, regular training

### Operational Risks
1. **Admin Overload:** Too many manual review cases
   - **Mitigation:** Monitor error rates, improve OCR/validation logic, add filtering in admin portal

2. **User Frustration:** Too many requests for resubmission
   - **Mitigation:** Provide clear guidance, helpful error messages, live chat support option

---

## Glossary

- **OCR:** Optical Character Recognition - technology to extract text from images
- **Clearinghouse:** Third-party service that validates insurance eligibility with payers
- **Stedi:** Clearinghouse API provider for healthcare eligibility verification
- **Mindee:** OCR service provider
- **PHI:** Protected Health Information (HIPAA regulated data)
- **BAA:** Business Associate Agreement (required for HIPAA compliance)
- **Idempotency:** Ensuring duplicate submissions don't create duplicate records
- **Exponential Backoff:** Retry strategy with increasing wait times between attempts

---

## Document Version
- **Version:** 1.0
- **Last Updated:** January 21, 2026
- **Author:** Development Team
- **Status:** Draft - Pending Review