# Accordo AI Chatbot - Quick Start Guide

**Get up and running in 5 minutes**

Last Updated: January 4, 2026

---

## Prerequisites

Before you begin, ensure you have:

- Node.js 18+ and npm installed
- PostgreSQL 14+ running locally
- Git installed
- Ollama installed (for LLM features)
- A terminal/command line interface

---

## Backend Setup (5 Steps)

### Step 1: Clone and Install

```bash
cd "Accordo AI/Accordo-ai-backend"
npm install
```

### Step 2: Configure Environment

```bash
cp .env.example .env
```

Edit `.env` with your settings:

```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/accordo

# Server
PORT=8000

# JWT (use secure random strings in production)
JWT_SECRET=your-secret-key-here
JWT_REFRESH_SECRET=your-refresh-secret-here

# LLM (Ollama)
CHATBOT_LLM_BASE_URL=http://localhost:11434
CHATBOT_LLM_MODEL=llama3.1
```

### Step 3: Setup Database

```bash
# Run migrations to create tables
npm run migrate
```

### Step 4: Start Ollama

```bash
# In a new terminal
ollama serve

# Pull the model
ollama pull llama3.1
```

### Step 5: Start Backend

```bash
npm run dev
```

You should see:
```
Server running on port 8000
Database connected successfully
```

---

## Frontend Setup (5 Steps)

### Step 1: Navigate to Frontend

```bash
cd "../Accordo-ai-frontend"
npm install
```

### Step 2: Configure Environment

```bash
cp env.local.template .env.local
```

Edit `.env.local`:

```env
VITE_BACKEND_URL=http://localhost:8000
VITE_FRONTEND_URL=http://localhost:3000
VITE_ASSEST_URL=http://localhost:8000
VITE_DEV_HOST=0.0.0.0
VITE_DEV_PORT=3000
```

### Step 3: Start Frontend

```bash
npm run dev
```

### Step 4: Open Browser

Navigate to: `http://localhost:3000`

### Step 5: Create Account

1. Click "Sign Up"
2. Fill in your details
3. Click "Create Account"

---

## First Test: Demo Scenario

Test the chatbot with a pre-configured demo:

### 1. Create a New Deal

1. Navigate to `/chatbot` (or click "Negotiation Chatbot" in sidebar)
2. Click "New Deal" button
3. Fill in:
   - **Title**: "Office Supplies Q1"
   - **Counterparty**: "ABC Vendor"
   - **Mode**: Select "INSIGHTS"
4. Click "Create Deal"

### 2. Run Demo Scenario

On the negotiation room page:

1. Click one of the scenario chips:
   - **HARD**: Resistant vendor (small concessions)
   - **SOFT**: Flexible vendor (larger concessions)
   - **WALK_AWAY**: Inflexible vendor (refuses to negotiate)

2. Watch the automated negotiation unfold
3. View decision badges and utility scores on each message

### Expected Results

**SOFT Scenario**:
- Vendor starts at $105
- Accordo counters $95
- After 3-4 rounds, deal accepted around $97-100
- Final status: ACCEPTED (green badge)

**HARD Scenario**:
- Vendor starts at $115
- Small concessions ($1-2 per round)
- Takes 6-8 rounds
- May end with ESCALATE or ACCEPTED

**WALK_AWAY Scenario**:
- Vendor starts at $130
- Refuses to budge
- Accordo walks away after 2-3 rounds
- Final status: WALKED_AWAY (red badge)

---

## First Conversation Test

Test the natural language conversation mode:

### 1. Create Conversation Deal

1. Go to `/chatbot/deals/new`
2. Set **Mode**: "CONVERSATION"
3. Create deal

### 2. Start Chatting

The page will redirect to `/chatbot/conversation/:dealId`

1. Wait for greeting message to appear automatically
2. Type a vendor message: "Hi! I can offer $95 per unit with Net 45 payment terms."
3. Press Send
4. Wait for Accordo's response (2-3 seconds)

### 3. View Decision Breakdown

1. After Accordo replies, click "Show Decision" button
2. Review the ExplainDrawer showing:
   - Vendor's offer details
   - Utility breakdown (price + terms)
   - Decision reasoning
   - Counter offer (if applicable)

### 4. Continue Negotiating

Continue the conversation:
- Try different prices: "$100", "$92"
- Change payment terms: "Net 30", "Net 60"
- Test refusals: "No, I can't share that right now"
- Ask questions: "What are you looking for?"

---

## Verification Checklist

Check that everything works:

### Backend Health

- [ ] Backend running on port 8000
- [ ] Database connected (no errors in logs)
- [ ] Ollama responding (test: `curl http://localhost:11434/api/tags`)
- [ ] Can access `/api/health` endpoint

```bash
curl http://localhost:8000/api/health
# Should return: {"status":"ok"}
```

### Frontend Access

- [ ] Frontend loads at `http://localhost:3000`
- [ ] Can create account
- [ ] Can login successfully
- [ ] Dashboard loads with sidebar

### Chatbot Features

- [ ] Can create new deal (both modes)
- [ ] INSIGHTS mode: scenario chips work
- [ ] CONVERSATION mode: greeting appears automatically
- [ ] Messages send and receive replies
- [ ] Decision badges display correctly
- [ ] ExplainDrawer opens and shows data
- [ ] Deal status updates (NEGOTIATING → ACCEPTED/etc.)

### Database Verification

```bash
# Connect to PostgreSQL
psql -U your_user -d accordo

# Check tables exist
\dt chatbot*

# Should show:
# chatbot_deals
# chatbot_messages
# chatbot_templates
# chatbot_template_parameters

# Check a deal was created
SELECT id, title, status, mode FROM chatbot_deals LIMIT 5;
```

---

## Quick Troubleshooting

### Backend won't start

**Error**: `ECONNREFUSED` on database
```bash
# Check PostgreSQL is running
brew services list | grep postgresql
# Or on Linux:
sudo systemctl status postgresql
```

**Error**: Port 8000 already in use
```bash
# Find and kill process
lsof -ti:8000 | xargs kill -9
```

### Frontend won't start

**Error**: Port 3000 in use
```bash
# Change port in .env.local
VITE_DEV_PORT=3001
```

**Error**: Cannot connect to backend
- Check `VITE_BACKEND_URL` in `.env.local`
- Ensure backend is running
- Check CORS settings (should allow localhost:3000)

### Ollama Issues

**Error**: Cannot connect to Ollama
```bash
# Start Ollama
ollama serve

# Verify it's running
curl http://localhost:11434/api/tags
```

**Error**: Model not found
```bash
# Pull the model
ollama pull llama3.1

# Verify it's available
ollama list
```

### LLM Replies Too Slow

- Normal: 2-4 seconds
- Slow (>10s): Check Ollama logs, may need GPU acceleration
- Timeout (>30s): Increase timeout in `chatbotLlamaClient.ts`

---

## Next Steps

Now that you're up and running:

1. **Explore Features**: Try archiving, deleting, and restoring deals
2. **Read Architecture**: See [ARCHITECTURE.md](./ARCHITECTURE.md) for system overview
3. **API Reference**: Check [API_ENDPOINTS.md](./API_ENDPOINTS.md) for all endpoints
4. **Customize**: Modify negotiation config in backend `config.ts`
5. **Test Scenarios**: Create different deal types and vendor behaviors

---

## Getting Help

If you encounter issues:

1. Check [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for common problems
2. Review backend logs: `tail -f Accordo-ai-backend/logs/app.log`
3. Check browser console for frontend errors
4. Verify all environment variables are set correctly

---

## API Quick Reference

**Create Deal**:
```bash
curl -X POST http://localhost:8000/api/chatbot/deals \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"Test Deal","mode":"CONVERSATION"}'
```

**Send Message**:
```bash
curl -X POST http://localhost:8000/api/chatbot/deals/DEAL_ID/messages \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content":"I can offer $95 with Net 30","role":"VENDOR"}'
```

**List Deals**:
```bash
curl http://localhost:8000/api/chatbot/deals \
  -H "Authorization: Bearer YOUR_TOKEN"
```

---

## Summary

You've now:

- Installed and configured the backend and frontend
- Started Ollama LLM service
- Created your first negotiation deal
- Tested demo scenarios and conversation mode
- Verified all components are working

**Total Setup Time**: ~5-10 minutes

Ready to dive deeper? Check out the complete documentation in this `/docs` folder.

---

**See Also**:
- [INSTALLATION.md](./INSTALLATION.md) - Detailed installation guide
- [ARCHITECTURE.md](./ARCHITECTURE.md) - System architecture
- [API_ENDPOINTS.md](./API_ENDPOINTS.md) - Complete API reference
- [PHASE1_DEMO_FUNCTIONALITY.md](./PHASE1_DEMO_FUNCTIONALITY.md) - Demo mode guide
- [PHASE2_CONVERSATION_SYSTEM.md](./PHASE2_CONVERSATION_SYSTEM.md) - Conversation mode guide
