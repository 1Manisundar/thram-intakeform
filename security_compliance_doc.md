# Security & HIPAA Compliance Guide

## Overview
This document outlines security measures and HIPAA compliance requirements for the Insurance Verification System.

**Compliance Framework:** HIPAA (Health Insurance Portability and Accountability Act)  
**PHI Handled:** Patient names, dates of birth, member IDs, insurance information

---

## HIPAA Compliance Overview

### What is PHI (Protected Health Information)?

In this system, PHI includes:
- ✅ Patient names
- ✅ Dates of birth
- ✅ Member IDs
- ✅ Insurance information
- ✅ Contact information (email, phone)
- ✅ Insurance card images

### HIPAA Requirements

**Three Main Rules:**
1. **Privacy Rule** - How PHI can be used and disclosed
2. **Security Rule** - How PHI must be protected
3. **Breach Notification Rule** - What to do if PHI is compromised

---

## Technical Safeguards

### 1. Encryption

#### Data at Rest
**Database (PostgreSQL):**
```bash
# Enable encryption on RDS
# Via AWS Console: 
# - Enable encryption when creating RDS instance
# - Uses AWS KMS for key management
```

**File Storage (S3):**
```bash
# Enable default encryption
aws s3api put-bucket-encryption \
  --bucket insurance-verification-files \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

**Application Server:**
```bash
# Encrypt EBS volumes
# Via AWS Console when launching EC2:
# - Check "Encrypted" for root volume
# - Select KMS key
```

#### Data in Transit
**HTTPS Everywhere:**
```nginx
# Nginx SSL configuration
server {
    listen 443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/api.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourdomain.com/privkey.pem;
    
    # Strong SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers on;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}
```

**Database Connections:**
```env
# Always use SSL for database connections
DATABASE_URL="postgresql://user:pass@host:5432/db?sslmode=require"
```

**API Calls:**
- Mindee API: HTTPS only
- Stedi API: HTTPS only
- All internal communication: HTTPS

---

### 2. Access Control

#### Role-Based Access Control (RBAC)

**Admin Roles:**
```typescript
enum AdminRole {
  ADMIN = 'admin',        // Can view and manage verifications
  SUPER_ADMIN = 'super_admin'  // Can manage admins + all admin permissions
}
```

**Permission Matrix:**

| Action | Admin | Super Admin |
|--------|-------|-------------|
| View verifications | ✅ | ✅ |
| Add notes | ✅ | ✅ |
| Mark contacted | ✅ | ✅ |
| View audit logs | ✅ | ✅ |
| Create admin users | ❌ | ✅ |
| Delete admin users | ❌ | ✅ |
| Modify admin roles | ❌ | ✅ |

#### Authentication Implementation

**JWT Token Security:**
```typescript
// src/middleware/auth.middleware.ts
import jwt from 'jsonwebtoken';

export const authMiddleware = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Check if admin is still active
    const admin = await prisma.admin.findUnique({
      where: { id: decoded.adminId }
    });
    
    if (!admin || !admin.isActive) {
      return res.status(401).json({ error: 'Invalid or inactive user' });
    }
    
    req.admin = admin;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};
```

**Password Security:**
```typescript
// Password hashing with bcrypt
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12; // Strong enough for production

// Hash password
const hashedPassword = await bcrypt.hash(password, SALT_ROUNDS);

// Verify password
const isValid = await bcrypt.compare(inputPassword, hashedPassword);
```

**Password Requirements:**
- Minimum 12 characters
- At least one uppercase letter
- At least one lowercase letter
- At least one number
- At least one special character

**Implementation:**
```typescript
function validatePassword(password: string): boolean {
  const minLength = 12;
  const hasUpperCase = /[A-Z]/.test(password);
  const hasLowerCase = /[a-z]/.test(password);
  const hasNumbers = /\d/.test(password);
  const hasSpecialChar = /[!@#$%^&*(),.?":{}|<>]/.test(password);
  
  return password.length >= minLength && 
         hasUpperCase && 
         hasLowerCase && 
         hasNumbers && 
         hasSpecialChar;
}
```

#### Session Management

**Automatic Timeout:**
```typescript
// JWT expiration
const token = jwt.sign(
  { adminId: admin.id },
  process.env.JWT_SECRET,
  { expiresIn: '8h' } // Auto logout after 8 hours
);
```

**Concurrent Session Control:**
```typescript
// Track active sessions in database
model AdminSession {
  id          Int       @id @default(autoincrement())
  adminId     Int
  token       String    @unique
  createdAt   DateTime  @default(now())
  expiresAt   DateTime
  ipAddress   String?
  userAgent   String?
}

// Limit to 3 concurrent sessions per admin
```

---

### 3. Audit Logging

#### What to Log

**All PHI Access:**
- Who accessed
- What was accessed
- When it was accessed
- What action was taken
- IP address
- Result (success/failure)

#### Audit Log Implementation

```typescript
// src/services/audit.service.ts
export class AuditService {
  async logAction(data: {
    adminId?: number;
    action: string;
    entity: string;
    entityId?: string;
    details?: any;
    ipAddress?: string;
    result?: 'success' | 'failure';
  }) {
    await prisma.auditLog.create({
      data: {
        adminId: data.adminId,
        action: data.action,
        entity: data.entity,
        entityId: data.entityId,
        details: data.details,
        ipAddress: data.ipAddress,
        createdAt: new Date()
      }
    });
  }
}

// Usage in controller
await auditService.logAction({
  adminId: req.admin.id,
  action: 'VIEW_VERIFICATION',
  entity: 'verification',
  entityId: verificationId,
  ipAddress: req.ip,
  details: { submissionId: verification.submissionId },
  result: 'success'
});
```

#### Audit Log Retention

**HIPAA Requirement:** Retain audit logs for **6 years**

**Implementation:**
```sql
-- Archive old logs (run annually)
CREATE TABLE audit_logs_archive (LIKE audit_logs);

INSERT INTO audit_logs_archive
SELECT * FROM audit_logs
WHERE created_at < NOW() - INTERVAL '6 years';

-- Don't delete, move to cold storage instead
```

---

### 4. Input Validation & Sanitization

#### Prevent SQL Injection

**Using Prisma (Safe by default):**
```typescript
// Prisma uses parameterized queries automatically
const verification = await prisma.insuranceVerification.findUnique({
  where: { id: parseInt(req.params.id) } // Safe
});
```

#### Prevent XSS Attacks

**Sanitize User Input:**
```typescript
import { z } from 'zod';

// Validation schema
const createVerificationSchema = z.object({
  userName: z.string().min(1).max(255).trim(),
  dateOfBirth: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  memberIdSubmitted: z.string().min(1).max(100).trim(),
  payerName: z.string().min(1).max(255).trim(),
  userEmail: z.string().email().max(255),
  userPhone: z.string().max(20).optional()
});

// Validate in controller
const validatedData = createVerificationSchema.parse(req.body);
```

**Content Security Policy:**
```typescript
// Use helmet.js
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
    }
  }
}));
```

---

### 5. Network Security

#### Firewall Rules

**Backend EC2 Security Group:**
```
Inbound:
- SSH (22): Your IP only
- HTTP (80): 0.0.0.0/0 (redirect to HTTPS)
- HTTPS (443): 0.0.0.0/0

Outbound:
- PostgreSQL (5432): RDS security group only
- HTTPS (443): 0.0.0.0/0 (for API calls)
```

**RDS Security Group:**
```
Inbound:
- PostgreSQL (5432): Backend EC2 security group only
- PostgreSQL (5432): n8n EC2 security group only

Outbound:
- None needed
```

**n8n EC2 Security Group:**
```
Inbound:
- SSH (22): Your IP only
- HTTP (5678): Backend EC2 security group only

Outbound:
- HTTPS (443): 0.0.0.0/0 (for API calls)
- PostgreSQL (5432): RDS security group
```

#### Rate Limiting

**Prevent Brute Force Attacks:**
```typescript
import rateLimit from 'express-rate-limit';

// Login endpoint rate limit
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many login attempts, please try again later'
});

app.post('/api/auth/login', loginLimiter, authController.login);

// API rate limit
const apiLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 1000, // 1000 requests per hour
  message: 'Too many requests, please try again later'
});

app.use('/api', apiLimiter);
```

---

## Administrative Safeguards

### 1. Access Management

#### User Provisioning
```typescript
// Create admin (super_admin only)
async createAdmin(data: CreateAdminDto, createdBy: Admin) {
  // Audit log
  await auditService.logAction({
    adminId: createdBy.id,
    action: 'CREATE_ADMIN',
    entity: 'admin',
    details: { email: data.email, role: data.role }
  });
  
  return await prisma.admin.create({
    data: {
      email: data.email,
      passwordHash: await bcrypt.hash(data.password, 12),
      name: data.name,
      role: data.role,
      isActive: true
    }
  });
}
```

#### User Deprovisioning
```typescript
// Deactivate admin (super_admin only)
async deactivateAdmin(adminId: number, deactivatedBy: Admin) {
  // Audit log
  await auditService.logAction({
    adminId: deactivatedBy.id,
    action: 'DEACTIVATE_ADMIN',
    entity: 'admin',
    entityId: adminId.toString()
  });
  
  return await prisma.admin.update({
    where: { id: adminId },
    data: { isActive: false }
  });
}
```

#### Regular Access Reviews

**Quarterly Review Process:**
1. Export list of all admin users
2. Review with management
3. Deactivate inactive users
4. Verify roles are appropriate
5. Document review

```sql
-- Get all active admins
SELECT id, email, name, role, created_at, last_login_at
FROM admins
WHERE is_active = true
ORDER BY last_login_at DESC NULLS LAST;

-- Find inactive admins (no login in 90 days)
SELECT id, email, name, last_login_at
FROM admins
WHERE is_active = true
AND (last_login_at IS NULL OR last_login_at < NOW() - INTERVAL '90 days');
```

---

### 2. Training Requirements

#### Annual HIPAA Training

**Required Topics:**
- [ ] Understanding PHI
- [ ] Privacy and Security Rules
- [ ] Proper handling of patient data
- [ ] Breach notification procedures
- [ ] Password security
- [ ] Physical security
- [ ] Incident reporting

**Documentation:**
- Maintain training records for 6 years
- Track completion dates
- Update training materials annually

---

### 3. Incident Response Plan

#### Breach Detection

**Monitoring for Breaches:**
```typescript
// Monitor failed login attempts
const failedLogins = await prisma.auditLog.count({
  where: {
    action: 'LOGIN',
    details: { path: '$.result', equals: 'failure' },
    createdAt: { gte: new Date(Date.now() - 60 * 60 * 1000) }
  }
});

if (failedLogins > 50) {
  // Alert security team
  await sendSecurityAlert('High number of failed login attempts');
}
```

#### Breach Response Procedure

**If breach suspected:**
1. **Immediate Actions (0-24 hours):**
   - Contain the breach
   - Preserve evidence
   - Notify security team
   - Begin investigation

2. **Investigation (24-48 hours):**
   - Determine scope of breach
   - Identify affected PHI
   - Document timeline
   - Assess risk

3. **Notification (Within 60 days):**
   - Notify affected individuals
   - Notify HHS (if >500 individuals)
   - Notify media (if >500 individuals in same state)
   - Document all notifications

4. **Remediation:**
   - Fix vulnerability
   - Update security measures
   - Retrain staff
   - Document lessons learned

---

## Physical Safeguards

### Data Center Security (AWS)

**AWS Compliance:**
- AWS is HIPAA-eligible
- Must sign BAA with AWS
- Use HIPAA-eligible services only:
  - ✅ EC2
  - ✅ RDS
  - ✅ S3
  - ✅ CloudFront
  - ✅ SES
  - ❌ n8n Cloud (use self-hosted only)

### Workstation Security

**Admin Computers:**
- [ ] Password-protected
- [ ] Automatic screen lock (5 minutes)
- [ ] Encrypted hard drives
- [ ] Antivirus installed
- [ ] Firewall enabled
- [ ] Regular security updates
- [ ] Clean desk policy

---

## Third-Party Compliance

### Business Associate Agreements (BAA)

**Required BAAs:**
- [ ] AWS (infrastructure provider)
- [ ] Mindee (OCR provider)
- [ ] Stedi (clearinghouse API)

**BAA Must Include:**
- Permitted uses of PHI
- Safeguards to protect PHI
- Reporting requirements for breaches
- Return/destruction of PHI upon termination
- Subcontractor requirements

### Vendor Assessment

**Before Using Any Service:**
1. Verify HIPAA compliance
2. Request BAA
3. Review security practices
4. Assess data flow
5. Document decision

---

## Security Checklist

### Development Environment
- [ ] Use separate database for development
- [ ] Never use production data in development
- [ ] Use dummy/synthetic data for testing
- [ ] Secure development machines
- [ ] Code review for security issues

### Production Environment
- [ ] All data encrypted at rest
- [ ] All data encrypted in transit (HTTPS)
- [ ] Strong passwords enforced
- [ ] Multi-factor authentication available
- [ ] Regular security updates
- [ ] Automated backups configured
- [ ] Audit logging enabled
- [ ] Rate limiting configured
- [ ] Firewall rules configured
- [ ] SSL certificates valid
- [ ] BAAs signed with vendors
- [ ] Incident response plan documented
- [ ] Regular security audits scheduled

### Ongoing Maintenance
- [ ] Review audit logs weekly
- [ ] Update software monthly
- [ ] Review access quarterly
- [ ] Conduct security audit annually
- [ ] Train staff annually
- [ ] Test backups quarterly
- [ ] Test incident response annually

---

## Compliance Documentation

### Required Documents
1. **HIPAA Security Risk Assessment**
2. **Policies and Procedures Manual**
3. **Business Associate Agreements**
4. **Training Records**
5. **Audit Log Reports**
6. **Incident Response Plan**
7. **Breach Notification Procedures**
8. **Access Control Policies**

### Document Retention

**Keep for 6 years:**
- Security policies
- Training records
- Audit logs
- Incident reports
- Access reviews

---

## Penetration Testing

### Annual Security Assessment

**Recommended Tests:**
- SQL injection testing
- XSS vulnerability testing
- Authentication bypass attempts
- Session management testing
- API security testing
- Network penetration testing

**Tools:**
- OWASP ZAP (free)
- Burp Suite
- Nessus
- Metasploit

---

## Monitoring & Alerts

### CloudWatch Alarms

**Security-Related Alarms:**
```bash
# High failed login attempts
aws cloudwatch put-metric-alarm \
  --alarm-name high-failed-logins \
  --alarm-description "Alert on high failed login attempts" \
  --metric-name FailedLoginCount \
  --namespace Insurance/Security \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold

# Unusual API activity
aws cloudwatch put-metric-alarm \
  --alarm-name unusual-api-activity \
  --alarm-description "Alert on unusual API request volume" \
  --metric-name APIRequestCount \
  --namespace Insurance/API \
  --statistic Sum \
  --period 60 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold
```

---

## Encryption Key Management

### JWT Secret
```bash
# Generate strong JWT secret
openssl rand -base64 64

# Store in environment variable
JWT_SECRET="generated-secret-here"

# Rotate every 90 days
```

### Database Encryption
```bash
# AWS KMS for RDS encryption
# Automatic key rotation enabled
# Keys never exposed to application
```

---

## Disaster Recovery

### RPO & RTO Targets

**Recovery Point Objective (RPO):** 24 hours  
**Recovery Time Objective (RTO):** 4 hours

### Backup Strategy
- Database: Automated daily backups (7 days retention)
- Files: S3 versioning enabled
- Application: Git version control
- Configurations: Documented in deployment guide

### Disaster Recovery Testing
- Test quarterly
- Document results
- Update procedures as needed

---

## Compliance Summary

**HIPAA Compliance Status:**

✅ **Privacy Rule:**
- Minimum necessary access implemented
- Patient consent not required (treatment purposes)
- Privacy policies documented

✅ **Security Rule:**
- Administrative safeguards: Access control, training, audit
- Physical safeguards: AWS data centers, secure workstations
- Technical safeguards: Encryption, authentication, audit logs

✅ **Breach Notification:**
- Procedures documented
- Monitoring in place
- Response plan ready

---

## Document Information
- **Version:** 1.0
- **Last Updated:** January 22, 2026
- **Compliance Standard:** HIPAA
- **Status:** Complete