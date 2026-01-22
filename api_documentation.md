# API Documentation

## Overview
This document describes all REST API endpoints for the Insurance Verification System backend.

**Base URL (Development):** `http://localhost:3000/api`  
**Base URL (Production):** `https://your-domain.com/api`

**Authentication:** JWT Bearer tokens (except public endpoints)

---

## Table of Contents
1. [Authentication](#authentication)
2. [Verification Endpoints](#verification-endpoints)
3. [Admin Endpoints](#admin-endpoints)
4. [File Upload](#file-upload)
5. [Error Responses](#error-responses)

---

## Authentication

### Login
Authenticate admin user and receive JWT token.

**Endpoint:** `POST /api/auth/login`  
**Authentication:** None (public)

**Request Body:**
```json
{
  "email": "admin@example.com",
  "password": "admin123"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "admin": {
    "id": 1,
    "email": "admin@example.com",
    "name": "Admin User",
    "role": "admin"
  }
}
```

**Error Response (401):**
```json
{
  "success": false,
  "error": "Invalid email or password"
}
```

---

### Get Current User
Get currently authenticated admin information.

**Endpoint:** `GET /api/auth/me`  
**Authentication:** Required

**Headers:**
```
Authorization: Bearer <token>
```

**Success Response (200):**
```json
{
  "success": true,
  "admin": {
    "id": 1,
    "email": "admin@example.com",
    "name": "Admin User",
    "role": "admin",
    "lastLoginAt": "2026-01-22T10:30:00Z"
  }
}
```

---

### Logout
Logout admin user (client-side token removal).

**Endpoint:** `POST /api/auth/logout`  
**Authentication:** Required

**Success Response (200):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

---

### Forgot Password
Request password reset email.

**Endpoint:** `POST /api/auth/forgot-password`  
**Authentication:** None (public)

**Request Body:**
```json
{
  "email": "admin@example.com"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "Password reset email sent"
}
```

**Note:** Always returns success even if email doesn't exist (security best practice)

---

### Reset Password
Reset password using token from email.

**Endpoint:** `POST /api/auth/reset-password`  
**Authentication:** None (uses token from email)

**Request Body:**
```json
{
  "token": "reset-token-from-email",
  "newPassword": "newSecurePassword123"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "Password reset successfully"
}
```

---

## Verification Endpoints

### Submit Verification
User submits insurance verification request.

**Endpoint:** `POST /api/verification/submit`  
**Authentication:** None (public)

**Request Body:**
```json
{
  "userName": "John Doe",
  "dateOfBirth": "1985-03-15",
  "memberIdSubmitted": "ABC123456",
  "payerName": "Blue Cross Blue Shield",
  "userEmail": "john.doe@example.com",
  "userPhone": "555-0123",
  "insuranceCardFront": "base64EncodedImageData",
  "insuranceCardBack": "base64EncodedImageData"
}
```

**Success Response (201):**
```json
{
  "success": true,
  "message": "Our agent will review and get back to you",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Validation Error Response (400):**
```json
{
  "success": false,
  "error": "Validation failed",
  "details": [
    {
      "field": "userEmail",
      "message": "Invalid email format"
    }
  ]
}
```

---

### Get All Verifications
Get list of all verification submissions (admin only).

**Endpoint:** `GET /api/verifications`  
**Authentication:** Required

**Query Parameters:**
- `status` (optional): Filter by status
- `requiresUserAction` (optional): true/false
- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 20)
- `search` (optional): Search by name or member ID
- `sortBy` (optional): Field to sort by (default: createdAt)
- `sortOrder` (optional): asc/desc (default: desc)

**Example Request:**
```
GET /api/verifications?status=data_verified&page=1&limit=20
```

**Success Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "submissionId": "550e8400-e29b-41d4-a716-446655440000",
      "userName": "John Doe",
      "dateOfBirth": "1985-03-15",
      "memberIdSubmitted": "ABC123456",
      "payerName": "Blue Cross Blue Shield",
      "status": "data_verified",
      "requiresUserAction": false,
      "createdAt": "2026-01-22T10:00:00Z",
      "verifiedAt": "2026-01-22T10:05:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

---

### Get Single Verification
Get detailed information about a specific verification.

**Endpoint:** `GET /api/verifications/:id`  
**Authentication:** Required

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "submissionId": "550e8400-e29b-41d4-a716-446655440000",
    "userName": "John Doe",
    "dateOfBirth": "1985-03-15",
    "memberIdSubmitted": "ABC123456",
    "payerName": "Blue Cross Blue Shield",
    "userEmail": "john.doe@example.com",
    "userPhone": "555-0123",
    "insuranceCardFrontUrl": "/uploads/front-abc123.jpg",
    "insuranceCardBackUrl": "/uploads/back-abc123.jpg",
    "memberIdExtracted": "ABC123456",
    "payerNameExtracted": "BCBS",
    "ocrConfidenceScore": 95.5,
    "stediEligibilityStatus": "active",
    "stediPlanName": "PPO Gold Plan",
    "stediCoverageAmount": 5000.00,
    "stediCopay": 25.00,
    "stediDeductible": 1000.00,
    "stediResponseJson": { /* full Stedi response */ },
    "status": "data_verified",
    "verificationNotes": "All verified, ready to contact",
    "requiresUserAction": false,
    "createdAt": "2026-01-22T10:00:00Z",
    "updatedAt": "2026-01-22T10:05:00Z",
    "verifiedAt": "2026-01-22T10:05:00Z",
    "adminContactedAt": null,
    "assignedAdminId": null,
    "adminPriority": "normal"
  }
}
```

**Error Response (404):**
```json
{
  "success": false,
  "error": "Verification not found"
}
```

---

### Update Verification Status
Update the status of a verification.

**Endpoint:** `PUT /api/verifications/:id/status`  
**Authentication:** Required

**Request Body:**
```json
{
  "status": "admin_contacted",
  "verificationNotes": "Called patient, appointment scheduled"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "Status updated successfully",
  "data": {
    "id": 1,
    "status": "admin_contacted",
    "verificationNotes": "Called patient, appointment scheduled",
    "updatedAt": "2026-01-22T14:30:00Z"
  }
}
```

---

### Mark as Contacted
Mark verification as contacted by admin.

**Endpoint:** `PUT /api/verifications/:id/contact`  
**Authentication:** Required

**Request Body:**
```json
{
  "notes": "Spoke with patient, appointment set for next Tuesday"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "Marked as contacted",
  "data": {
    "id": 1,
    "adminContactedAt": "2026-01-22T14:30:00Z",
    "verificationNotes": "Spoke with patient, appointment set for next Tuesday"
  }
}
```

---

### Add Admin Note
Add a note to a verification.

**Endpoint:** `POST /api/verifications/:id/notes`  
**Authentication:** Required

**Request Body:**
```json
{
  "note": "Patient confirmed coverage details"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "Note added successfully",
  "data": {
    "id": 1,
    "verificationNotes": "Patient confirmed coverage details",
    "updatedAt": "2026-01-22T14:35:00Z"
  }
}
```

---

### Manually Trigger Verification
Manually re-trigger the n8n verification workflow.

**Endpoint:** `POST /api/verifications/:id/trigger`  
**Authentication:** Required

**Success Response (200):**
```json
{
  "success": true,
  "message": "Verification workflow triggered",
  "data": {
    "submissionId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "ocr_in_progress"
  }
}
```

---

### Get Verification Statistics
Get statistics for dashboard.

**Endpoint:** `GET /api/verifications/stats`  
**Authentication:** Required

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "total": 150,
    "byStatus": {
      "need_to_be_verified": 5,
      "ocr_in_progress": 2,
      "stedi_in_progress": 1,
      "data_verified": 120,
      "ocr_failed": 8,
      "data_mismatch": 4,
      "insurance_invalid": 10
    },
    "requiresUserAction": 12,
    "requiresAdminReview": 5,
    "today": 8,
    "thisWeek": 42,
    "thisMonth": 150
  }
}
```

---

## Admin Endpoints

### Get All Admins
Get list of all admin users (super_admin only).

**Endpoint:** `GET /api/admins`  
**Authentication:** Required (super_admin role)

**Success Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "email": "admin@example.com",
      "name": "Admin User",
      "role": "admin",
      "isActive": true,
      "createdAt": "2026-01-01T00:00:00Z",
      "lastLoginAt": "2026-01-22T10:00:00Z"
    }
  ]
}
```

---

### Create Admin
Create new admin user (super_admin only).

**Endpoint:** `POST /api/admins`  
**Authentication:** Required (super_admin role)

**Request Body:**
```json
{
  "email": "newadmin@example.com",
  "password": "securePassword123",
  "name": "New Admin",
  "role": "admin"
}
```

**Success Response (201):**
```json
{
  "success": true,
  "message": "Admin created successfully",
  "data": {
    "id": 2,
    "email": "newadmin@example.com",
    "name": "New Admin",
    "role": "admin",
    "isActive": true
  }
}
```

---

### Update Admin
Update admin user information.

**Endpoint:** `PUT /api/admins/:id`  
**Authentication:** Required (super_admin role)

**Request Body:**
```json
{
  "name": "Updated Name",
  "isActive": false
}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "Admin updated successfully",
  "data": {
    "id": 2,
    "email": "newadmin@example.com",
    "name": "Updated Name",
    "isActive": false
  }
}
```

---

### Delete Admin
Deactivate admin user (soft delete).

**Endpoint:** `DELETE /api/admins/:id`  
**Authentication:** Required (super_admin role)

**Success Response (200):**
```json
{
  "success": true,
  "message": "Admin deactivated successfully"
}
```

---

## File Upload

### Upload Insurance Card
Upload insurance card image files.

**Endpoint:** `POST /api/upload/insurance-card`  
**Authentication:** None (public, used during form submission)

**Request:** Multipart form data
```
Content-Type: multipart/form-data

Fields:
- frontImage: File (image/jpeg, image/png)
- backImage: File (image/jpeg, image/png)
```

**Success Response (200):**
```json
{
  "success": true,
  "files": {
    "frontUrl": "/uploads/insurance-front-abc123.jpg",
    "backUrl": "/uploads/insurance-back-abc123.jpg"
  }
}
```

**Error Response (400):**
```json
{
  "success": false,
  "error": "File size exceeds 5MB limit"
}
```

**Allowed File Types:**
- image/jpeg
- image/jpg
- image/png

**Max File Size:** 5MB per file

---

## Webhook Endpoints (For n8n)

### Verification Status Update
n8n calls this to update verification status.

**Endpoint:** `POST /api/webhook/verification-update`  
**Authentication:** Webhook secret key

**Headers:**
```
X-Webhook-Secret: <webhook_secret>
```

**Request Body:**
```json
{
  "submissionId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "data_verified",
  "ocrResults": {
    "memberIdExtracted": "ABC123456",
    "payerNameExtracted": "BCBS",
    "confidenceScore": 95.5
  },
  "stediResults": {
    "eligibilityStatus": "active",
    "planName": "PPO Gold Plan",
    "coverageAmount": 5000.00,
    "copay": 25.00,
    "deductible": 1000.00,
    "fullResponse": { /* complete Stedi response */ }
  }
}
```

**Success Response (200):**
```json
{
  "success": true,
  "message": "Verification updated successfully"
}
```

---

## Error Responses

### Standard Error Format
```json
{
  "success": false,
  "error": "Error message here",
  "code": "ERROR_CODE"
}
```

### HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET, PUT, DELETE |
| 201 | Created | Successful POST (resource created) |
| 400 | Bad Request | Validation error, malformed request |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but lacks permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource (e.g., email exists) |
| 422 | Unprocessable Entity | Business logic validation failed |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |
| 503 | Service Unavailable | Temporary unavailability |

### Common Error Codes

| Error Code | Description |
|------------|-------------|
| `VALIDATION_ERROR` | Input validation failed |
| `AUTHENTICATION_FAILED` | Invalid credentials |
| `TOKEN_EXPIRED` | JWT token expired |
| `TOKEN_INVALID` | Invalid JWT token |
| `UNAUTHORIZED` | Not authenticated |
| `FORBIDDEN` | Insufficient permissions |
| `NOT_FOUND` | Resource not found |
| `DUPLICATE_EMAIL` | Email already exists |
| `FILE_TOO_LARGE` | Upload exceeds size limit |
| `INVALID_FILE_TYPE` | Unsupported file format |
| `RATE_LIMIT_EXCEEDED` | Too many requests |
| `INTERNAL_ERROR` | Server error |

### Example Error Responses

**Validation Error:**
```json
{
  "success": false,
  "error": "Validation failed",
  "code": "VALIDATION_ERROR",
  "details": [
    {
      "field": "userEmail",
      "message": "Invalid email format"
    },
    {
      "field": "memberIdSubmitted",
      "message": "Member ID is required"
    }
  ]
}
```

**Authentication Error:**
```json
{
  "success": false,
  "error": "Authentication token is missing or invalid",
  "code": "TOKEN_INVALID"
}
```

**Rate Limit Error:**
```json
{
  "success": false,
  "error": "Too many requests. Please try again in 60 seconds",
  "code": "RATE_LIMIT_EXCEEDED",
  "retryAfter": 60
}
```

---

## Rate Limiting

### Limits
- **Public endpoints:** 100 requests per hour per IP
- **Authenticated endpoints:** 1000 requests per hour per admin
- **File uploads:** 20 uploads per hour per IP

### Headers
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642867200
```

---

## Pagination

All list endpoints support pagination:

**Query Parameters:**
- `page`: Page number (starts at 1)
- `limit`: Items per page (default: 20, max: 100)

**Response Format:**
```json
{
  "success": true,
  "data": [ /* items */ ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

---

## Filtering & Sorting

### Filter Examples
```
GET /api/verifications?status=data_verified
GET /api/verifications?requiresUserAction=true
GET /api/verifications?status=data_verified&page=2
```

### Search
```
GET /api/verifications?search=John%20Doe
```

### Sorting
```
GET /api/verifications?sortBy=createdAt&sortOrder=desc
GET /api/verifications?sortBy=userName&sortOrder=asc
```

### Combined
```
GET /api/verifications?status=data_verified&sortBy=createdAt&sortOrder=desc&page=1&limit=20
```

---

## Authentication Flow

### 1. Login
```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@example.com",
    "password": "admin123"
  }'
```

**Response:**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### 2. Use Token in Requests
```bash
curl -X GET http://localhost:3000/api/verifications \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### 3. Token Expiration
Tokens expire after 7 days (configurable in `.env`).

When expired, client receives:
```json
{
  "success": false,
  "error": "Token expired",
  "code": "TOKEN_EXPIRED"
}
```

Client should redirect to login page.

---

## CORS Configuration

### Development
```
Access-Control-Allow-Origin: http://localhost:4200
```

### Production
```
Access-Control-Allow-Origin: https://your-frontend-domain.com
```

### Allowed Methods
- GET
- POST
- PUT
- DELETE
- OPTIONS

### Allowed Headers
- Content-Type
- Authorization
- X-Requested-With

---

## Testing with cURL

### Submit Verification
```bash
curl -X POST http://localhost:3000/api/verification/submit \
  -H "Content-Type: application/json" \
  -d '{
    "userName": "John Doe",
    "dateOfBirth": "1985-03-15",
    "memberIdSubmitted": "ABC123456",
    "payerName": "Blue Cross Blue Shield",
    "userEmail": "john@example.com",
    "userPhone": "555-0123"
  }'
```

### Get Verifications (with auth)
```bash
curl -X GET "http://localhost:3000/api/verifications?status=data_verified" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

### Update Status
```bash
curl -X PUT http://localhost:3000/api/verifications/1/status \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "admin_contacted",
    "verificationNotes": "Called patient"
  }'
```

---

## Postman Collection

A Postman collection is available for easy API testing:

**Location:** `/docs/postman/insurance-verification-api.json`

**Import Steps:**
1. Open Postman
2. Click "Import"
3. Select the JSON file
4. Configure environment variables:
   - `baseUrl`: http://localhost:3000/api
   - `token`: Your JWT token

---

## Versioning

**Current Version:** v1

API version is included in the base URL (future):
- `http://localhost:3000/api/v1/...`

For now, all endpoints use:
- `http://localhost:3000/api/...`

---

## Document Information
- **Version:** 1.0
- **Last Updated:** January 22, 2026
- **API Version:** v1
- **Status:** Complete