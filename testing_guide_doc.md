# Testing Guide

## Overview
This document outlines testing strategies and procedures for the Insurance Verification System.

**Testing Types:**
- Unit Testing
- Integration Testing
- End-to-End (E2E) Testing
- Manual Testing
- Security Testing

---

## Testing Setup

### Install Testing Dependencies

**Backend:**
```bash
cd backend

# Install testing frameworks
npm install --save-dev jest @types/jest ts-jest
npm install --save-dev supertest @types/supertest
npm install --save-dev @faker-js/faker

# Create Jest config
npx ts-jest config:init
```

**Frontend:**
```bash
cd frontend

# Angular includes testing by default
# Jasmine + Karma for unit tests
# Protractor/Playwright for E2E
npm install --save-dev @playwright/test
```

---

## Unit Testing

### Backend Unit Tests

#### Test Structure
```typescript
// src/__tests__/services/verification.service.test.ts
import { VerificationService } from '../../services/verification.service';
import { PrismaClient } from '@prisma/client';

// Mock Prisma
jest.mock('@prisma/client');

describe('VerificationService', () => {
  let service: VerificationService;
  let prisma: PrismaClient;

  beforeEach(() => {
    prisma = new PrismaClient();
    service = new VerificationService(prisma);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('createVerification', () => {
    it('should create verification with correct data', async () => {
      const mockData = {
        userName: 'John Doe',
        dateOfBirth: '1985-03-15',
        memberIdSubmitted: 'ABC123',
        payerName: 'Blue Cross',
        userEmail: 'john@example.com'
      };

      const mockResult = {
        id: 1,
        submissionId: 'test-uuid',
        ...mockData,
        status: 'need_to_be_verified',
        createdAt: new Date()
      };

      (prisma.insuranceVerification.create as jest.Mock).mockResolvedValue(mockResult);

      const result = await service.createVerification(mockData);

      expect(result.submissionId).toBeDefined();
      expect(result.status).toBe('need_to_be_verified');
      expect(prisma.insuranceVerification.create).toHaveBeenCalledTimes(1);
    });

    it('should throw error for invalid email', async () => {
      const invalidData = {
        userName: 'John Doe',
        dateOfBirth: '1985-03-15',
        memberIdSubmitted: 'ABC123',
        payerName: 'Blue Cross',
        userEmail: 'invalid-email'
      };

      await expect(service.createVerification(invalidData))
        .rejects
        .toThrow('Invalid email format');
    });
  });

  describe('getVerificationById', () => {
    it('should return verification when found', async () => {
      const mockVerification = {
        id: 1,
        submissionId: 'test-uuid',
        userName: 'John Doe',
        status: 'data_verified'
      };

      (prisma.insuranceVerification.findUnique as jest.Mock)
        .mockResolvedValue(mockVerification);

      const result = await service.getVerificationById(1);

      expect(result).toEqual(mockVerification);
    });

    it('should throw error when not found', async () => {
      (prisma.insuranceVerification.findUnique as jest.Mock)
        .mockResolvedValue(null);

      await expect(service.getVerificationById(999))
        .rejects
        .toThrow('Verification not found');
    });
  });
});
```

#### Run Backend Tests
```bash
# Run all tests
npm test

# Run with coverage
npm test -- --coverage

# Run specific test file
npm test verification.service.test

# Watch mode
npm test -- --watch
```

---

### Frontend Unit Tests

#### Component Test Example
```typescript
// src/app/modules/user-form/components/insurance-form/insurance-form.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ReactiveFormsModule } from '@angular/forms';
import { InsuranceFormComponent } from './insurance-form.component';
import { VerificationService } from '../../../../services/verification.service';
import { of, throwError } from 'rxjs';

describe('InsuranceFormComponent', () => {
  let component: InsuranceFormComponent;
  let fixture: ComponentFixture<InsuranceFormComponent>;
  let mockVerificationService: jasmine.SpyObj<VerificationService>;

  beforeEach(async () => {
    const verificationServiceSpy = jasmine.createSpyObj('VerificationService', ['submitVerification']);

    await TestBed.configureTestingModule({
      declarations: [ InsuranceFormComponent ],
      imports: [ ReactiveFormsModule ],
      providers: [
        { provide: VerificationService, useValue: verificationServiceSpy }
      ]
    }).compileComponents();

    mockVerificationService = TestBed.inject(VerificationService) as jasmine.SpyObj<VerificationService>;
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(InsuranceFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should initialize form with empty values', () => {
    expect(component.form.get('userName')?.value).toBe('');
    expect(component.form.get('memberIdSubmitted')?.value).toBe('');
  });

  it('should mark form as invalid when empty', () => {
    expect(component.form.valid).toBeFalse();
  });

  it('should mark form as valid when all required fields filled', () => {
    component.form.patchValue({
      userName: 'John Doe',
      dateOfBirth: '1985-03-15',
      memberIdSubmitted: 'ABC123',
      payerName: 'Blue Cross',
      userEmail: 'john@example.com'
    });
    
    expect(component.form.valid).toBeTrue();
  });

  it('should call service on form submit', () => {
    mockVerificationService.submitVerification.and.returnValue(of({ 
      success: true, 
      submissionId: 'test-123' 
    }));

    component.form.patchValue({
      userName: 'John Doe',
      dateOfBirth: '1985-03-15',
      memberIdSubmitted: 'ABC123',
      payerName: 'Blue Cross',
      userEmail: 'john@example.com'
    });

    component.onSubmit();

    expect(mockVerificationService.submitVerification).toHaveBeenCalledTimes(1);
  });

  it('should show error message on submission failure', () => {
    mockVerificationService.submitVerification.and.returnValue(
      throwError(() => new Error('Submission failed'))
    );

    component.form.patchValue({
      userName: 'John Doe',
      dateOfBirth: '1985-03-15',
      memberIdSubmitted: 'ABC123',
      payerName: 'Blue Cross',
      userEmail: 'john@example.com'
    });

    component.onSubmit();

    expect(component.errorMessage).toBe('Submission failed');
  });
});
```

#### Run Frontend Tests
```bash
# Run all tests
ng test

# Run with code coverage
ng test --code-coverage

# Run in headless mode (CI)
ng test --watch=false --browsers=ChromeHeadless
```

---

## Integration Testing

### API Integration Tests

```typescript
// src/__tests__/integration/verification.api.test.ts
import request from 'supertest';
import { app } from '../../app';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

describe('Verification API Integration Tests', () => {
  beforeAll(async () => {
    // Setup test database
    await prisma.$connect();
  });

  afterAll(async () => {
    // Cleanup
    await prisma.insuranceVerification.deleteMany();
    await prisma.$disconnect();
  });

  beforeEach(async () => {
    // Clear data before each test
    await prisma.insuranceVerification.deleteMany();
  });

  describe('POST /api/verification/submit', () => {
    it('should create verification and return submission ID', async () => {
      const response = await request(app)
        .post('/api/verification/submit')
        .send({
          userName: 'John Doe',
          dateOfBirth: '1985-03-15',
          memberIdSubmitted: 'ABC123',
          payerName: 'Blue Cross',
          userEmail: 'john@example.com',
          userPhone: '555-0123'
        })
        .expect(201);

      expect(response.body.success).toBe(true);
      expect(response.body.submissionId).toBeDefined();
      expect(response.body.message).toContain('review');

      // Verify in database
      const verification = await prisma.insuranceVerification.findUnique({
        where: { submissionId: response.body.submissionId }
      });

      expect(verification).toBeDefined();
      expect(verification?.userName).toBe('John Doe');
      expect(verification?.status).toBe('need_to_be_verified');
    });

    it('should return validation error for missing fields', async () => {
      const response = await request(app)
        .post('/api/verification/submit')
        .send({
          userName: 'John Doe'
          // Missing required fields
        })
        .expect(400);

      expect(response.body.success).toBe(false);
      expect(response.body.error).toContain('Validation');
    });

    it('should return error for invalid email', async () => {
      const response = await request(app)
        .post('/api/verification/submit')
        .send({
          userName: 'John Doe',
          dateOfBirth: '1985-03-15',
          memberIdSubmitted: 'ABC123',
          payerName: 'Blue Cross',
          userEmail: 'invalid-email',
          userPhone: '555-0123'
        })
        .expect(400);

      expect(response.body.success).toBe(false);
      expect(response.body.details[0].field).toBe('userEmail');
    });
  });

  describe('GET /api/verifications', () => {
    let authToken: string;

    beforeEach(async () => {
      // Create admin and get token
      const loginResponse = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'admin@example.com',
          password: 'admin123'
        });

      authToken = loginResponse.body.token;

      // Create test verifications
      await prisma.insuranceVerification.createMany({
        data: [
          {
            submissionId: 'test-1',
            userName: 'John Doe',
            dateOfBirth: new Date('1985-03-15'),
            memberIdSubmitted: 'ABC123',
            payerName: 'Blue Cross',
            status: 'data_verified'
          },
          {
            submissionId: 'test-2',
            userName: 'Jane Smith',
            dateOfBirth: new Date('1990-06-20'),
            memberIdSubmitted: 'XYZ789',
            payerName: 'UnitedHealth',
            status: 'ocr_failed'
          }
        ]
      });
    });

    it('should require authentication', async () => {
      await request(app)
        .get('/api/verifications')
        .expect(401);
    });

    it('should return all verifications with auth', async () => {
      const response = await request(app)
        .get('/api/verifications')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(response.body.success).toBe(true);
      expect(response.body.data).toHaveLength(2);
    });

    it('should filter by status', async () => {
      const response = await request(app)
        .get('/api/verifications?status=data_verified')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(response.body.data).toHaveLength(1);
      expect(response.body.data[0].status).toBe('data_verified');
    });

    it('should support pagination', async () => {
      const response = await request(app)
        .get('/api/verifications?page=1&limit=1')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(response.body.data).toHaveLength(1);
      expect(response.body.pagination.total).toBe(2);
      expect(response.body.pagination.totalPages).toBe(2);
    });
  });
});
```

---

## End-to-End Testing

### E2E Test with Playwright

```typescript
// e2e/verification-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Insurance Verification Flow', () => {
  test('should complete full verification submission', async ({ page }) => {
    // Navigate to form
    await page.goto('http://localhost:4200');

    // Fill out form
    await page.fill('[name="userName"]', 'John Doe');
    await page.fill('[name="dateOfBirth"]', '03/15/1985');
    await page.fill('[name="memberIdSubmitted"]', 'ABC123456');
    await page.fill('[name="payerName"]', 'Blue Cross Blue Shield');
    await page.fill('[name="userEmail"]', 'john.doe@example.com');
    await page.fill('[name="userPhone"]', '555-0123');

    // Upload insurance card images
    const frontFileInput = page.locator('input[type="file"][name="frontImage"]');
    await frontFileInput.setInputFiles('test-data/insurance-card-front.jpg');

    const backFileInput = page.locator('input[type="file"][name="backImage"]');
    await backFileInput.setInputFiles('test-data/insurance-card-back.jpg');

    // Submit form
    await page.click('button[type="submit"]');

    // Wait for success message
    await page.waitForSelector('.success-message');
    const successMessage = await page.textContent('.success-message');
    expect(successMessage).toContain('Our agent will review');

    // Verify confirmation number is displayed
    const confirmationNumber = await page.textContent('.confirmation-number');
    expect(confirmationNumber).toBeTruthy();
  });

  test('should show validation errors for empty form', async ({ page }) => {
    await page.goto('http://localhost:4200');

    // Try to submit empty form
    await page.click('button[type="submit"]');

    // Check for validation errors
    const errors = await page.locator('.error-message').all();
    expect(errors.length).toBeGreaterThan(0);
  });

  test('should show error for invalid email', async ({ page }) => {
    await page.goto('http://localhost:4200');

    await page.fill('[name="userEmail"]', 'invalid-email');
    await page.blur('[name="userEmail"]');

    const emailError = await page.textContent('.error-message.email');
    expect(emailError).toContain('Invalid email');
  });
});

test.describe('Admin Portal', () => {
  test('should login and view dashboard', async ({ page }) => {
    // Navigate to admin portal
    await page.goto('http://localhost:4200/admin');

    // Login
    await page.fill('[name="email"]', 'admin@example.com');
    await page.fill('[name="password"]', 'admin123');
    await page.click('button[type="submit"]');

    // Wait for dashboard
    await page.waitForURL('**/admin/dashboard');

    // Verify dashboard elements
    await expect(page.locator('.dashboard-metrics')).toBeVisible();
    await expect(page.locator('.verifications-table')).toBeVisible();
  });

  test('should filter verifications by status', async ({ page }) => {
    // Login first (assume helper function)
    await loginAsAdmin(page);

    // Navigate to verifications
    await page.goto('http://localhost:4200/admin/verifications');

    // Select status filter
    await page.selectOption('[name="statusFilter"]', 'data_verified');

    // Wait for filtered results
    await page.waitForLoadState('networkidle');

    // Verify all rows have correct status
    const statusBadges = await page.locator('.status-badge').allTextContents();
    statusBadges.forEach(badge => {
      expect(badge).toBe('Verified');
    });
  });

  test('should view verification details', async ({ page }) => {
    await loginAsAdmin(page);

    // Click on first verification
    await page.click('.verification-row:first-child');

    // Verify detail page loaded
    await expect(page.locator('.verification-detail')).toBeVisible();

    // Check key elements
    await expect(page.locator('.user-info')).toBeVisible();
    await expect(page.locator('.verification-results')).toBeVisible();
    await expect(page.locator('.insurance-card-images')).toBeVisible();
  });
});

// Helper function
async function loginAsAdmin(page) {
  await page.goto('http://localhost:4200/admin');
  await page.fill('[name="email"]', 'admin@example.com');
  await page.fill('[name="password"]', 'admin123');
  await page.click('button[type="submit"]');
  await page.waitForURL('**/admin/dashboard');
}
```

#### Run E2E Tests
```bash
# Install Playwright
npx playwright install

# Run tests
npx playwright test

# Run with UI
npx playwright test --ui

# Run specific test
npx playwright test verification-flow

# Generate test report
npx playwright show-report
```

---

## n8n Workflow Testing

### Manual Testing Checklist

**Successful Verification Path:**
- [ ] Webhook receives correct payload
- [ ] Database updated to `ocr_in_progress`
- [ ] Mindee OCR returns results with >70% confidence
- [ ] OCR data matches user-submitted data
- [ ] Stedi API returns active insurance
- [ ] Database updated to `data_verified`
- [ ] All fields populated correctly

**OCR Failure Path:**
- [ ] Low confidence score triggers `ocr_failed` status
- [ ] `requires_user_action` set to true
- [ ] User notification message set
- [ ] Workflow stops appropriately

**Data Mismatch Path:**
- [ ] Mismatched Member IDs detected
- [ ] Status set to `data_mismatch`
- [ ] Comparison shown in message
- [ ] Workflow stops

**API Retry Path:**
- [ ] First attempt fails
- [ ] Wait 30 seconds
- [ ] Second attempt executes
- [ ] If all fail, status = `api_error_manual_review`

### Test Data

```json
// Successful case
{
  "submissionId": "test-success-001",
  "userName": "Test User",
  "dateOfBirth": "1985-01-01",
  "memberIdSubmitted": "TEST123",
  "payerName": "Test Insurance",
  "insuranceCardFrontUrl": "http://example.com/front.jpg"
}

// OCR failure case (use invalid image URL)
{
  "submissionId": "test-ocr-fail-001",
  "insuranceCardFrontUrl": "http://example.com/invalid.jpg"
}

// Data mismatch case (modify OCR code to return different ID)
{
  "submissionId": "test-mismatch-001",
  "memberIdSubmitted": "TEST123",
  // OCR will return "TEST456"
}
```

---

## Load Testing

### Simple Load Test with Artillery

```bash
# Install Artillery
npm install -g artillery

# Create load test config
```

```yaml
# load-test.yml
config:
  target: "http://localhost:3000"
  phases:
    - duration: 60
      arrivalRate: 10  # 10 requests per second
      name: "Warm up"
    - duration: 120
      arrivalRate: 50  # 50 requests per second
      name: "Sustained load"
  
scenarios:
  - name: "Submit verification"
    flow:
      - post:
          url: "/api/verification/submit"
          json:
            userName: "Load Test User"
            dateOfBirth: "1985-01-01"
            memberIdSubmitted: "LOAD{{ $randomNumber() }}"
            payerName: "Test Insurance"
            userEmail: "test{{ $randomNumber() }}@example.com"
            userPhone: "555-0123"
```

```bash
# Run load test
artillery run load-test.yml

# Generate report
artillery run --output report.json load-test.yml
artillery report report.json
```

---

## Security Testing

### Automated Security Checks

**OWASP ZAP Scan:**
```bash
# Pull OWASP ZAP Docker image
docker pull owasp/zap2docker-stable

# Run baseline scan
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t http://localhost:3000 \
  -r zap-report.html
```

**npm audit:**
```bash
# Check for vulnerabilities
npm audit

# Fix vulnerabilities
npm audit fix

# Check production dependencies only
npm audit --production
```

### Manual Security Tests

**Authentication Tests:**
- [ ] Cannot access admin endpoints without token
- [ ] Expired tokens rejected
- [ ] Invalid tokens rejected
- [ ] Password hashing works
- [ ] Rate limiting prevents brute force

**Authorization Tests:**
- [ ] Admin cannot access super_admin endpoints
- [ ] Users cannot access admin endpoints
- [ ] Cannot view other users' data

**Input Validation Tests:**
- [ ] SQL injection attempts blocked
- [ ] XSS attempts sanitized
- [ ] File upload restrictions enforced
- [ ] Request size limits enforced

---

## Test Coverage Goals

### Target Coverage

**Backend:**
- Unit Tests: 80%+ coverage
- Integration Tests: Key user flows
- API Tests: All endpoints

**Frontend:**
- Component Tests: 70%+ coverage
- Service Tests: 80%+ coverage
- E2E Tests: Critical paths

**n8n Workflows:**
- Manual testing: All paths
- Documented test cases

---

## Continuous Integration

### GitHub Actions Example

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  backend-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: |
          cd backend
          npm ci
      
      - name: Run tests
        run: |
          cd backend
          npm test -- --coverage
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./backend/coverage/lcov.info

  frontend-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: |
          cd frontend
          npm ci
      
      - name: Run tests
        run: |
          cd frontend
          ng test --watch=false --code-coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./frontend/coverage/lcov.info

  e2e-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install Playwright
        run: npx playwright install --with-deps
      
      - name: Run E2E tests
        run: npx playwright test
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
```

---

## Test Data Management

### Create Test Data

```typescript
// scripts/seed-test-data.ts
import { PrismaClient } from '@prisma/client';
import { faker } from '@faker-js/faker';

const prisma = new PrismaClient();

async function seedTestData() {
  console.log('🌱 Seeding test data...');

  // Create test verifications
  for (let i = 0; i < 50; i++) {
    await prisma.insuranceVerification.create({
      data: {
        submissionId: faker.string.uuid(),
        userName: faker.person.fullName(),
        dateOfBirth: faker.date.past({ years: 40 }),
        memberIdSubmitted: faker.string.alphanumeric(10).toUpperCase(),
        payerName: faker.helpers.arrayElement([
          'Blue Cross Blue Shield',
          'UnitedHealthcare',
          'Aetna',
          'Cigna',
          'Humana'
        ]),
        userEmail: faker.internet.email(),
        userPhone: faker.phone.number(),
        status: faker.helpers.arrayElement([
          'need_to_be_verified',
          'data_verified',
          'ocr_failed',
          'insurance_invalid'
        ])
      }
    });
  }

  console.log('✅ Test data seeded successfully');
}

seedTestData()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

---

## Document Information
- **Version:** 1.0
- **Last Updated:** January 22, 2026
- **Coverage:** Unit, Integration, E2E, Security
- **Status:** Complete