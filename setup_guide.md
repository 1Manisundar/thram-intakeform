# Development Environment Setup Guide

## Overview
This guide will help you set up your local development environment for the Insurance Verification System. Follow these steps in order.

**Estimated Setup Time:** 2-3 hours

---

## Prerequisites

### System Requirements
- **OS:** macOS, Linux, or Windows 10/11 with WSL2
- **RAM:** 8GB minimum (16GB recommended)
- **Storage:** 20GB free space
- **Internet:** Stable connection for downloading packages

---

## Step 1: Install Core Tools

### 1.1 Install Node.js (v18+ LTS)

**macOS (using Homebrew):**
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Node.js
brew install node@18
```

**Windows (using installer):**
1. Download from: https://nodejs.org/en/download/
2. Run installer and follow prompts
3. Choose "LTS" version (18.x or higher)

**Linux (Ubuntu/Debian):**
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

**Verify Installation:**
```bash
node --version  # Should show v18.x.x or higher
npm --version   # Should show v9.x.x or higher
```

---

### 1.2 Install Git

**macOS:**
```bash
brew install git
```

**Windows:**
Download from: https://git-scm.com/download/win

**Linux:**
```bash
sudo apt-get update
sudo apt-get install git
```

**Verify:**
```bash
git --version
```

---

### 1.3 Install PostgreSQL (v15+)

**macOS:**
```bash
brew install postgresql@15
brew services start postgresql@15
```

**Windows:**
1. Download from: https://www.postgresql.org/download/windows/
2. Run installer
3. Remember the password you set for 'postgres' user

**Linux:**
```bash
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**Verify:**
```bash
psql --version  # Should show 15.x
```

**Create Development Database:**
```bash
# Login to PostgreSQL
psql -U postgres

# Inside psql:
CREATE DATABASE insurance_verification_dev;
CREATE USER dev_user WITH PASSWORD 'dev_password';
GRANT ALL PRIVILEGES ON DATABASE insurance_verification_dev TO dev_user;
\q
```

---

### 1.4 Install Docker (for n8n)

**macOS:**
```bash
brew install --cask docker
# Open Docker Desktop after installation
```

**Windows:**
Download Docker Desktop: https://www.docker.com/products/docker-desktop/

**Linux:**
```bash
sudo apt-get update
sudo apt-get install docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
# Logout and login again
```

**Verify:**
```bash
docker --version
docker-compose --version
```

---

### 1.5 Install Visual Studio Code

Download from: https://code.visualstudio.com/

**Recommended Extensions:**
- Angular Language Service
- Prisma
- ESLint
- Prettier - Code formatter
- Thunder Client (API testing)
- Docker
- PostgreSQL (by Chris Kolkman)

---

## Step 2: Project Setup

### 2.1 Create Project Structure

```bash
# Create main project directory
mkdir insurance-verification-system
cd insurance-verification-system

# Create subdirectories
mkdir frontend backend n8n shared docs
```

---

### 2.2 Setup Backend (Node.js + Express + Prisma)

```bash
cd backend

# Initialize Node.js project
npm init -y

# Install dependencies
npm install express cors dotenv bcrypt jsonwebtoken
npm install @prisma/client
npm install multer helmet express-rate-limit
npm install axios zod winston

# Install dev dependencies
npm install -D typescript @types/node @types/express
npm install -D @types/cors @types/bcrypt @types/jsonwebtoken
npm install -D @types/multer ts-node nodemon prisma
npm install -D eslint prettier

# Initialize TypeScript
npx tsc --init
```

**Create `tsconfig.json`:**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

**Create folder structure:**
```bash
mkdir -p src/{routes,controllers,services,middleware,utils,config,types}
mkdir uploads  # For storing uploaded files locally
```

**Update `package.json` scripts:**
```json
{
  "scripts": {
    "dev": "nodemon src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "prisma:studio": "prisma studio"
  }
}
```

---

### 2.3 Setup Prisma

```bash
# Initialize Prisma
npx prisma init

# This creates:
# - prisma/schema.prisma
# - .env file
```

**Edit `.env` file:**
```env
# Database
DATABASE_URL="postgresql://dev_user:dev_password@localhost:5432/insurance_verification_dev?schema=public"

# JWT
JWT_SECRET="your-super-secret-jwt-key-change-in-production"
JWT_EXPIRES_IN="7d"

# Server
PORT=3000
NODE_ENV="development"

# File Upload
UPLOAD_DIR="./uploads"
MAX_FILE_SIZE=5242880  # 5MB in bytes

# n8n Webhook
N8N_WEBHOOK_URL="http://localhost:5678/webhook/insurance-verification"

# CORS
FRONTEND_URL="http://localhost:4200"
```

**Create `prisma/schema.prisma`:**
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

**Run migration:**
```bash
npx prisma migrate dev --name init
npx prisma generate
```

---

### 2.4 Setup Frontend (Angular)

```bash
cd ../frontend

# Install Angular CLI globally
npm install -g @angular/cli@17

# Create new Angular project
ng new insurance-verification-frontend --routing --style=scss

cd insurance-verification-frontend

# Install dependencies
npm install @angular/material @angular/cdk
npm install tailwindcss postcss autoprefixer
npx tailwindcss init
```

**Configure Tailwind CSS:**

Update `tailwind.config.js`:
```javascript
module.exports = {
  content: [
    "./src/**/*.{html,ts}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

Add to `src/styles.scss`:
```scss
@tailwind base;
@tailwind components;
@tailwind utilities;
```

**Create Angular modules:**
```bash
ng generate module modules/user-form --routing
ng generate module modules/admin-portal --routing
ng generate module shared

ng generate component modules/user-form/components/insurance-form
ng generate component modules/admin-portal/components/dashboard
ng generate component modules/admin-portal/components/verification-detail

ng generate service services/verification
ng generate service services/auth
```

---

### 2.5 Setup n8n (Docker)

```bash
cd ../n8n

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=admin123
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://localhost:5678/
      - GENERIC_TIMEZONE=America/New_York
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
EOF

# Start n8n
docker-compose up -d

# Check if running
docker-compose ps
```

**Access n8n:**
- URL: http://localhost:5678
- Username: admin
- Password: admin123

---

## Step 3: Verify Installation

### 3.1 Test Backend

```bash
cd backend

# Create a simple test file
cat > src/server.ts << 'EOF'
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';

dotenv.config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());

app.get('/health', (req, res) => {
  res.json({ 
    status: 'OK', 
    message: 'Backend is running',
    timestamp: new Date().toISOString()
  });
});

app.listen(PORT, () => {
  console.log(`🚀 Server running on http://localhost:${PORT}`);
});
EOF

# Run the server
npm run dev
```

**Test:** Open browser to http://localhost:3000/health

---

### 3.2 Test Frontend

```bash
cd ../frontend/insurance-verification-frontend

# Start Angular dev server
ng serve
```

**Test:** Open browser to http://localhost:4200

---

### 3.3 Test PostgreSQL Connection

```bash
cd backend

# Create test script
cat > test-db.ts << 'EOF'
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function testConnection() {
  try {
    await prisma.$connect();
    console.log('✅ Database connected successfully');
    
    const count = await prisma.insuranceVerification.count();
    console.log(`📊 Records in database: ${count}`);
    
  } catch (error) {
    console.error('❌ Database connection failed:', error);
  } finally {
    await prisma.$disconnect();
  }
}

testConnection();
EOF

# Run test
npx ts-node test-db.ts
```

---

### 3.4 Test n8n

1. Open http://localhost:5678
2. Login with admin/admin123
3. Create a new workflow
4. Add a "Webhook" trigger node
5. Set "HTTP Method" to POST
6. Set "Path" to `insurance-verification`
7. Click "Execute Workflow"
8. Copy the test webhook URL

**Test with curl:**
```bash
curl -X POST http://localhost:5678/webhook-test/insurance-verification \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

---

## Step 4: Project Configuration Files

### 4.1 Backend .gitignore

```bash
cd backend
cat > .gitignore << 'EOF'
# Dependencies
node_modules/

# Build
dist/

# Environment
.env
.env.local
.env.production

# Uploads
uploads/*
!uploads/.gitkeep

# Logs
*.log
logs/

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp
*.swo
EOF
```

### 4.2 Frontend .gitignore

```bash
cd ../frontend/insurance-verification-frontend
cat > .gitignore << 'EOF'
# Dependencies
node_modules/

# Build
/dist
/tmp
/out-tsc

# Environment
.env
.env.local

# IDE
.vscode/
.idea/

# OS
.DS_Store

# Angular
/.angular/cache
EOF
```

---

## Step 5: Create Helper Scripts

### 5.1 Backend Development Script

```bash
cd backend
cat > dev.sh << 'EOF'
#!/bin/bash
echo "🚀 Starting Backend Development Server..."
echo "📦 Installing dependencies..."
npm install
echo "🔄 Generating Prisma client..."
npx prisma generate
echo "🏃 Running migrations..."
npx prisma migrate dev
echo "✨ Starting server..."
npm run dev
EOF

chmod +x dev.sh
```

### 5.2 Database Reset Script

```bash
cat > reset-db.sh << 'EOF'
#!/bin/bash
echo "⚠️  WARNING: This will delete all data!"
read -p "Are you sure? (yes/no): " confirm
if [ "$confirm" = "yes" ]; then
  echo "🗑️  Resetting database..."
  npx prisma migrate reset --force
  echo "✅ Database reset complete"
else
  echo "❌ Cancelled"
fi
EOF

chmod +x reset-db.sh
```

---

## Step 6: Troubleshooting Common Issues

### Issue: PostgreSQL not starting

**macOS:**
```bash
brew services restart postgresql@15
```

**Linux:**
```bash
sudo systemctl restart postgresql
```

---

### Issue: Port already in use

**Kill process on port 3000:**
```bash
# macOS/Linux
lsof -ti:3000 | xargs kill -9

# Windows
netstat -ano | findstr :3000
taskkill /PID <PID_NUMBER> /F
```

---

### Issue: Prisma migration fails

```bash
# Reset Prisma
npx prisma migrate reset

# Or manually drop database and recreate
psql -U postgres
DROP DATABASE insurance_verification_dev;
CREATE DATABASE insurance_verification_dev;
\q

# Run migrations again
npx prisma migrate dev
```

---

### Issue: Docker not starting n8n

```bash
# Check Docker is running
docker ps

# Restart n8n
cd n8n
docker-compose down
docker-compose up -d

# Check logs
docker-compose logs -f
```

---

## Step 7: Verify Complete Setup

**Checklist:**
- [ ] Node.js v18+ installed
- [ ] PostgreSQL 15+ installed and running
- [ ] Docker installed and running
- [ ] Backend runs on http://localhost:3000
- [ ] Frontend runs on http://localhost:4200
- [ ] n8n runs on http://localhost:5678
- [ ] Database connection works
- [ ] Prisma migrations completed

---

## Next Steps

Once setup is complete:
1. Review `database-schema.md` for detailed schema information
2. Follow `api-documentation.md` to build API endpoints
3. Check `n8n-workflows.md` to create workflows
4. Start building features!

---

## Useful Commands Reference

```bash
# Backend
npm run dev              # Start dev server
npm run build            # Build for production
npx prisma studio        # Open Prisma GUI
npx prisma migrate dev   # Run migrations

# Frontend
ng serve                 # Start dev server
ng build                 # Build for production
ng test                  # Run tests

# n8n
docker-compose up -d     # Start n8n
docker-compose down      # Stop n8n
docker-compose logs -f   # View logs

# Database
psql -U postgres         # Connect to PostgreSQL
psql -U dev_user -d insurance_verification_dev  # Connect to dev DB
```

---

## Support

If you encounter issues not covered here, check:
- `troubleshooting.md` (coming soon)
- Project GitHub issues
- Stack Overflow with relevant tags

---

**Document Version:** 1.0  
**Last Updated:** January 22, 2026  
**Status:** Complete