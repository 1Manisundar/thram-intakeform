# Insurance Verification System

> Automated insurance eligibility verification system to streamline healthcare admin workflows

## 📋 Overview

The Insurance Verification System automates the manual process of verifying patient insurance eligibility. Instead of admins manually reviewing insurance cards and calling providers, the system uses OCR and real-time API verification to automatically validate insurance information.

**Problem Solved:** Healthcare staff spend hours manually verifying insurance details before appointments.

**Solution:** Automated workflow that processes insurance cards, extracts data via OCR, verifies eligibility through clearinghouse APIs, and presents admins with pre-verified information ready for patient contact.

---

## ✨ Key Features

### For Users (Patients)
- 📝 Simple form to submit insurance information
- 📸 Upload insurance card images
- ⚡ Instant confirmation that submission is being reviewed
- 📧 Email notifications if additional information needed

### For Admins
- 📊 Dashboard showing all submissions with real-time status
- ✅ Pre-verified insurance information (eligibility, coverage, plan details)
- 🔍 Filter and search capabilities
- 📝 Add notes and track contact history
- 🚦 Priority-based workflow management

### Automated Processing
- 🤖 OCR extraction of insurance card data (Mindee API)
- 🔄 Real-time eligibility verification (Stedi API)
- ✨ Intelligent error handling and retry logic
- 📋 Automatic status tracking throughout workflow

---

## 🏗️ Architecture

```
┌─────────────┐
│   User      │
│   (Web)     │
└──────┬──────┘
       │ Submit Form
       ↓
┌─────────────┐
│   Angular   │  ← Frontend
│   Frontend  │
└──────┬──────┘
       │ HTTP/REST
       ↓
┌─────────────┐
│   Express   │  ← Backend API
│   Backend   │
└──────┬──────┘
       │ Triggers
       ↓
┌─────────────┐
│     n8n     │  ← Workflow Automation
│  Workflows  │
└──┬────┬─────┘
   │    │
   │    └─────────┐
   ↓              ↓
┌──────┐      ┌────────┐
│Mindee│      │ Stedi  │  ← External APIs
│ OCR  │      │  API   │
└──────┘      └────────┘
```

---

## 🛠️ Tech Stack

### Frontend
- **Framework:** Angular 17+
- **Language:** TypeScript
- **Styling:** Tailwind CSS
- **State:** RxJS

### Backend
- **Runtime:** Node.js 18+
- **Framework:** Express.js
- **Language:** TypeScript
- **ORM:** Prisma
- **Database:** PostgreSQL 15+

### Automation
- **Platform:** n8n (self-hosted)
- **Deployment:** Docker

### External Services
- **OCR:** Mindee API
- **Insurance Verification:** Stedi API
- **File Storage:** AWS S3 (production) / Local (development)
- **Email:** AWS SES (production) / Console (development)

---

## 📁 Project Structure

```
insurance-verification-system/
│
├── frontend/                       # Angular Application
│   └── src/
│       ├── app/
│       │   ├── modules/
│       │   │   ├── user-form/     # User submission form
│       │   │   └── admin-portal/  # Admin dashboard
│       │   ├── services/          # API services
│       │   └── shared/            # Shared components
│       └── environments/          # Environment configs
│
├── backend/                        # Node.js + Express API
│   ├── src/
│   │   ├── routes/               # API routes
│   │   ├── controllers/          # Request handlers
│   │   ├── services/             # Business logic
│   │   ├── middleware/           # Auth, validation
│   │   └── utils/                # Helper functions
│   ├── prisma/
│   │   └── schema.prisma         # Database schema
│   └── uploads/                  # Local file storage
│
├── n8n/                           # Workflow automation
│   ├── workflows/                # n8n workflow JSONs
│   └── docker-compose.yml        # n8n Docker setup
│
├── shared/                        # Shared TypeScript types
│   └── types/
│
└── docs/                          # Documentation
    ├── requirements.md
    ├── TechStack.md
    ├── setup-guide.md
    ├── database-schema.md
    ├── api-documentation.md
    ├── n8n-workflows.md
    └── deployment-guide.md
```

---

## 🚀 Quick Start

### Prerequisites
- Node.js 18+ LTS
- PostgreSQL 15+
- Docker (for n8n)
- Git

### Installation

1. **Clone the repository**
```bash
git clone <repository-url>
cd insurance-verification-system
```

2. **Setup Backend**
```bash
cd backend
npm install
cp .env.example .env
# Edit .env with your database credentials
npx prisma migrate dev
npx prisma generate
npm run dev
```

3. **Setup Frontend**
```bash
cd frontend
npm install
ng serve
```

4. **Setup n8n**
```bash
cd n8n
docker-compose up -d
```

5. **Access Applications**
- Frontend: http://localhost:4200
- Backend: http://localhost:3000
- n8n: http://localhost:5678

📚 **Detailed setup instructions:** See [setup-guide.md](docs/setup-guide.md)

---

## 📖 Documentation

| Document | Description |
|----------|-------------|
| [requirements.md](docs/requirements.md) | Business requirements and workflow |
| [TechStack.md](docs/TechStack.md) | Complete technology stack |
| [setup-guide.md](docs/setup-guide.md) | Development environment setup |
| [database-schema.md](docs/database-schema.md) | Database design and schema |
| [api-documentation.md](docs/api-documentation.md) | REST API endpoints |
| [n8n-workflows.md](docs/n8n-workflows.md) | Workflow automation logic |
| [deployment-guide.md](docs/deployment-guide.md) | Production deployment |
| [security-compliance.md](docs/security-compliance.md) | HIPAA compliance |

---

## 🔄 Workflow Overview

### User Submission Flow
1. User fills form with personal info and uploads insurance card
2. System responds immediately: "Our agent will review and get back to you"
3. Data stored in database with status: `need_to_be_verified`

### Automated Verification Flow
4. n8n workflow triggered automatically
5. OCR extracts data from insurance card (Mindee)
6. System compares OCR data with user-submitted data
7. If data matches, calls Stedi API for eligibility verification
8. Stedi returns: eligibility status, plan details, coverage info
9. Database updated with verification results
10. Status changed to `data_verified`

### Admin Action Flow
11. Admin sees verified submission in dashboard
12. Admin reviews pre-verified information
13. Admin contacts user to discuss next steps
14. Admin marks as "contacted" and adds notes

### Error Handling
- **OCR fails** → Status: `ocr_failed`, User notified to upload clearer images
- **Data mismatch** → Status: `data_mismatch`, User asked to verify info
- **API error** → Automatic retry with exponential backoff (3 attempts)
- **Insurance invalid** → Admin handles cash payment discussion

---

## 🎯 Key Status States

| Status | Meaning | Next Action |
|--------|---------|-------------|
| `need_to_be_verified` | Just submitted | Automated processing |
| `ocr_in_progress` | OCR processing | Wait |
| `stedi_in_progress` | API verification | Wait |
| `data_verified` | ✅ Verified successfully | Admin contacts user |
| `ocr_failed` | ❌ Bad image quality | User resubmits images |
| `data_mismatch` | ❌ Data doesn't match | User verifies info |
| `insurance_invalid` | ⚠️ Insurance not active | Admin discusses payment |
| `api_error_manual_review` | ⚠️ API issues | Admin manually verifies |

---

## 🔐 Security & Compliance

### HIPAA Compliance
- ✅ Encrypted data at rest and in transit
- ✅ Audit logging of all PHI access
- ✅ Self-hosted n8n (no third-party cloud)
- ✅ Business Associate Agreements with vendors
- ✅ Role-based access control
- ✅ Automatic session timeout

### Security Measures
- JWT-based authentication
- bcrypt password hashing
- Rate limiting on API endpoints
- Input validation and sanitization
- SQL injection prevention via Prisma ORM
- CORS configuration
- Security headers via Helmet.js

📚 **Full details:** [security-compliance.md](docs/security-compliance.md)

---

## 🧪 Testing

```bash
# Backend tests
cd backend
npm test

# Frontend tests
cd frontend
ng test

# E2E tests
npm run e2e
```

---

## 📊 API Endpoints

### Public Endpoints
- `POST /api/verification/submit` - Submit insurance verification
- `POST /api/auth/forgot-password` - Request password reset

### Admin Endpoints (Requires Auth)
- `GET /api/verifications` - Get all verifications (with filters)
- `GET /api/verifications/:id` - Get single verification
- `PUT /api/verifications/:id/status` - Update status
- `POST /api/verifications/:id/notes` - Add admin note
- `PUT /api/verifications/:id/contact` - Mark as contacted

### Auth Endpoints
- `POST /api/auth/login` - Admin login
- `POST /api/auth/logout` - Admin logout
- `GET /api/auth/me` - Get current admin

📚 **Full API documentation:** [api-documentation.md](docs/api-documentation.md)

---

## 🚢 Deployment

### Development
- Frontend: `ng serve` on localhost:4200
- Backend: `npm run dev` on localhost:3000
- Database: PostgreSQL locally
- n8n: Docker on localhost:5678

### Production (AWS)
- Frontend: S3 + CloudFront
- Backend: EC2 instance
- Database: RDS PostgreSQL
- n8n: EC2 instance with Docker
- Files: S3 bucket
- Email: AWS SES

📚 **Deployment guide:** [deployment-guide.md](docs/deployment-guide.md)

---

## 🎨 Environment Variables

### Backend (.env)
```env
# Database
DATABASE_URL="postgresql://user:password@localhost:5432/dbname"

# JWT
JWT_SECRET="your-secret-key"
JWT_EXPIRES_IN="7d"

# Server
PORT=3000
NODE_ENV="development"

# n8n
N8N_WEBHOOK_URL="http://localhost:5678/webhook/insurance-verification"

# AWS (Production only)
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
AWS_S3_BUCKET=""
AWS_REGION="us-east-1"
```

### Frontend (environment.ts)
```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api'
};
```

---

## 📈 Performance Metrics

### Target Goals
- **Automation Rate:** > 80%
- **Verification Success:** > 90%
- **Processing Time:** < 2 minutes
- **OCR Failure Rate:** < 5%
- **API Response Time:** < 500ms

---

## 🤝 Contributing

### Development Workflow
1. Create feature branch: `git checkout -b feature/your-feature`
2. Make changes and commit: `git commit -m "feat: add feature"`
3. Push to branch: `git push origin feature/your-feature`
4. Create pull request

### Commit Convention
- `feat:` New feature
- `fix:` Bug fix
- `docs:` Documentation
- `style:` Formatting
- `refactor:` Code restructuring
- `test:` Tests
- `chore:` Maintenance

---

## 📝 License

[Add your license here]

---

## 👤 Author

**Solo Developer Project**

---

## 🆘 Support

For issues and questions:
1. Check documentation in `/docs`
2. Review [troubleshooting.md](docs/troubleshooting.md)
3. Check existing GitHub issues
4. Create new issue with detailed description

---

## 🗺️ Roadmap

### Phase 1 - MVP (Current)
- [x] Basic user form
- [x] Admin dashboard
- [x] OCR integration
- [x] Stedi API integration
- [x] Email notifications
- [x] Basic error handling

### Phase 2 - Enhancements
- [ ] SMS notifications
- [ ] Multi-factor authentication
- [ ] Advanced analytics dashboard
- [ ] Export to CSV
- [ ] Batch processing
- [ ] Mobile app

### Phase 3 - Advanced
- [ ] Machine learning for OCR improvement
- [ ] Integration with scheduling systems
- [ ] Patient portal
- [ ] Multi-language support
- [ ] Advanced reporting

---

## 📊 Project Stats

- **Lines of Code:** TBD
- **Test Coverage:** TBD
- **Dependencies:** ~30 packages
- **API Endpoints:** ~15
- **Database Tables:** 3

---

## 🙏 Acknowledgments

- n8n for workflow automation platform
- Mindee for OCR services
- Stedi for insurance verification API
- Angular and Node.js communities

---

**Built with ❤️ for healthcare workflow automation**

---

**Last Updated:** January 22, 2026  
**Version:** 1.0.0  
**Status:** In Development