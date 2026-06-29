# Environment Variables Setup Guide

## Required Environment Variables

Create a `.env.local` file in the root of the frontend directory with the following variables:

```env
# Backend API URL
# The frontend automatically appends /api when calling the backend
# You can provide either the root host (http://localhost:8000) or the full API origin (http://localhost:8000/api)
VITE_BACKEND_URL=http://localhost:8000

# Frontend URL
# Used for generating contract links and other frontend URLs
# In production, this should be your actual frontend domain
VITE_FRONTEND_URL=http://localhost:3000

# Asset/Upload URL
# Base URL for serving uploaded files (images, documents, etc.)
# This should point to your backend's upload directory or CDN
VITE_ASSEST_URL=http://localhost:8000

# Development Server Configuration
# Host for the Vite dev server (0.0.0.0 allows access from network)
VITE_DEV_HOST=0.0.0.0

# Port for the Vite dev server
VITE_DEV_PORT=3000
```

## Environment Variables Explained

### VITE_BACKEND_URL
- **Purpose**: Backend API base URL
- **Default**: `http://localhost:8000`
- **Usage**: Used in `src/api/index.js` and `src/services/chat.service.js`
- **Note**: The frontend automatically appends `/api` to this URL

### VITE_FRONTEND_URL
- **Purpose**: Frontend application URL
- **Default**: `http://localhost:3000`
- **Usage**: Used for generating contract links and sharing URLs
- **Files**: 
  - `src/components/Requisition/VendorDetails.jsx`
  - `src/components/vendor/Contracts.jsx`

### VITE_ASSEST_URL
- **Purpose**: Base URL for uploaded assets (images, documents)
- **Default**: `http://localhost:8000`
- **Usage**: Used throughout the app for displaying uploaded files
- **Files**: Multiple components including:
  - `src/components/VendorForm/*`
  - `src/components/vendor/*`
  - `src/components/user/*`
  - `src/components/settingForm/*`
  - `src/components/Requisition/*`

### VITE_DEV_HOST
- **Purpose**: Development server host
- **Default**: `0.0.0.0`
- **Usage**: Used in `vite.config.js` for dev server configuration
- **Note**: `0.0.0.0` allows access from network, use `localhost` for local-only access

### VITE_DEV_PORT
- **Purpose**: Development server port
- **Default**: `3000` (as configured in vite.config.js, but README says 5173)
- **Usage**: Used in `vite.config.js` for dev server configuration

## Setup Instructions

1. **Create `.env.local` file**:
   ```bash
   cd /Users/vidhyashah/workspace/AccordoFrontend
   touch .env.local
   ```

2. **Add the environment variables** (copy from above)

3. **Update values** according to your setup:
   - For production: Update all URLs to production domains
   - For staging: Update to staging URLs
   - For local: Use the defaults shown above

4. **Restart the dev server** after making changes:
   ```bash
   npm run dev
   ```

## Production Setup

For production, create a `.env.production` file with production values:

```env
VITE_BACKEND_URL=https://api.yourdomain.com
VITE_FRONTEND_URL=https://yourdomain.com
VITE_ASSEST_URL=https://api.yourdomain.com
VITE_DEV_HOST=0.0.0.0
VITE_DEV_PORT=3000
```

## Important Notes

- `.env.local` is gitignored (see `.gitignore`)
- Never commit `.env.local` with sensitive values
- Use `.env.example` as a template (if you create one)
- All Vite environment variables must be prefixed with `VITE_`
- Changes to `.env.local` require a dev server restart

