# Accordo AI Chatbot - Installation Guide

**Complete installation and configuration guide**

Last Updated: January 4, 2026

---

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Backend Installation](#backend-installation)
3. [Frontend Installation](#frontend-installation)
4. [LLM Setup (Ollama)](#llm-setup-ollama)
5. [Configuration Files](#configuration-files)
6. [Verification Steps](#verification-steps)
7. [Common Installation Issues](#common-installation-issues)

---

## System Requirements

### Minimum Requirements

**Hardware**:
- CPU: 2+ cores
- RAM: 4GB minimum (8GB recommended for LLM)
- Disk: 10GB free space
- GPU: Optional (accelerates LLM inference)

**Software**:
- **Operating System**: macOS 10.15+, Ubuntu 20.04+, Windows 10+ (with WSL2)
- **Node.js**: v18.0.0 or higher
- **npm**: v9.0.0 or higher
- **PostgreSQL**: v14.0 or higher
- **Git**: v2.30.0 or higher
- **Ollama**: Latest version (for LLM features)

### Recommended Setup

- CPU: 4+ cores (Intel i5/AMD Ryzen 5 or better)
- RAM: 16GB
- SSD: For better database and LLM performance
- GPU: NVIDIA GPU with 4GB+ VRAM (for faster LLM inference)

---

## Backend Installation

### Step 1: Clone Repository

```bash
cd "/Users/safayavatsal/Downloads/Deuex/Accordo AI"
cd Accordo-ai-backend

# Or if cloning from Git
git clone <repository-url> Accordo-ai-backend
cd Accordo-ai-backend
```

### Step 2: Install Dependencies

```bash
# Install all npm packages
npm install

# This installs:
# - Express.js (web framework)
# - Sequelize (ORM)
# - PostgreSQL driver
# - JWT authentication
# - Axios (HTTP client)
# - Winston (logging)
# - And 50+ other dependencies
```

Expected output:
```
added 234 packages, and audited 235 packages in 45s
```

### Step 3: Database Setup

#### Install PostgreSQL

**macOS (Homebrew)**:
```bash
brew install postgresql@14
brew services start postgresql@14
```

**Ubuntu/Debian**:
```bash
sudo apt update
sudo apt install postgresql-14 postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**Windows** (WSL2):
```bash
sudo apt update
sudo apt install postgresql
sudo service postgresql start
```

#### Create Database

```bash
# Connect to PostgreSQL
psql -U postgres

# Create database
CREATE DATABASE accordo;

# Create user (optional)
CREATE USER accordo_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE accordo TO accordo_user;

# Exit
\q
```

#### Configure Database Connection

Edit `.env` (see Configuration Files section):

```env
DATABASE_URL=postgresql://accordo_user:your_secure_password@localhost:5432/accordo

# Or individual settings
DB_HOST=localhost
DB_PORT=5432
DB_NAME=accordo
DB_USER=accordo_user
DB_PASSWORD=your_secure_password
DB_DIALECT=postgres
```

### Step 4: Run Migrations

```bash
# Run all migrations to create tables
npm run migrate
```

Expected output:
```
Sequelize CLI [Node: 18.x.x, CLI: 6.6.3, ORM: 6.37.7]

Loaded configuration file "sequelize.config.cjs".
Using environment "development".

== 20260101000001-create-chatbot-templates: migrating =======
== 20260101000001-create-chatbot-templates: migrated (0.123s)

== 20260101000002-create-chatbot-template-parameters: migrating =======
== 20260101000002-create-chatbot-template-parameters: migrated (0.089s)

== 20260101000003-create-chatbot-deals: migrating =======
== 20260101000003-create-chatbot-deals: migrated (0.156s)

== 20260101000004-create-chatbot-messages: migrating =======
== 20260101000004-create-chatbot-messages: migrated (0.112s)

== 20260101000005-add-history-columns-to-chatbot-deals: migrating =======
== 20260101000005-add-history-columns-to-chatbot-deals: migrated (0.078s)
```

#### Verify Tables

```bash
psql -U accordo_user -d accordo

\dt chatbot*

# Should show:
#  Schema |              Name              | Type  |    Owner
# --------+--------------------------------+-------+-------------
#  public | chatbot_deals                  | table | accordo_user
#  public | chatbot_messages               | table | accordo_user
#  public | chatbot_template_parameters    | table | accordo_user
#  public | chatbot_templates              | table | accordo_user
```

### Step 5: Seed Data (Optional)

```bash
# Run seed files to populate initial data
npm run seed

# This creates:
# - Default negotiation templates
# - Sample users (for testing)
# - Example deals
```

### Step 6: Start Backend

```bash
# Development mode (with auto-reload)
npm run dev

# Production mode
npm run build
npm start
```

Expected output:
```
[2026-01-04 10:00:00] INFO: Server starting...
[2026-01-04 10:00:01] INFO: Database connected successfully
[2026-01-04 10:00:01] INFO: Models synchronized
[2026-01-04 10:00:02] INFO: Server running on port 8000
[2026-01-04 10:00:02] INFO: Environment: development
```

### Verify Backend

```bash
# Health check
curl http://localhost:8000/api/health

# Expected: {"status":"ok","timestamp":"2026-01-04T10:00:00.000Z"}

# Check chatbot routes
curl http://localhost:8000/api/chatbot/deals

# Expected: {"success":false,"message":"Unauthorized"} (auth required)
```

---

## Frontend Installation

### Step 1: Navigate to Frontend

```bash
cd ../Accordo-ai-frontend
```

### Step 2: Install Dependencies

```bash
npm install

# This installs:
# - React 19
# - Vite (build tool)
# - React Router v7
# - Tailwind CSS
# - Axios
# - Chart.js
# - jsPDF
# - And 100+ other packages
```

Expected output:
```
added 567 packages, and audited 568 packages in 1m 12s
```

### Step 3: Configure Environment

Copy template:
```bash
cp env.local.template .env.local
```

Edit `.env.local`:
```env
# Backend API URL
VITE_BACKEND_URL=http://localhost:8000

# Frontend URL (for generating links)
VITE_FRONTEND_URL=http://localhost:3000

# Asset/Upload URL
VITE_ASSEST_URL=http://localhost:8000

# Development Server
VITE_DEV_HOST=0.0.0.0
VITE_DEV_PORT=3000
```

### Step 4: Start Frontend

```bash
npm run dev
```

Expected output:
```
VITE v5.4.11  ready in 1234 ms

  ➜  Local:   http://localhost:3000/
  ➜  Network: http://192.168.1.100:3000/
  ➜  press h + enter to show help
```

### Step 5: Build for Production (Optional)

```bash
# Build static assets
npm run build

# Output directory: dist/
# Files: index.html, assets/index-*.js, assets/index-*.css

# Preview production build
npm run preview
```

### Verify Frontend

1. Open browser: `http://localhost:3000`
2. Should see Accordo landing page
3. Click "Sign In" - login page should load
4. Check browser console - no errors

---

## LLM Setup (Ollama)

### Step 1: Install Ollama

**macOS**:
```bash
# Download from ollama.ai or use Homebrew
brew install ollama
```

**Linux**:
```bash
curl -fsSL https://ollama.ai/install.sh | sh
```

**Windows**:
- Download installer from [ollama.ai](https://ollama.ai)
- Run installer
- Ollama will start automatically as a service

### Step 2: Start Ollama Server

```bash
# Start server (runs in background)
ollama serve

# Expected output:
# Ollama server listening on http://localhost:11434
```

### Step 3: Pull LLM Model

```bash
# Pull llama3.1 (recommended, ~4GB download)
ollama pull llama3.1

# Or other models:
ollama pull llama3.2        # Smaller, faster
ollama pull llama3.2:70b    # Larger, more accurate
ollama pull mistral         # Alternative model
```

Expected output:
```
pulling manifest
pulling c3068e6c03ce... 100%
pulling 8c17c2ebb0ea... 100%
pulling 4ce79e29bfb4... 100%
pulling eb85c1c45798... 100%
success
```

### Step 4: Verify Ollama

```bash
# List installed models
ollama list

# Expected output:
# NAME              ID            SIZE      MODIFIED
# llama3.1:latest   42ab0f6b0..   4.7 GB    2 minutes ago

# Test model
ollama run llama3.1 "Hello, how are you?"

# Should generate a response
```

### Step 5: Configure Backend

Edit `.env` in backend:

```env
CHATBOT_LLM_BASE_URL=http://localhost:11434
CHATBOT_LLM_MODEL=llama3.1
```

### Step 6: Test LLM Integration

```bash
# In backend directory
npm run dev

# Watch logs for:
# [INFO] LLM service initialized: http://localhost:11434 (model: llama3.1)

# Test via API (requires auth token):
curl -X POST http://localhost:8000/api/chatbot/conversation/deals/DEAL_ID/start \
  -H "Authorization: Bearer YOUR_TOKEN"

# Should receive greeting message generated by LLM
```

---

## Configuration Files

### Backend: `.env`

Create from template:
```bash
cd Accordo-ai-backend
cp .env.example .env
```

Complete configuration:

```env
# ============================================
# SERVER CONFIGURATION
# ============================================
NODE_ENV=development
PORT=8000

# ============================================
# DATABASE CONFIGURATION
# ============================================
DATABASE_URL=postgresql://accordo_user:password@localhost:5432/accordo

# Or individual settings:
DB_HOST=localhost
DB_PORT=5432
DB_NAME=accordo
DB_USER=accordo_user
DB_PASSWORD=your_secure_password
DB_DIALECT=postgres
DB_POOL_MAX=20
DB_POOL_MIN=0
DB_POOL_ACQUIRE=60000
DB_POOL_IDLE=10000

# ============================================
# JWT AUTHENTICATION
# ============================================
JWT_SECRET=your-super-secret-jwt-key-change-this-in-production
JWT_REFRESH_SECRET=your-super-secret-refresh-key-change-this-in-production
JWT_EXPIRES_IN=24h
JWT_REFRESH_EXPIRES_IN=7d

# ============================================
# LLM CONFIGURATION (Ollama)
# ============================================
CHATBOT_LLM_BASE_URL=http://localhost:11434
CHATBOT_LLM_MODEL=llama3.1
LLM_TIMEOUT=30000
LLM_MAX_TOKENS=500
LLM_TEMPERATURE=0.7

# ============================================
# EMAIL CONFIGURATION (Optional)
# ============================================
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
SMTP_FROM=noreply@accordo.ai

# ============================================
# FRONTEND URLS
# ============================================
VENDOR_PORTAL_URL=http://localhost:3000/vendor
CHATBOT_FRONTEND_URL=http://localhost:3000/chatbot

# ============================================
# LOGGING
# ============================================
LOG_LEVEL=info
LOG_FILE_PATH=./logs/app.log

# ============================================
# CORS
# ============================================
CORS_ORIGIN=http://localhost:3000
```

### Frontend: `.env.local`

```env
# Backend API URL
VITE_BACKEND_URL=http://localhost:8000

# Frontend URL
VITE_FRONTEND_URL=http://localhost:3000

# Asset URL
VITE_ASSEST_URL=http://localhost:8000

# Development Server
VITE_DEV_HOST=0.0.0.0
VITE_DEV_PORT=3000
```

### Important Notes

- Never commit `.env` or `.env.local` to version control
- Use strong, random secrets in production
- Change default passwords immediately
- Use environment-specific configs for staging/production

---

## Verification Steps

### 1. Backend Health Check

```bash
# Server running
curl http://localhost:8000/api/health

# Database connection
curl http://localhost:8000/api/chatbot/deals \
  -H "Authorization: Bearer test"  # Should get 401 if auth working

# LLM connection
# Check backend logs for:
# [INFO] LLM service initialized successfully
```

### 2. Frontend Connectivity

```bash
# Open browser console (F12)
# Navigate to http://localhost:3000
# Check Network tab - should see requests to backend
# Check Console - no CORS errors
```

### 3. Database Tables

```sql
psql -U accordo_user -d accordo

-- Check all tables exist
\dt

-- Should include:
-- Users, Contracts, Requisitions, etc.
-- chatbot_deals
-- chatbot_messages
-- chatbot_templates
-- chatbot_template_parameters

-- Check chatbot tables are empty
SELECT COUNT(*) FROM chatbot_deals;     -- 0
SELECT COUNT(*) FROM chatbot_messages;  -- 0
```

### 4. Authentication Flow

1. Navigate to `http://localhost:3000/sign-up`
2. Create account
3. Should redirect to dashboard
4. Check localStorage - should have `%accessToken%`
5. Logout - token should be cleared

### 5. Chatbot Creation

1. Navigate to `/chatbot`
2. Click "New Deal"
3. Fill form and create
4. Should redirect to negotiation room
5. Check database:
   ```sql
   SELECT id, title, status, mode FROM chatbot_deals LIMIT 1;
   ```

---

## Common Installation Issues

### Issue 1: npm install fails

**Error**: `npm ERR! code ERESOLVE`

**Solution**:
```bash
# Clear npm cache
npm cache clean --force

# Delete node_modules and package-lock.json
rm -rf node_modules package-lock.json

# Reinstall
npm install --legacy-peer-deps
```

---

### Issue 2: PostgreSQL connection refused

**Error**: `Error: connect ECONNREFUSED 127.0.0.1:5432`

**Solution**:
```bash
# Check PostgreSQL is running
brew services list | grep postgresql
# or
sudo systemctl status postgresql

# Start PostgreSQL
brew services start postgresql@14
# or
sudo systemctl start postgresql

# Check port
lsof -i :5432  # Should show postgres process
```

---

### Issue 3: Migrations fail

**Error**: `ERROR: relation "chatbot_deals" already exists`

**Solution**:
```bash
# Drop and recreate database
psql -U postgres
DROP DATABASE accordo;
CREATE DATABASE accordo;
GRANT ALL PRIVILEGES ON DATABASE accordo TO accordo_user;
\q

# Re-run migrations
npm run migrate
```

---

### Issue 4: Ollama not found

**Error**: `Error: connect ECONNREFUSED 127.0.0.1:11434`

**Solution**:
```bash
# Check Ollama is installed
which ollama

# If not installed:
brew install ollama  # macOS
# or
curl -fsSL https://ollama.ai/install.sh | sh  # Linux

# Start Ollama
ollama serve

# Verify
curl http://localhost:11434/api/tags
```

---

### Issue 5: Port already in use

**Error**: `Error: listen EADDRINUSE: address already in use :::8000`

**Solution**:
```bash
# Find process using port
lsof -ti:8000

# Kill process
lsof -ti:8000 | xargs kill -9

# Or change port in .env
PORT=8001
```

---

### Issue 6: CORS errors in browser

**Error**: `Access to XMLHttpRequest at 'http://localhost:8000' from origin 'http://localhost:3000' has been blocked by CORS policy`

**Solution**:
```env
# In backend .env, add:
CORS_ORIGIN=http://localhost:3000

# Or allow all (development only):
CORS_ORIGIN=*
```

Restart backend after changing.

---

### Issue 7: Module not found errors

**Error**: `Error [ERR_MODULE_NOT_FOUND]: Cannot find module`

**Solution**:
```bash
# Backend (TypeScript)
npm run build  # Compile first
npm start      # Then run

# Or use dev mode (no build needed)
npm run dev

# Frontend
rm -rf node_modules .vite
npm install
npm run dev
```

---

### Issue 8: LLM timeouts

**Error**: `LLM request timed out after 30000ms`

**Solution**:
```env
# Increase timeout in .env
LLM_TIMEOUT=60000

# Or use smaller model
CHATBOT_LLM_MODEL=llama3.2

# Or use GPU acceleration (if available)
# Ollama automatically uses GPU if detected
```

---

## Post-Installation

After successful installation:

1. **Create Admin User**:
   ```bash
   # Via seed script or directly in database
   npm run seed:admin
   ```

2. **Configure Email** (Optional):
   - Set up SMTP credentials in `.env`
   - Test: Create contract, check vendor receives email

3. **Setup Monitoring** (Production):
   - Configure Winston logging
   - Set up error tracking (Sentry, etc.)
   - Monitor LLM response times

4. **Backup Database**:
   ```bash
   pg_dump -U accordo_user accordo > backup.sql
   ```

5. **Test End-to-End**:
   - Create account
   - Create deal
   - Send messages
   - Check database for records
   - Test all lifecycle operations

---

## Next Steps

- Read [ARCHITECTURE.md](./ARCHITECTURE.md) for system overview
- Check [API_ENDPOINTS.md](./API_ENDPOINTS.md) for API documentation
- Review [PHASE1_DEMO_FUNCTIONALITY.md](./PHASE1_DEMO_FUNCTIONALITY.md) for demo mode
- See [PHASE2_CONVERSATION_SYSTEM.md](./PHASE2_CONVERSATION_SYSTEM.md) for conversation mode

---

**Installation Support**:
- For detailed troubleshooting: [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)
- For deployment: [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md)
- For testing: [TESTING_GUIDE.md](./TESTING_GUIDE.md)
