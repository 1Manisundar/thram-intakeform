# n8n Workflows Documentation

## Overview
This document describes the n8n workflow automation for the Insurance Verification System.

**n8n Access:** http://localhost:5678 (development)  
**Workflow Name:** Insurance Verification Automation  
**Trigger:** Webhook from Express Backend

---

## Workflow Architecture

```
Webhook Trigger (Express Backend)
        ↓
┌───────────────────────────────┐
│  Step 1: Update Status        │
│  "ocr_in_progress"            │
└───────┬───────────────────────┘
        ↓
┌───────────────────────────────┐
│  Step 2: OCR Processing       │
│  (Mindee API)                 │
└───────┬───────────────────────┘
        ↓
    [Decision]
   OCR Success?
    /        \
  Yes        No
   │          │
   │          └──► Update Status "ocr_failed"
   │               Notify User
   │               STOP
   ↓
┌───────────────────────────────┐
│  Step 3: Validate Data        │
│  (Code Node)                  │
└───────┬───────────────────────┘
        ↓
    [Decision]
   Data Match?
    /        \
  Yes        No
   │          │
   │          └──► Update Status "data_mismatch"
   │               Notify User
   │               STOP
   ↓
┌───────────────────────────────┐
│  Step 4: Update Status        │
│  "stedi_in_progress"          │
└───────┬───────────────────────┘
        ↓
┌───────────────────────────────┐
│  Step 5: Stedi API Call       │
│  (with Retry Logic)           │
└───────┬───────────────────────┘
        ↓
    [Decision]
   API Success?
    /        \
  Yes        No
   │          │
   │          └──► Retry Logic (3 attempts)
   │               If all fail: "api_error_manual_review"
   │               STOP
   ↓
┌───────────────────────────────┐
│  Step 6: Parse Stedi Response │
│  (Code Node)                  │
└───────┬───────────────────────┘
        ↓
┌───────────────────────────────┐
│  Step 7: Update Database      │
│  Final Status & Results       │
└───────┬───────────────────────┘
        ↓
┌───────────────────────────────┐
│  Step 8: Notify Admin         │
│  (if verified successfully)   │
└───────────────────────────────┘
```

---

## Workflow Nodes Configuration

### Node 1: Webhook Trigger

**Type:** Webhook  
**Method:** POST  
**Path:** `insurance-verification`  
**Authentication:** None (secured via backend)

**Expected Payload:**
```json
{
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "userName": "John Doe",
  "dateOfBirth": "1985-03-15",
  "memberIdSubmitted": "ABC123456",
  "payerName": "Blue Cross Blue Shield",
  "insuranceCardFrontUrl": "/uploads/front-abc123.jpg",
  "insuranceCardBackUrl": "/uploads/back-abc123.jpg"
}
```

**Node Settings:**
```javascript
{
  "httpMethod": "POST",
  "path": "insurance-verification",
  "responseMode": "responseNode",
  "options": {}
}
```

---

### Node 2: Respond to Webhook

**Type:** Respond to Webhook  
**Purpose:** Immediate acknowledgment to backend

**Response:**
```json
{
  "success": true,
  "message": "Verification workflow started",
  "submissionId": "{{$json.submissionId}}"
}
```

**Node Settings:**
```javascript
{
  "options": {
    "responseCode": 200,
    "responseHeaders": {
      "Content-Type": "application/json"
    }
  }
}
```

---

### Node 3: Update Status (OCR In Progress)

**Type:** Postgres  
**Operation:** Update  
**Table:** insurance_verifications

**SQL:**
```sql
UPDATE insurance_verifications
SET 
  status = 'ocr_in_progress',
  ocr_attempt_count = ocr_attempt_count + 1,
  ocr_last_attempt_at = NOW(),
  updated_at = NOW()
WHERE submission_id = $1
RETURNING *
```

**Parameters:**
```javascript
[
  "{{$json.submissionId}}"
]
```

---

### Node 4: HTTP Request - Mindee OCR

**Type:** HTTP Request  
**Method:** POST  
**URL:** `https://api.mindee.net/v1/products/mindee/insurance_card/v1/predict`

**Headers:**
```javascript
{
  "Authorization": "Token {{$env.MINDEE_API_KEY}}",
  "Content-Type": "multipart/form-data"
}
```

**Body (Multipart Form):**
```javascript
{
  "document": {
    "type": "file",
    "url": "{{$node.Webhook.json.insuranceCardFrontUrl}}"
  }
}
```

**Expected Response:**
```json
{
  "document": {
    "inference": {
      "prediction": {
        "member_id": {
          "value": "ABC123456",
          "confidence": 0.95
        },
        "payer_name": {
          "value": "Blue Cross Blue Shield",
          "confidence": 0.92
        }
      }
    }
  }
}
```

**Error Handling:**
- Continue on Fail: No
- Retry on Fail: No (handled in next node)

---

### Node 5: Code - Validate OCR Results

**Type:** Code (JavaScript)  
**Purpose:** Check OCR confidence and extract data

```javascript
// Get OCR response
const ocrResponse = $input.item.json;
const webhookData = $node["Webhook"].json;

// Minimum confidence threshold
const MIN_CONFIDENCE = 0.70; // 70%

try {
  const prediction = ocrResponse.document.inference.prediction;
  
  // Extract data
  const memberIdData = prediction.member_id;
  const payerNameData = prediction.payer_name;
  
  // Check if data exists
  if (!memberIdData || !memberIdData.value) {
    return {
      json: {
        ocrSuccess: false,
        status: 'ocr_failed',
        requiresUserAction: true,
        userActionMessage: 'Unable to read insurance card. Please upload a clearer image.',
        verificationNotes: 'OCR failed to extract member ID',
        submissionId: webhookData.submissionId
      }
    };
  }
  
  // Check confidence score
  const confidence = memberIdData.confidence;
  if (confidence < MIN_CONFIDENCE) {
    return {
      json: {
        ocrSuccess: false,
        status: 'ocr_failed',
        requiresUserAction: true,
        userActionMessage: `Image quality too low (confidence: ${(confidence * 100).toFixed(1)}%). Please upload clearer images.`,
        verificationNotes: `Low OCR confidence: ${(confidence * 100).toFixed(1)}%`,
        submissionId: webhookData.submissionId,
        ocrConfidenceScore: (confidence * 100).toFixed(2)
      }
    };
  }
  
  // Success - return extracted data
  return {
    json: {
      ocrSuccess: true,
      memberIdExtracted: memberIdData.value,
      payerNameExtracted: payerNameData?.value || null,
      ocrConfidenceScore: (confidence * 100).toFixed(2),
      submissionId: webhookData.submissionId
    }
  };
  
} catch (error) {
  return {
    json: {
      ocrSuccess: false,
      status: 'ocr_failed',
      requiresUserAction: true,
      userActionMessage: 'Error processing insurance card image. Please try again.',
      verificationNotes: `OCR processing error: ${error.message}`,
      submissionId: webhookData.submissionId
    }
  };
}
```

---

### Node 6: IF - OCR Success Check

**Type:** IF  
**Condition:** Check if OCR was successful

**Condition:**
```javascript
{{$json.ocrSuccess}} === true
```

**Routes:**
- **True:** Continue to data validation
- **False:** Go to update database with error status

---

### Node 7: Update Database - OCR Failed

**Type:** Postgres  
**Operation:** Update  
**Condition:** Only runs if OCR failed

**SQL:**
```sql
UPDATE insurance_verifications
SET 
  status = $1,
  requires_user_action = $2,
  user_action_message = $3,
  verification_notes = $4,
  ocr_confidence_score = $5,
  updated_at = NOW()
WHERE submission_id = $6
RETURNING *
```

**Parameters:**
```javascript
[
  "{{$json.status}}",
  "{{$json.requiresUserAction}}",
  "{{$json.userActionMessage}}",
  "{{$json.verificationNotes}}",
  "{{$json.ocrConfidenceScore}}",
  "{{$json.submissionId}}"
]
```

---

### Node 8: Code - Validate Data Match

**Type:** Code (JavaScript)  
**Purpose:** Compare OCR extracted data with user submitted data

```javascript
const ocrData = $input.item.json;
const webhookData = $node["Webhook"].json;

// Normalize function to remove spaces, special chars, and uppercase
function normalizeId(id) {
  if (!id) return '';
  return id.toString().trim().replace(/[^a-zA-Z0-9]/g, '').toUpperCase();
}

const ocrMemberId = normalizeId(ocrData.memberIdExtracted);
const userMemberId = normalizeId(webhookData.memberIdSubmitted);

if (ocrMemberId === userMemberId) {
  // Data matches - proceed to Stedi verification
  return {
    json: {
      validationPassed: true,
      memberIdExtracted: ocrData.memberIdExtracted,
      payerNameExtracted: ocrData.payerNameExtracted,
      ocrConfidenceScore: ocrData.ocrConfidenceScore,
      submissionId: webhookData.submissionId
    }
  };
} else {
  // Mismatch - notify user
  return {
    json: {
      validationPassed: false,
      status: 'data_mismatch',
      requiresUserAction: true,
      userActionMessage: `Data mismatch detected:
        You entered: ${webhookData.memberIdSubmitted}
        Found on card: ${ocrData.memberIdExtracted}
        Please verify and resubmit.`,
      verificationNotes: `Member ID mismatch: Submitted [${webhookData.memberIdSubmitted}] vs OCR [${ocrData.memberIdExtracted}]`,
      memberIdExtracted: ocrData.memberIdExtracted,
      ocrConfidenceScore: ocrData.ocrConfidenceScore,
      submissionId: webhookData.submissionId
    }
  };
}
```

---

### Node 9: IF - Validation Check

**Type:** IF  
**Condition:** Check if data validation passed

**Condition:**
```javascript
{{$json.validationPassed}} === true
```

**Routes:**
- **True:** Continue to Stedi API
- **False:** Go to update database with mismatch status

---

### Node 10: Update Database - Data Mismatch

**Type:** Postgres  
**Operation:** Update  
**Condition:** Only runs if validation failed

**SQL:**
```sql
UPDATE insurance_verifications
SET 
  status = $1,
  requires_user_action = $2,
  user_action_message = $3,
  verification_notes = $4,
  member_id_extracted = $5,
  ocr_confidence_score = $6,
  updated_at = NOW()
WHERE submission_id = $7
RETURNING *
```

---

### Node 11: Update Status (Stedi In Progress)

**Type:** Postgres  
**Operation:** Update

**SQL:**
```sql
UPDATE insurance_verifications
SET 
  status = 'stedi_in_progress',
  stedi_attempt_count = stedi_attempt_count + 1,
  stedi_last_attempt_at = NOW(),
  member_id_extracted = $1,
  payer_name_extracted = $2,
  ocr_confidence_score = $3,
  updated_at = NOW()
WHERE submission_id = $4
RETURNING *
```

---

### Node 12: HTTP Request - Stedi API (with Retry)

**Type:** HTTP Request  
**Method:** POST  
**URL:** `https://api.stedi.com/2024-01-01/eligibility`

**Headers:**
```javascript
{
  "Authorization": "Key {{$env.STEDI_API_KEY}}",
  "Content-Type": "application/json"
}
```

**Body:**
```javascript
{
  "memberId": "{{$node['Code - Validate Data Match'].json.memberIdExtracted}}",
  "payerName": "{{$node['Webhook'].json.payerName}}",
  "dateOfBirth": "{{$node['Webhook'].json.dateOfBirth}}",
  "serviceDate": "{{new Date().toISOString().split('T')[0]}}"
}
```

**Retry Settings:**
- Retry on Fail: Yes
- Max Tries: 4 (1 initial + 3 retries)
- Wait Between Tries (ms):
  - Try 1: 0 (immediate)
  - Try 2: 30000 (30 seconds)
  - Try 3: 300000 (5 minutes)
  - Try 4: 900000 (15 minutes)

**Expected Response:**
```json
{
  "eligibility": {
    "status": "active",
    "planName": "PPO Gold Plan",
    "coverageAmount": 5000.00,
    "copay": 25.00,
    "deductible": 1000.00,
    "effectiveDate": "2024-01-01",
    "terminationDate": null
  }
}
```

---

### Node 13: Code - Parse Stedi Response

**Type:** Code (JavaScript)  
**Purpose:** Parse and validate Stedi response

```javascript
const stediResponse = $input.item.json;
const webhookData = $node["Webhook"].json;

// Required fields for complete verification
const requiredFields = ['status', 'planName', 'coverageAmount'];

try {
  const eligibility = stediResponse.eligibility || stediResponse;
  
  // Check if all required fields exist
  const hasAllFields = requiredFields.every(field => 
    eligibility[field] !== null && 
    eligibility[field] !== undefined
  );
  
  let status, verificationNotes;
  
  if (eligibility.status === 'active' || eligibility.status === 'Active') {
    if (hasAllFields) {
      status = 'data_verified';
      verificationNotes = 'Full verification completed successfully';
    } else {
      status = 'partial_verification';
      const missingFields = requiredFields.filter(field => !eligibility[field]);
      verificationNotes = `Missing fields: ${missingFields.join(', ')}`;
    }
  } else if (eligibility.status === 'inactive' || eligibility.status === 'Inactive') {
    status = 'insurance_invalid';
    verificationNotes = 'Insurance policy is not active';
  } else {
    status = 'api_error_manual_review';
    verificationNotes = `Unknown eligibility status from Stedi: ${eligibility.status}`;
  }
  
  return {
    json: {
      status,
      verificationNotes,
      stediEligibilityStatus: eligibility.status,
      stediPlanName: eligibility.planName || null,
      stediCoverageAmount: eligibility.coverageAmount || null,
      stediCopay: eligibility.copay || null,
      stediDeductible: eligibility.deductible || null,
      stediResponseJson: stediResponse,
      verifiedAt: status === 'data_verified' ? new Date().toISOString() : null,
      submissionId: webhookData.submissionId
    }
  };
  
} catch (error) {
  return {
    json: {
      status: 'api_error_manual_review',
      verificationNotes: `Error parsing Stedi response: ${error.message}`,
      stediResponseJson: stediResponse,
      submissionId: webhookData.submissionId
    }
  };
}
```

---

### Node 14: Update Database - Final Status

**Type:** Postgres  
**Operation:** Update

**SQL:**
```sql
UPDATE insurance_verifications
SET 
  status = $1,
  verification_notes = $2,
  stedi_eligibility_status = $3,
  stedi_plan_name = $4,
  stedi_coverage_amount = $5,
  stedi_copay = $6,
  stedi_deductible = $7,
  stedi_response_json = $8,
  verified_at = $9,
  updated_at = NOW()
WHERE submission_id = $10
RETURNING *
```

**Parameters:**
```javascript
[
  "{{$json.status}}",
  "{{$json.verificationNotes}}",
  "{{$json.stediEligibilityStatus}}",
  "{{$json.stediPlanName}}",
  "{{$json.stediCoverageAmount}}",
  "{{$json.stediCopay}}",
  "{{$json.stediDeductible}}",
  "{{JSON.stringify($json.stediResponseJson)}}",
  "{{$json.verifiedAt}}",
  "{{$json.submissionId}}"
]
```

---

### Node 15: IF - Check if Verified

**Type:** IF  
**Condition:** Check if verification was successful

**Condition:**
```javascript
{{$json.status}} === 'data_verified' || {{$json.status}} === 'partial_verification'
```

**Routes:**
- **True:** Notify admin
- **False:** End workflow

---

### Node 16: Send Admin Notification (Optional)

**Type:** HTTP Request  
**Method:** POST  
**URL:** `http://backend:3000/api/internal/notify-admin`

**Body:**
```javascript
{
  "submissionId": "{{$json.submissionId}}",
  "status": "{{$json.status}}",
  "message": "New verification ready for review"
}
```

---

## Error Handling Strategy

### OCR Errors

**Error Type:** API timeout, low confidence, extraction failure

**Action:**
1. Update status to `ocr_failed`
2. Set `requires_user_action = true`
3. Store error message in `user_action_message`
4. Send email notification to user
5. **STOP workflow**

---

### Data Mismatch Errors

**Error Type:** OCR data ≠ User submitted data

**Action:**
1. Update status to `data_mismatch`
2. Set `requires_user_action = true`
3. Store comparison in `user_action_message`
4. Send email notification to user
5. **STOP workflow**

---

### Stedi API Errors

**Error Type:** Timeout, 500 errors, rate limits

**Retry Logic:**
1. **Attempt 1:** Immediate call
2. **Wait 30 seconds** → Attempt 2
3. **Wait 5 minutes** → Attempt 3
4. **Wait 15 minutes** → Attempt 4 (final)
5. If all fail → Update status to `api_error_manual_review`

**Non-Retryable Errors (401, 403):**
- Update status immediately to `api_error_manual_review`
- Set admin_priority to `high`
- **STOP workflow**

**404 - Member Not Found:**
- Update status to `insurance_invalid`
- **STOP workflow**

---

## Environment Variables (n8n)

Set these in n8n environment or workflow settings:

```env
MINDEE_API_KEY=your_mindee_api_key
STEDI_API_KEY=your_stedi_api_key
BACKEND_URL=http://backend:3000
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=insurance_verification_dev
DATABASE_USER=dev_user
DATABASE_PASSWORD=dev_password
```

---

## Workflow Export/Import

### Export Workflow
1. Open n8n at http://localhost:5678
2. Open the workflow
3. Click "..." menu → Download
4. Save as `insurance-verification-workflow.json`

### Import Workflow
1. Open n8n
2. Click "+ Add workflow"
3. Click "..." menu → Import from File
4. Select `insurance-verification-workflow.json`

---

## Testing the Workflow

### Test in n8n UI

1. Open workflow in n8n
2. Click "Execute Workflow"
3. Use "Webhook" node test URL
4. Send POST request with sample data:

```bash
curl -X POST http://localhost:5678/webhook-test/insurance-verification \
  -H "Content-Type: application/json" \
  -d '{
    "submissionId": "test-123",
    "userName": "Test User",
    "dateOfBirth": "1985-01-01",
    "memberIdSubmitted": "TEST123",
    "payerName": "Test Insurance",
    "insuranceCardFrontUrl": "http://example.com/test.jpg"
  }'
```

5. Watch execution in n8n UI
6. Check each node's output

---

### Test Error Scenarios

**Test OCR Failure:**
- Use invalid image URL
- Expected: Status = `ocr_failed`

**Test Data Mismatch:**
- Modify OCR code to return different Member ID
- Expected: Status = `data_mismatch`

**Test API Timeout:**
- Use invalid Stedi API key temporarily
- Expected: Retry 3 times, then status = `api_error_manual_review`

---

## Monitoring & Logging

### Enable Workflow Logging
In n8n settings:
- Enable "Log Level: debug"
- Enable "Save Execution Progress"

### View Execution History
1. Go to "Executions" tab in n8n
2. View all workflow runs
3. Click on execution to see detailed logs
4. Check node outputs and errors

### Database Logging
All status changes are logged in PostgreSQL:
- `ocr_attempt_count` tracks OCR tries
- `stedi_attempt_count` tracks API tries
- `updated_at` shows last modification
- `verification_notes` stores error details

---

## Performance Optimization

### Reduce Processing Time
- Use CDN for uploaded images (faster OCR access)
- Enable Stedi API caching (if available)
- Optimize database queries with indexes

### Handle High Volume
- Use n8n queue mode for concurrent workflows
- Scale n8n with multiple instances
- Use PostgreSQL connection pooling

---

## Workflow Maintenance

### Regular Tasks
- Review failed executions weekly
- Update retry timing based on API performance
- Monitor OCR confidence trends
- Check Stedi API response times

### Updates
- Keep n8n version updated
- Update Mindee/Stedi API integrations
- Adjust confidence thresholds based on accuracy
- Refine error messages based on user feedback

---

## Document Information
- **Version:** 1.0
- **Last Updated:** January 22, 2026
- **Workflow Version:** 1.0
- **Status:** Complete