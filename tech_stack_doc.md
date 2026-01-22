# Insurance Verification System - Technology Stack

## Overview
This document outlines the complete technology stack for the Insurance Verification System, designed to automate insurance eligibility verification with HIPAA compliance.

---

## Frontend Stack

### Core Framework
- **Framework:** Angular 17+
- **Language:** TypeScript 5.3+
- **Package Manager:** npm / pnpm

### UI & Styling
- **Styling Framework:** Tailwind CSS / Angular Material
- **Component Library:** Angular Material (optional)
- **Icons:** Lucide Angular / Material Icons

### State Management & Data Flow
- **State Management:** RxJS (Angular built-in)
- **Form Handling:** Angular Reactive Forms
- **HTTP Client:** Angular HttpClient
- **Routing:** Angular Router

### Development Tools
- **Build Tool:** Angular CLI
- **TypeScript Compiler:** tsc
- **Linting:** ESLint
- **Formatting:** Prettier

---

## Backend Stack

### Core Framework
- **Runtime:** Node.js v18+ LTS
- **Framework:** Express.js 4.18+
- **Language:** TypeScript 5.3+

### Database & ORM
- **Database:** PostgreSQL 15+
- **ORM:** Prisma 5.7+
- **Database Client:** Prisma Client
- **Migrations:** Prisma Migrate

### API & Middleware
- **Validation:** Zod 3.22+ / express-validator
- **Authentication:** JWT (jsonwebtoken 9.0+)
- **Password Hashing:** bcrypt 5.1+
- **CORS:** cors 2.8+
- **Security Headers:** Helmet.js 7.1+
- **Rate Limiting:** express-rate-limit 7.1+

### File Handling
- **File Upload:** Multer 1.4+
- **File Storage:** AWS S3 / Cloudinary

### HTTP Client
- **External APIs:** Axios 1.6+

### Logging & Monitoring
- **Application Logging:** Winston 3.11+ / Pino
- **Error Tracking:** Sentry (optional)

### Environment & Configuration
- **Environment Variables:** dotenv 16.3+
- **Configuration Management:** Custom config service

### API Documentation
- **API Docs:** Swagger / OpenAPI (optional)

---

## Database

### Primary Database
- **Database System:** PostgreSQL 15+
- **Hosting Options:** 
  - AWS RDS PostgreSQL
  - Supabase
  - Self-hosted on VPS

### Database Features
- **JSONB Support:** For storing Stedi API responses
- **Encryption:** PostgreSQL encryption at rest
- **Indexing:** B-tree indexes for performance
- **Connection Pooling:** PgBouncer (optional)

### Database Management Tools
- **GUI Tools:** pgAdmin / DBeaver / TablePlus
- **CLI:** psql
- **Migrations:** Prisma Migrate

---

## Workflow Automation

### n8n Platform
- **Platform:** n8n (self-hosted)
- **Version:** Latest stable
- **Deployment:** Docker container
- **Hosting:** VPS (DigitalOcean / AWS EC2)

### n8n Features Used
- **Webhook Triggers:** For receiving form submissions
- **HTTP Request Nodes:** For API calls (Mindee, Stedi)
- **PostgreSQL Nodes:** For database operations
- **Code Nodes:** For custom JavaScript logic
- **Error Handling:** Try-catch blocks and error workflows
- **Retry Logic:** Built-in retry mechanisms

---

## External Services & APIs

### OCR Service
- **Provider:** Mindee API
- **Purpose:** Extract data from insurance card images
- **Integration:** REST API via n8n HTTP Request node

### Insurance Verification
- **Provider:** Stedi Clearinghouse API
- **Purpose:** Real-time insurance eligibility verification
- **Integration:** REST API via n8n HTTP Request node

### File Storage
- **Primary:** AWS S3
- **Alternative:** Cloudinary
- **Purpose:** Store uploaded insurance card images

### Email Service
- **Options:** 
  - SendGrid
  - AWS SES
  - Mailgun
- **Purpose:** Send notifications to users and admins

### SMS Service (Optional)
- **Options:**
  - Twilio
  - AWS SNS
- **Purpose:** Urgent notifications to users

---

## Development Tools

### Code Editor & IDE
- **Primary:** Visual Studio Code
- **Extensions:**
  - Angular Language Service
  - Prisma
  - ESLint
  - Prettier
  - Thunder Client / REST Client

### Version Control
- **System:** Git
- **Repository Hosting:** GitHub / GitLab / Bitbucket
- **Branching Strategy:** GitFlow / Trunk-based

### API Testing
- **Tools:**
  - Postman
  - Thunder Client (VS Code extension)
  - Insomnia
  - cURL

### Database Tools
- **GUI Clients:**
  - pgAdmin 4
  - DBeaver
  - TablePlus
- **CLI:** psql

### Package Management
- **Backend:** npm / pnpm
- **Frontend:** npm / pnpm

---

## Deployment & Infrastructure

### Hosting Platforms

#### Backend API
- **Options:**
  - AWS EC2
  - DigitalOcean Droplets
  - Railway
  - Render
  - Heroku (legacy)
- **Recommended:** DigitalOcean or AWS EC2

#### Frontend Application
- **Options:**
  - Vercel
  - Netlify
  - AWS S3 + CloudFront
  - DigitalOcean App Platform
- **Recommended:** Vercel (easiest) or AWS S3 + CloudFront (HIPAA compliant)

#### Database
- **Options:**
  - AWS RDS PostgreSQL
  - Supabase
  - DigitalOcean Managed Database
  - Self-hosted on VPS
- **Recommended:** AWS RDS (HIPAA compliant)

#### n8n Workflow Engine
- **Deployment:** Docker container on VPS
- **Hosting:** DigitalOcean / AWS EC2
- **Requirements:** Must be self-hosted (not n8n cloud) for HIPAA compliance

### Containerization & Orchestration
- **Containerization:** Docker
- **Orchestration:** Docker Compose
- **Registry:** Docker Hub / AWS ECR

### Web Server & Reverse Proxy
- **Reverse Proxy:** Nginx
- **Load Balancer:** Nginx / AWS ALB (for scaling)

### SSL/TLS Certificates
- **Provider:** Let's Encrypt (free)
- **Management:** Certbot
- **Renewal:** Automatic via cron jobs

### CI/CD Pipeline
- **Options:**
  - GitHub Actions
  - GitLab CI/CD
  - Jenkins
  - CircleCI
- **Recommended:** GitHub Actions (if using GitHub)

### Environment Management
- **Development:** Local machine
- **Staging:** Separate server/environment
- **Production:** Production server with backups

---

## Security & Compliance (HIPAA)

### Encryption
- **In Transit:** SSL/TLS (HTTPS everywhere)
- **At Rest:** PostgreSQL encryption, encrypted S3 buckets
- **Application Level:** bcrypt for passwords, JWT for sessions

### Authentication & Authorization
- **Method:** JWT tokens
- **Storage:** HTTP-only cookies / Local storage (with caution)
- **Password Security:** bcrypt hashing (salt rounds: 10-12)
- **Role-Based Access:** Admin vs User roles

### Security Headers & Middleware
- **Headers:** Helmet.js (sets secure HTTP headers)
- **CORS:** Configured to allow only trusted origins
- **Rate Limiting:** Prevent brute force attacks
- **Input Validation:** Zod schemas for all inputs
- **SQL Injection Prevention:** Prisma ORM (parameterized queries)
- **XSS Prevention:** Input sanitization, CSP headers

### Secrets Management
- **Development:** .env files (not committed to git)
- **Production:** Environment variables on server
- **Options:** AWS Secrets Manager / HashiCorp Vault (for advanced setups)

### Audit Logging
- **Backend Logs:** Winston / Pino
- **Log Storage:** File system / CloudWatch / LogDNA
- **What to Log:**
  - All PHI access (who, when, what)
  - Authentication attempts
  - API errors
  - Database queries (in development only)

### HIPAA Compliance Requirements
- ✅ Business Associate Agreements (BAA) with Mindee and Stedi
- ✅ Self-hosted n8n (no third-party cloud)
- ✅ Encrypted data at rest and in transit
- ✅ Access controls and audit logs
- ✅ Regular backups with encryption
- ✅ Automatic session timeout
- ✅ Multi-factor authentication for admins (optional but recommended)

---

## Monitoring & Logging

### Application Monitoring
- **Backend Logging:** Winston / Pino
- **Frontend Errors:** Browser console / Sentry
- **Log Levels:** Error, Warn, Info, Debug

### Uptime Monitoring
- **Services:**
  - UptimeRobot (free)
  - Pingdom
  - AWS CloudWatch
- **Alerts:** Email / SMS when services go down

### Performance Monitoring
- **Options:**
  - New Relic
  - Datadog
  - AWS CloudWatch
- **Metrics:** Response times, error rates, throughput

### Error Tracking
- **Service:** Sentry
- **Purpose:** Capture and track errors in production
- **Integration:** Backend + Frontend

---

## Backup & Disaster Recovery

### Database Backups
- **Frequency:** Daily automated backups
- **Retention:** 30 days minimum
- **Storage:** AWS S3 / DigitalOcean Spaces
- **Encryption:** Encrypted backups

### File Storage Backups
- **Insurance Card Images:** S3 versioning enabled
- **Backup Location:** Separate S3 bucket / region

### Application Backups
- **Code:** Version controlled in Git
- **Configuration:** Documented in deployment guide
- **n8n Workflows:** Exported JSON files (version controlled)

---

## Testing Tools

### Backend Testing
- **Unit Tests:** Jest / Mocha
- **Integration Tests:** Supertest
- **Test Coverage:** Istanbul / nyc

### Frontend Testing
- **Unit Tests:** Jasmine / Jest
- **E2E Tests:** Cypress / Playwright
- **Component Testing:** Angular Testing Library

### API Testing
- **Manual:** Postman collections
- **Automated:** Jest + Supertest

---

## Shared Code & Types

### TypeScript Shared Types
- **Location:** `/shared` directory
- **Purpose:** Share type definitions between frontend and backend
- **Examples:**
  - User submission types
  - Verification status enums
  - API response types
  - Database models

### Package Structure
```
shared/
├── types/
│   ├── user.types.ts
│   ├── verification.types.ts
│   └── api-response.types.ts
└── enums/
    └── verification-status.enum.ts
```

---

## Development Environment Requirements

### System Requirements
- **OS:** macOS / Linux / Windows (with WSL2)
- **RAM:** 8GB minimum (16GB recommended)
- **Storage:** 20GB free space

### Software Prerequisites
- **Node.js:** v18+ LTS
- **npm:** v9+ (comes with Node.js)
- **PostgreSQL:** v15+
- **Git:** Latest version
- **Docker:** Latest version (for n8n)
- **VS Code:** Latest version

### Browser Requirements
- **Development:** Chrome / Firefox (latest)
- **Testing:** Chrome, Firefox, Safari, Edge

---

## Production Environment Requirements

### Server Specifications
- **Backend Server:**
  - CPU: 2+ cores
  - RAM: 4GB minimum
  - Storage: 50GB SSD
  - OS: Ubuntu 22.04 LTS

- **n8n Server:**
  - CPU: 2 cores
  - RAM: 2GB minimum
  - Storage: 20GB SSD

### Network Requirements
- **SSL Certificate:** Required for HTTPS
- **Domain Name:** Required for production
- **Firewall:** Configured to allow only necessary ports
- **Backup Internet:** Recommended for high availability

---

## Scalability Considerations

### Horizontal Scaling Options
- **Backend:** Multiple instances behind load balancer
- **Database:** Read replicas for read-heavy workloads
- **File Storage:** CDN (CloudFront) for faster delivery
- **n8n:** Multiple n8n instances (requires separate queue)

### Vertical Scaling Options
- **Increase server resources:** More CPU, RAM, storage
- **Database optimization:** Better indexing, query optimization
- **Caching:** Redis for frequently accessed data

---

## Performance Optimization

### Frontend Optimization
- **Lazy Loading:** Load modules on demand
- **Tree Shaking:** Remove unused code
- **Minification:** Reduce file sizes
- **Caching:** Browser caching for static assets
- **CDN:** Serve assets from CDN

### Backend Optimization
- **Database Indexing:** Index frequently queried columns
- **Connection Pooling:** Reuse database connections
- **Caching:** Cache API responses (Redis)
- **Compression:** Gzip responses
- **Query Optimization:** Efficient Prisma queries

### API Optimization
- **Pagination:** Limit results per request
- **Rate Limiting:** Prevent API abuse
- **Response Compression:** Reduce payload size
- **Async Processing:** Offload heavy tasks to background

---

## Documentation Tools

### Code Documentation
- **TypeScript:** TSDoc comments
- **API Documentation:** Swagger / OpenAPI spec
- **README Files:** Markdown in each directory

### User Documentation
- **Admin Guide:** How to use admin portal
- **Deployment Guide:** Step-by-step deployment instructions
- **API Guide:** API endpoints and usage

---

## Version Control Strategy

### Branching Model
```
main/master          → Production code
├── develop          → Development integration
│   ├── feature/*    → New features
│   ├── bugfix/*     → Bug fixes
│   └── hotfix/*     → Emergency fixes
```

### Commit Convention
- **Format:** `type(scope): message`
- **Types:** feat, fix, docs, style, refactor, test, chore
- **Example:** `feat(auth): add JWT authentication`

---

## Tech Stack Summary

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | Angular + TypeScript | User interface |
| Backend | Node.js + Express + TypeScript | REST API |
| Database | PostgreSQL + Prisma | Data persistence |
| Workflow | n8n (self-hosted) | Automation engine |
| OCR | Mindee API | Extract card data |
| Insurance | Stedi API | Verify eligibility |
| File Storage | AWS S3 | Store documents |
| Email | SendGrid / AWS SES | Notifications |
| Hosting | AWS / DigitalOcean | Infrastructure |
| SSL | Let's Encrypt | Security |
| Monitoring | Winston / Sentry | Logging & errors |

---

## Document Information
- **Version:** 1.0
- **Last Updated:** January 22, 2026
- **Maintained By:** Development Team
- **Status:** Approved