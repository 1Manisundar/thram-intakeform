# Database Schema Documentation

## Overview
This document provides detailed information about the PostgreSQL database schema for the Insurance Verification System.

**Database:** PostgreSQL 15+  
**ORM:** Prisma  
**Schema Location:** `backend/prisma/schema.prisma`

---

## Entity Relationship Diagram

```
┌─────────────────────────────────┐
│  insurance_verifications        │
│─────────────────────────────────│
│  id (PK)                        │
│  submission_id (UNIQUE)         │
│  user_name                      │
│  date_of_birth                  │
│  member_id_submitted            │
│  payer_name                     │
│  status                         │
│  ...                            │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  admins                         │
│─────────────────────────────────│
│  id (PK)                        │
│  email (UNIQUE)                 │
│  password_hash                  │
│  name                           │
│  role                           │
│  ...                            │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  audit_logs                     │
│─────────────────────────────────│
│  id (PK)                        │
│  admin_id (FK)                  │
│  action                         │
│  entity                         │
│  details                        │
│  ...                            │
└─────────────────────────────────┘
```

---

## Table: insurance_verifications

### Purpose
Stores all insurance verification submissions and their processing status.

### Columns

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER | No | AUTO | Primary key |
| `submission_id` | VARCHAR(100) | No | UUID | Unique submission identifier |
| `user_name` | VARCHAR(255) | No | - | Full name of user |
| `date_of_birth` | DATE | No | - | User's date of birth |
| `member_id_submitted` | VARCHAR(100) | No | - | Member ID entered by user |
| `payer_name` | VARCHAR(255) | No | - | Insurance payer name |
| `user_email` | VARCHAR(255) | Yes | NULL | User's email address |
| `user_phone` | VARCHAR(20) | Yes | NULL | User's phone number |
| `insurance_card_front_url` | TEXT | Yes | NULL | URL/path to front image |
| `insurance_card_back_url` | TEXT | Yes | NULL | URL/path to back image |
| `member_id_extracted` | VARCHAR(100) | Yes | NULL | Member ID from OCR |
| `payer_name_extracted` | VARCHAR(255) | Yes | NULL | Payer name from OCR |
| `ocr_confidence_score` | DECIMAL(5,2) | Yes | NULL | OCR confidence (0-100) |
| `ocr_attempt_count` | INTEGER | No | 0 | Number of OCR attempts |
| `ocr_last_attempt_at` | TIMESTAMP | Yes | NULL | Last OCR attempt time |
| `stedi_eligibility_status` | VARCHAR(50) | Yes | NULL | active/inactive/unknown |
| `stedi_plan_name` | VARCHAR(255) | Yes | NULL | Insurance plan name |
| `stedi_coverage_amount` | DECIMAL(10,2) | Yes | NULL | Coverage amount |
| `stedi_copay` | DECIMAL(10,2) | Yes | NULL | Copay amount |
| `stedi_deductible` | DECIMAL(10,2) | Yes | NULL | Deductible amount |
| `stedi_response_json` | JSONB | Yes | NULL | Full Stedi API response |
| `stedi_attempt_count` | INTEGER | No | 0 | Number of Stedi API attempts |
| `stedi_last_attempt_at` | TIMESTAMP | Yes | NULL | Last API call time |
| `status` | VARCHAR(50) | No | 'need_to_be_verified' | Current status |
| `verification_notes` | TEXT | Yes | NULL | Admin/system notes |
| `requires_user_action` | BOOLEAN | No | false | Needs user to fix something |
| `user_action_message` | TEXT | Yes | NULL | Message to show user |
| `created_at` | TIMESTAMP | No | NOW() | Record creation time |
| `updated_at` | TIMESTAMP | No | NOW() | Last update time |
| `verified_at` | TIMESTAMP | Yes | NULL | Verification completion time |
| `admin_contacted_at` | TIMESTAMP | Yes | NULL | When admin contacted user |
| `assigned_admin_id` | INTEGER | Yes | NULL | Assigned admin ID |
| `admin_priority` | VARCHAR(20) | No | 'normal' | urgent/high/normal/low |

### Indexes
```sql
CREATE INDEX idx_status ON insurance_verifications(status);
CREATE INDEX idx_requires_user_action ON insurance_verifications(requires_user_action);
CREATE INDEX idx_created_at ON insurance_verifications(created_at);
```

### Status Values

| Status | Description | User Action Required |
|--------|-------------|---------------------|
| `need_to_be_verified` | Initial state after submission | No |
| `ocr_in_progress` | OCR processing started | No |
| `ocr_failed` | OCR couldn't read document | Yes |
| `data_mismatch` | OCR data ≠ User data | Yes |
| `stedi_in_progress` | API call in progress | No |
| `stedi_retrying` | Retrying failed API call | No |
| `api_error_manual_review` | API failed after retries | No (Admin) |
| `partial_verification` | Incomplete Stedi data | No (Admin) |
| `data_verified` | Successfully verified | No |
| `insurance_invalid` | Insurance not active | No (Admin) |
| `awaiting_user_resubmission` | Waiting for new data | Yes |
| `admin_contacted` | Admin reached out | No |
| `appointment_scheduled` | Next steps arranged | No |
| `closed` | Process complete | No |

---

## Table: admins

### Purpose
Stores admin user accounts for the admin portal.

### Columns

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER | No | AUTO | Primary key |
| `email` | VARCHAR(255) | No | - | Admin email (unique) |
| `password_hash` | VARCHAR(255) | No | - | bcrypt hashed password |
| `name` | VARCHAR(255) | No | - | Admin full name |
| `role` | VARCHAR(50) | No | 'admin' | admin/super_admin |
| `is_active` | BOOLEAN | No | true | Account active status |
| `created_at` | TIMESTAMP | No | NOW() | Account creation time |
| `last_login_at` | TIMESTAMP | Yes | NULL | Last successful login |

### Indexes
```sql
CREATE UNIQUE INDEX idx_admin_email ON admins(email);
```

### Roles

| Role | Permissions |
|------|------------|
| `admin` | View submissions, add notes, mark contacted |
| `super_admin` | All admin permissions + manage admins |

---

## Table: audit_logs

### Purpose
Tracks all admin actions for HIPAA compliance and security auditing.

### Columns

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | INTEGER | No | AUTO | Primary key |
| `admin_id` | INTEGER | Yes | NULL | Admin who performed action |
| `action` | VARCHAR(100) | No | - | Action performed |
| `entity` | VARCHAR(50) | No | - | Entity type (verification/admin) |
| `entity_id` | VARCHAR(100) | Yes | NULL | ID of affected entity |
| `details` | JSONB | Yes | NULL | Additional context |
| `ip_address` | VARCHAR(45) | Yes | NULL | IP address of request |
| `created_at` | TIMESTAMP | No | NOW() | When action occurred |

### Indexes
```sql
CREATE INDEX idx_audit_admin_id ON audit_logs(admin_id);
CREATE INDEX idx_audit_created_at ON audit_logs(created_at);
```

### Common Actions

| Action | Description |
|--------|-------------|
| `LOGIN` | Admin logged in |
| `LOGOUT` | Admin logged out |
| `VIEW_VERIFICATION` | Viewed verification details |
| `UPDATE_STATUS` | Changed verification status |
| `ADD_NOTE` | Added admin note |
| `MARK_CONTACTED` | Marked as contacted |
| `ASSIGN_ADMIN` | Assigned to admin |
| `TRIGGER_VERIFICATION` | Manually triggered verification |
| `CREATE_ADMIN` | Created new admin account |
| `UPDATE_ADMIN` | Updated admin account |
| `DELETE_ADMIN` | Deleted admin account |

---

## Prisma Schema Definition

**File:** `backend/prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model InsuranceVerification {
  id                       Int       @id @default(autoincrement())
  submissionId             String    @unique @map("submission_id")
  
  // User Data
  userName                 String    @map("user_name")
  dateOfBirth              DateTime  @map("date_of_birth") @db.Date
  memberIdSubmitted        String    @map("member_id_submitted")
  payerName                String    @map("payer_name")
  userEmail                String?   @map("user_email")
  userPhone                String?   @map("user_phone")
  
  // Document URLs
  insuranceCardFrontUrl    String?   @map("insurance_card_front_url")
  insuranceCardBackUrl     String?   @map("insurance_card_back_url")
  
  // OCR Results
  memberIdExtracted        String?   @map("member_id_extracted")
  payerNameExtracted       String?   @map("payer_name_extracted")
  ocrConfidenceScore       Decimal?  @map("ocr_confidence_score") @db.Decimal(5, 2)
  ocrAttemptCount          Int       @default(0) @map("ocr_attempt_count")
  ocrLastAttemptAt         DateTime? @map("ocr_last_attempt_at")
  
  // Stedi Verification Results
  stediEligibilityStatus   String?   @map("stedi_eligibility_status")
  stediPlanName            String?   @map("stedi_plan_name")
  stediCoverageAmount      Decimal?  @map("stedi_coverage_amount") @db.Decimal(10, 2)
  stediCopay               Decimal?  @map("stedi_copay") @db.Decimal(10, 2)
  stediDeductible          Decimal?  @map("stedi_deductible") @db.Decimal(10, 2)
  stediResponseJson        Json?     @map("stedi_response_json")
  stediAttemptCount        Int       @default(0) @map("stedi_attempt_count")
  stediLastAttemptAt       DateTime? @map("stedi_last_attempt_at")
  
  // Status Management
  status                   String    @default("need_to_be_verified")
  verificationNotes        String?   @map("verification_notes")
  requiresUserAction       Boolean   @default(false) @map("requires_user_action")
  userActionMessage        String?   @map("user_action_message")
  
  // Timestamps
  createdAt                DateTime  @default(now()) @map("created_at")
  updatedAt                DateTime  @updatedAt @map("updated_at")
  verifiedAt               DateTime? @map("verified_at")
  adminContactedAt         DateTime? @map("admin_contacted_at")
  
  // Admin tracking
  assignedAdminId          Int?      @map("assigned_admin_id")
  adminPriority            String    @default("normal") @map("admin_priority")
  
  @@index([status])
  @@index([requiresUserAction])
  @@index([createdAt])
  @@map("insurance_verifications")
}

model Admin {
  id                Int       @id @default(autoincrement())
  email             String    @unique
  passwordHash      String    @map("password_hash")
  name              String
  role              String    @default("admin")
  isActive          Boolean   @default(true) @map("is_active")
  createdAt         DateTime  @default(now()) @map("created_at")
  lastLoginAt       DateTime? @map("last_login_at")
  
  @@map("admins")
}

model AuditLog {
  id                Int       @id @default(autoincrement())
  adminId           Int?      @map("admin_id")
  action            String
  entity            String
  entityId          String?   @map("entity_id")
  details           Json?
  ipAddress         String?   @map("ip_address")
  createdAt         DateTime  @default(now()) @map("created_at")
  
  @@index([adminId])
  @@index([createdAt])
  @@map("audit_logs")
}
```

---

## Database Migrations

### Initial Migration
```bash
# Create initial migration
npx prisma migrate dev --name init

# This creates:
# - Database tables
# - Indexes
# - Constraints
```

### Common Migration Commands
```bash
# Generate Prisma Client
npx prisma generate

# Create a new migration
npx prisma migrate dev --name <migration_name>

# Apply migrations in production
npx prisma migrate deploy

# Reset database (CAUTION: Deletes all data)
npx prisma migrate reset

# View migration status
npx prisma migrate status
```

---

## Seed Data (Development Only)

### Create Seed Script
**File:** `backend/prisma/seed.ts`

```typescript
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcrypt';

const prisma = new PrismaClient();

async function main() {
  console.log('🌱 Seeding database...');

  // Create admin user
  const hashedPassword = await bcrypt.hash('admin123', 10);
  
  const admin = await prisma.admin.upsert({
    where: { email: 'admin@example.com' },
    update: {},
    create: {
      email: 'admin@example.com',
      passwordHash: hashedPassword,
      name: 'Admin User',
      role: 'super_admin',
    },
  });

  console.log('✅ Admin created:', admin.email);

  // Create sample verification (for testing)
  const verification = await prisma.insuranceVerification.create({
    data: {
      submissionId: 'test-submission-001',
      userName: 'John Doe',
      dateOfBirth: new Date('1985-03-15'),
      memberIdSubmitted: 'ABC123456',
      payerName: 'Blue Cross Blue Shield',
      userEmail: 'john.doe@example.com',
      userPhone: '555-0123',
      status: 'need_to_be_verified',
    },
  });

  console.log('✅ Sample verification created:', verification.submissionId);
}

main()
  .catch((e) => {
    console.error('❌ Seed error:', e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

**Run seed:**
```bash
npx ts-node prisma/seed.ts
```

---

## Query Examples

### Common Queries Using Prisma

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Get all verifications with status filter
const verifications = await prisma.insuranceVerification.findMany({
  where: {
    status: 'data_verified'
  },
  orderBy: {
    createdAt: 'desc'
  }
});

// Get single verification by submission ID
const verification = await prisma.insuranceVerification.findUnique({
  where: {
    submissionId: 'abc-123-xyz'
  }
});

// Update verification status
const updated = await prisma.insuranceVerification.update({
  where: { id: 1 },
  data: {
    status: 'data_verified',
    verifiedAt: new Date()
  }
});

// Get verifications requiring user action
const needsAction = await prisma.insuranceVerification.findMany({
  where: {
    requiresUserAction: true
  }
});

// Search by user name
const results = await prisma.insuranceVerification.findMany({
  where: {
    userName: {
      contains: 'John',
      mode: 'insensitive'
    }
  }
});

// Get count by status
const counts = await prisma.insuranceVerification.groupBy({
  by: ['status'],
  _count: true
});

// Create audit log
await prisma.auditLog.create({
  data: {
    adminId: 1,
    action: 'VIEW_VERIFICATION',
    entity: 'verification',
    entityId: '123',
    ipAddress: '192.168.1.1',
    details: {
      verificationId: 123,
      action: 'viewed details page'
    }
  }
});
```

---

## Performance Optimization

### Indexing Strategy
All frequently queried columns are indexed:
- `status` - Most common filter
- `requiresUserAction` - Admin dashboard filter
- `createdAt` - Sorting and date range queries

### Query Optimization Tips
```typescript
// Good: Select only needed fields
const verification = await prisma.insuranceVerification.findMany({
  select: {
    id: true,
    submissionId: true,
    userName: true,
    status: true
  }
});

// Bad: Selecting all fields when not needed
const verification = await prisma.insuranceVerification.findMany();

// Use pagination for large datasets
const page = await prisma.insuranceVerification.findMany({
  skip: (pageNumber - 1) * pageSize,
  take: pageSize
});
```

---

## Backup Strategy

### Automated Backups (Production)
```bash
# Daily backup script
#!/bin/bash
BACKUP_DIR="/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
FILENAME="insurance_verification_${DATE}.sql"

pg_dump -U postgres insurance_verification > ${BACKUP_DIR}/${FILENAME}
gzip ${BACKUP_DIR}/${FILENAME}

# Keep only last 30 days
find ${BACKUP_DIR} -name "*.sql.gz" -mtime +30 -delete
```

### Manual Backup
```bash
# Backup
pg_dump -U postgres insurance_verification > backup.sql

# Restore
psql -U postgres insurance_verification < backup.sql
```

---

## Data Retention Policy

### Production Data
- **Verification Records:** Keep indefinitely (HIPAA compliance)
- **Audit Logs:** Keep for 7 years (HIPAA requirement)
- **Uploaded Files:** Keep for duration of records

### Development Data
- **Reset as needed:** No retention requirements
- **Use seed data:** For testing

---

## Security Considerations

### Encryption
- **At Rest:** Enable PostgreSQL encryption
- **In Transit:** Always use SSL connections
- **Application Level:** Sensitive fields encrypted if needed

### Access Control
```sql
-- Production: Limit permissions
GRANT SELECT, INSERT, UPDATE ON insurance_verifications TO app_user;
GRANT SELECT, INSERT ON audit_logs TO app_user;

-- Admins table: Extra protection
REVOKE ALL ON admins FROM PUBLIC;
GRANT SELECT, INSERT, UPDATE ON admins TO app_user;
```

### Connection Security
```env
# Always use SSL in production
DATABASE_URL="postgresql://user:pass@host:5432/db?sslmode=require"
```

---

## Monitoring Queries

### Check Database Size
```sql
SELECT pg_size_pretty(pg_database_size('insurance_verification'));
```

### Check Table Sizes
```sql
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Check Active Connections
```sql
SELECT count(*) FROM pg_stat_activity WHERE datname = 'insurance_verification';
```

---

## Document Information
- **Version:** 1.0
- **Last Updated:** January 22, 2026
- **Schema Version:** 1 (initial)
- **Status:** Complete