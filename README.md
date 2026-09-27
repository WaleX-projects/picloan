# PickLoan - AI-Powered Loan Document Processing System

A full-stack mobile and web application for loan officers to capture, process, and analyze loan application documents using AI-powered data extraction. Built with React Native/Expo and FastAPI.

## 📋 Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Authentication](#authentication)
- [API Reference](#api-reference)
- [Development](#development)
- [Deployment](#deployment)
- [Database Schema](#database-schema)
- [Troubleshooting](#troubleshooting)

## 🎯 Overview

PickLoan streamlines the loan application process by enabling loan officers to:

1. **Capture** loan application documents via mobile camera
2. **Upload** documents to secure cloud storage (Supabase)
3. **Extract** applicant information and loan details using AI (Nova API)
4. **Review** extracted data with status tracking
5. **Synchronize** data across devices and maintain offline access

The system operates on an **offline-first architecture** where applications are stored locally on the device and synchronized with the backend when connectivity is available.

### Problem Solved

Manual loan document processing is time-consuming and error-prone. PickLoan automates document scanning, optical character recognition (OCR), and data extraction, reducing processing time and human error.

## ✨ Key Features

### Mobile/Web Features

- **📸 Document Capture**: Capture loan application documents directly from device camera
- **📱 Offline-First**: Create and view applications without internet connection
- **🔄 Automatic Sync**: Upload documents to server when connection available
- **🤖 AI Extraction**: Automatic extraction of applicant name, loan amount, and other key details
- **📊 Application Status Tracking**: Monitor applications through processing pipeline
  - `LOCAL` - Created on device, not yet uploaded
  - `SYNC_PENDING` - Ready to upload
  - `PROCESSING` - Server processing the document
  - `NEEDS_REVIEW` - Awaiting officer review
  - `CONFIRMED` - Officer confirmed the extracted data
  - `SYNCED` - Successfully synchronized with server
  - `FAILED` - Upload or processing failed
- **🔍 Search & Filter**: Find applications by ID or applicant name, filter by status
- **👤 OAuth Authentication**: Secure login with Manus OAuth provider
- **🌐 Cross-Platform**: Works on iOS, Android, and web browsers

## 🛠 Tech Stack

### Frontend
- **React Native** 0.81.5 - Cross-platform mobile framework
- **Expo** 54.0.29 - Build and deployment platform
- **Expo Router** 6.0.19 - File-based routing
- **TypeScript** 5.9.3 - Type safety
- **TailwindCSS** + **NativeWind** 4.2.1 - Styling
- **React Query** 5.90.12 - Server state management
- **tRPC** 11.7.2 - Type-safe RPC
- **Expo SQLite** 16.0.10 - Local database
- **Expo Secure Store** - Secure credential storage

### Backend
- **FastAPI** - Python web framework
- **SQLAlchemy** - ORM for database operations
- **PostgreSQL** - Primary database (via Supabase)
- **Supabase** - File storage and authentication
- **Nova API** - AI document analysis and data extraction

### Infrastructure
- **Drizzle ORM** 0.44.7 - Database schema management
- **Express** 4.22.1 - Optional backend server
- **Metro** - React Native bundler
- **esbuild** - Production bundling

## 🏗 Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────┐
│           Mobile App (iOS/Android) + Web            │
│  ┌──────────────────────────────────────────────┐   │
│  │  React Native / Expo                         │   │
│  │  - Capture Screen (Camera)                   │   │
│  │  - Home Screen (Application List)            │   │
│  │  - Detail Screen (Extract Data Review)       │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────┬──────────────────────────────────┘
                   │ HTTP/REST
        ┌──────────▼──────────┐
        │    API Gateway      │
        │  (FastAPI Server)   │
        └──────────┬──────────┘
                   │
        ┌──────────┴─────────────┬──────────┐
        │                        │          │
   ┌────▼────┐          ┌────────▼──┐   ┌──▼──────┐
   │PostgreSQL│          │  Supabase  │   │ Nova AI │
   │Database  │          │  Storage   │   │ Service │
   └──────────┘          └────────────┘   └─────────┘

Local Device Storage:
┌─────────────────────────┐
│  SQLite Database        │
│  - applications table   │
│  - Local image paths    │
│  - Extracted data cache │
└─────────────────────────┘
```

### Data Flow

#### Document Capture & Upload Flow

```
1. User opens app
   ↓
2. Navigate to "Capture" tab
   ↓
3. Take photo of loan document
   ↓
4. Photo saved locally: /scans/scan_<timestamp>.jpg
   ↓
5. Application created in local SQLite with:
   - id: "PL-<timestamp>"
   - status: "LOCAL"
   - localImagePath: <path to saved image>
   ↓
6. User navigates to "Home" tab, sees application in list
   ↓
7. User selects application to upload
   ↓
8. App uploads to backend:
   POST /upload with:
   - sync_id: application ID
   - files: image file
   ↓
9. Backend processes:
   - Stores image in Supabase Storage
   - Creates Loan record in PostgreSQL
   - Calls Nova API for data extraction
   - Stores extracted data as JSON
   ↓
10. Backend returns:
    - loan_id
    - status: "SUCCESS" or "FAILED"
    - extracted_data: { applicant: {...}, loan: {...} }
    - public_url: Supabase URL
    ↓
11. App updates local record:
    - status: "SYNCED"
    - syncImagePath: public_url
    - extractedData: { applicant.full_name, loan.amount_requested, ... }
    ↓
12. User can view extracted information
```

### Authentication Flow

#### Native (iOS/Android)

```
1. User taps Login
   ↓
2. App opens system browser to Manus OAuth portal
   ↓
3. User authenticates and grants permissions
   ↓
4. OAuth redirects to: manus<timestamp>://oauth/callback?code=X&state=Y
   ↓
5. Deep link returns to app
   ↓
6. OAuth callback handler exchanges code:
   GET /api/oauth/mobile?code=X&state=Y
   ↓
7. Backend returns: { app_session_id, user: {...} }
   ↓
8. App stores token in Expo SecureStore:
   - SESSION_TOKEN_KEY: <token>
   - USER_INFO_KEY: <user json>
   ↓
9. On subsequent requests, app:
   - Retrieves token from SecureStore
   - Adds "Authorization: Bearer <token>" header
   ↓
10. App navigates to home screen
```

#### Web

```
1. User taps Login
   ↓
2. App redirects to Manus OAuth portal
   ↓
3. User authenticates
   ↓
4. OAuth redirects to: https://api.example.com/api/oauth/callback?code=X&state=Y
   ↓
5. Backend validates code and sets HttpOnly cookie
   ↓
6. Browser automatically includes cookie in subsequent requests
   ↓
7. App navigates to home screen
```

## 🚀 Getting Started

### Prerequisites

- **Node.js** 18+ and **pnpm** 9.12.0
- **Expo CLI** (install via: `npm install -g expo-cli`)
- **Python** 3.9+ (for backend)
- **PostgreSQL** database
- **Supabase** account (for file storage)
- **Nova API** credentials (for AI extraction)
- **Manus OAuth** credentials (portal URL, server URL, app ID)

### Frontend Setup

1. **Clone the repository**
   ```bash
   git clone 
   cd pickloan
   ```

2. **Install dependencies**
   ```bash
   pnpm install
   ```

3. **Configure environment variables**
   
   Create `.env` file (copy from `.env.example` if available):
   ```env
   EXPO_PUBLIC_OAUTH_PORTAL_URL=https://oauth.manus.example.com
   EXPO_PUBLIC_OAUTH_SERVER_URL=https://oauth-server.manus.example.com
   EXPO_PUBLIC_API_BASE_URL=http://localhost:8000
   EXPO_PUBLIC_APP_ID=your-app-id
   EXPO_PUBLIC_OWNER_OPEN_ID=your-owner-id
   EXPO_PUBLIC_OWNER_NAME="Your Organization"
   ```

4. **Start development server**
   ```bash
   pnpm dev
   ```

   This starts both:
   - Metro bundler on port 8081 (mobile)
   - Backend server (if present)

5. **Run on mobile**
   ```bash
   # iOS (macOS only)
   pnpm ios
   
   # Android
   pnpm android
   
   # Web
   pnpm dev:metro  # Already running from 'pnpm dev'
   # Open: http://localhost:8081
   ```

### Backend Setup

1. **Navigate to backend directory**
   ```bash
   cd ../loan_manager
   ```

2. **Create Python virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Configure environment variables**
   ```env
   DATABASE_URL=postgresql://user:password@localhost/pickloan_db
   SUPABASE_URL=https://your-project.supabase.co
   SUPABASE_SERVICE_KEY=your-service-key
   NOVA_API_KEY=your-nova-api-key
   ```

5. **Run database migrations**
   ```bash
   alembic upgrade head
   ```

6. **Start FastAPI server**
   ```bash
   uvicorn app.main:app --reload --port 8000
   ```

   API will be available at: http://localhost:8000

## 📁 Project Structure

```
pickloan/
├── app/                           # Frontend application
│   ├── (tabs)/                    # Tab-based navigation
│   │   ├── index.tsx              # Home screen (application list)
│   │   ├── capture.tsx            # Camera capture screen
│   │   └── profile.tsx            # User profile screen
│   ├── application/               # Application details
│   │   └── [id].tsx               # Individual application details
│   ├── oauth/                     # Authentication
│   │   └── callback.tsx           # OAuth callback handler
│   ├── _layout.tsx                # Root layout and providers
│   └── processing.tsx             # Processing status screen
├── components/                    # Reusable React components
│   ├── ui/                        # UI component library
│   ├── themed-view.tsx
│   ├── screen-container.tsx
│   └── ...
├── constants/                     # Application constants
│   ├── oauth.ts                   # OAuth configuration
│   ├── theme.ts                   # Theme colors
│   └── const.ts                   # General constants
├── hooks/                         # React hooks
│   ├── use-auth.ts                # Authentication hook
│   ├── use-color-scheme.ts
│   └── use-colors.ts
├── lib/                           # Shared utilities and core logic
│   ├── _core/                     # Core libraries
│   │   ├── api.ts                 # HTTP client with auth
│   │   ├── auth.ts                # Token/credential management
│   │   ├── nativewind-pressable.ts
│   │   └── manus-runtime.ts
│   ├── loan-store.ts              # SQLite database layer
│   ├── server.ts                  # File upload orchestration
│   ├── theme-provider.tsx         # Theme context
│   └── utils.ts                   # Utility functions
├── shared/                        # Shared types and constants
│   ├── types.ts                   # TypeScript type definitions
│   └── const.ts                   # Shared constants
├── drizzle/                       # Database schema and migrations
│   ├── schema.ts                  # Drizzle schema definitions
│   ├── relations.ts               # Relationships
│   ├── migrations/                # Migration files
│   └── meta/                      # Metadata
├── tests/                         # Unit and integration tests
│   ├── auth.logout.test.ts
│   └── loan-store.test.ts
├── scripts/                       # Build and utility scripts
│   ├── generate_qr.mjs            # QR code generation
│   ├── load-env.js
│   └── reset-project.js
├── public/                        # Static assets
├── package.json                   # Dependencies and scripts
├── tsconfig.json                  # TypeScript configuration
├── tailwind.config.js             # Tailwind CSS configuration
├── babel.config.js                # Babel configuration
├── metro.config.js                # Metro bundler configuration
├── drizzle.config.ts              # Drizzle ORM configuration
├── app.config.ts                  # Expo app configuration
└── README.md                      # This file

loan_manager/                      # Backend Python application
├── app/
│   ├── main.py                    # FastAPI entry point and routes
│   ├── models.py                  # SQLAlchemy ORM models
│   ├── database.py                # Database connection
│   ├── services/
│   │   ├── storage_service.py     # Supabase file upload
│   │   └── ai_service.py          # Nova API integration
│   └── __init__.py
├── alembic/                       # Database migrations
│   ├── env.py
│   ├── script.py.mako
│   ├── versions/
│   └── README
├── requirements.txt               # Python dependencies
├── alembic.ini                    # Alembic configuration
├── .env                           # Environment variables
└── .env.example                   # Environment template
```

## 🔐 Authentication

### OAuth Provider Configuration

PickLoan uses **Manus OAuth** for secure authentication. Configure the following environment variables:

```typescript
// constants/oauth.ts
export const OAUTH_PORTAL_URL = process.env.EXPO_PUBLIC_OAUTH_PORTAL_URL;
export const OAUTH_SERVER_URL = process.env.EXPO_PUBLIC_OAUTH_SERVER_URL;
export const APP_ID = process.env.EXPO_PUBLIC_APP_ID;
export const OWNER_OPEN_ID = process.env.EXPO_PUBLIC_OWNER_OPEN_ID;
export const OWNER_NAME = process.env.EXPO_PUBLIC_OWNER_NAME;
```

### Session Management

#### Native Platforms
- Session tokens stored in **Expo SecureStore** (encrypted)
- Tokens automatically included in `Authorization: Bearer <token>` header
- User info cached for offline access

#### Web Platform
- Sessions managed via **HttpOnly cookies**
- Cookies automatically sent with requests (`credentials: "include"`)
- No manual token management needed

### Key Auth Functions

```typescript
// Login
import { startOAuthLogin } from '@/constants/oauth';
await startOAuthLogin();  // Initiates OAuth flow

// Get current user
import { useAuth } from '@/hooks/use-auth';
const { user, isAuthenticated, loading } = useAuth();

// Logout
const { logout } = useAuth();
await logout();

// Manual token management (native only)
import * as Auth from '@/lib/_core/auth';
const token = await Auth.getSessionToken();
await Auth.setSessionToken(newToken);
await Auth.removeSessionToken();
```

## 📡 API Reference

### Core Endpoints

#### Upload Loan Application

```http
POST /upload
Content-Type: multipart/form-data

Parameters:
- sync_id (string): Application ID from mobile app
- files (file[]): One or more image files

Response:
{
  "loan_id": 123,
  "status": "SUCCESS",  // or "FAILED"
  "files": [
    {
      "filename": "loan-PL-12345.jpg",
      "content_type": "image/jpeg",
      "size": 2048000,
      "storage_path": "loans/123/uuid-filename.jpg",
      "public_url": "https://supabase.co/storage/v1/object/public/..."
    }
  ],
  "extracted_data": {
    "documents": [
      {
        "applicant": {
          "full_name": "John Doe",
          "date_of_birth": "1990-01-15"
        },
        "loan": {
          "amount_requested": 50000,
          "purpose": "Business expansion"
        }
      }
    ]
  }
}
```

#### OAuth Mobile Callback

```http
GET /api/oauth/mobile?code=AUTH_CODE&state=STATE_ENCODED

Response:
{
  "app_session_id": "token-string",
  "user": {
    "id": 1,
    "openId": "oauth-user-id",
    "name": "John Officer",
    "email": "john@example.com",
    "loginMethod": "manus-oauth",
    "lastSignedIn": "2024-01-15T10:30:00Z"
  }
}
```

#### Get Current User

```http
GET /api/auth/me
Authorization: Bearer <token>

Response:
{
  "user": {
    "id": 1,
    "openId": "oauth-user-id",
    "name": "John Officer",
    "email": "john@example.com",
    "loginMethod": "manus-oauth",
    "lastSignedIn": "2024-01-15T10:30:00Z"
  }
}
```

#### Logout

```http
POST /api/auth/logout
Authorization: Bearer <token>

Response: 200 OK (session cleared)
```

### API Client Usage

```typescript
import { apiCall } from '@/lib/_core/api';

// Simple GET request
const data = await apiCall<MyType>('/api/data');

// POST with JSON body
const result = await apiCall<ResponseType>('/api/create', {
  method: 'POST',
  body: JSON.stringify({ name: 'value' }),
});

// File upload with FormData
const formData = new FormData();
formData.append('file', imageFile);
formData.append('sync_id', appId);
const response = await apiCall<UploadResponse>('/upload', {
  method: 'POST',
  body: formData,
});
```

**Key Features of `apiCall`:**
- Automatic platform detection (native vs web)
- Bearer token injection for native platforms
- Cookie credential handling for web
- JSON response parsing
- Comprehensive error logging
- Works with FormData for file uploads

## 💻 Development

### Build Scripts

```bash
# Development
pnpm dev              # Run Metro + backend server concurrently
pnpm dev:metro        # Start Metro bundler only
pnpm dev:server       # Start backend server only

# Production
pnpm build            # Build server bundle
pnpm start            # Run production build

# Code Quality
pnpm check            # TypeScript type checking
pnpm lint             # ESLint
pnpm format           # Prettier formatting
pnpm test             # Run Vitest

# Database
pnpm db:push          # Generate and migrate database schema

# Platform-specific
pnpm ios              # Build and run iOS app
pnpm android          # Build and run Android app
pnpm qr               # Generate QR code for app preview
```

### Local Database Development

The app uses **Expo SQLite** for local storage. Key tables:

#### Applications Table

```sql
CREATE TABLE applications (
  id TEXT PRIMARY KEY,
  local_image_path TEXT NOT NULL,
  sync_image_path TEXT,
  extracted_data TEXT,  -- JSON string
  created_at TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'LOCAL',
  updated_at TEXT NOT NULL
);
```

### Working with Local Data

```typescript
import {
  getApplications,           // Get all applications
  getApplicationById,        // Get single application
  insertApplication,         // Create application
  markSynced,                // Mark as synced
  updateExtractedData,       // Store AI results
  saveCapturedImage,         // Move image to permanent storage
} from '@/lib/loan-store';

// List all applications
const apps = getApplications();

// Create application
const app = await createLocalApplication(imageUri);

// Update after sync
markSynced(appId, syncUrl);
updateExtractedData(appId, extractedData, syncUrl);
```

### Testing

```bash
# Run all tests
pnpm test

# Run specific test file
pnpm test auth.logout.test.ts

# Watch mode
pnpm test --watch
```

**Existing tests:**
- `tests/auth.logout.test.ts` - Logout functionality
- `tests/loan-store.test.ts` - Database operations

## 🌐 Deployment

### Frontend Deployment

#### Web
```bash
# Build for web production
pnpm build

# Output: dist/ directory
# Deploy to: Vercel, Netlify, S3, etc.
```

#### Mobile (Expo Managed Hosting)
```bash
# Build iOS
eas build --platform ios

# Build Android
eas build --platform android
```

#### Self-Hosted Backend
```bash
# Build server bundle
pnpm build

# Output: dist/index.js
# Deploy to: AWS, DigitalOcean, Heroku, etc.

# Run production server
NODE_ENV=production node dist/index.js
```

### Backend Deployment

```bash
# Install production dependencies
pip install -r requirements.txt

# Run migrations
alembic upgrade head

# Start Gunicorn server
gunicorn app.main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

### Environment Configuration

**Development:**
```env
NODE_ENV=development
DATABASE_URL=postgresql://localhost/pickloan_dev
SUPABASE_URL=https://dev-project.supabase.co
API_BASE_URL=http://localhost:8000
```

**Production:**
```env
NODE_ENV=production
DATABASE_URL=postgresql://prod-user:password@prod-host/pickloan
SUPABASE_URL=https://prod-project.supabase.co
SUPABASE_SERVICE_KEY=secure-key
NOVA_API_KEY=secure-key
API_BASE_URL=https://api.example.com
OAUTH_PORTAL_URL=https://oauth.example.com
OAUTH_SERVER_URL=https://oauth-server.example.com
```

## 🗄 Database Schema

### Frontend Local Database (SQLite)

**applications** table:
- `id` (TEXT, PRIMARY KEY) - Unique application ID (e.g., "PL-12345")
- `local_image_path` (TEXT) - Path to image file on device
- `sync_image_path` (TEXT, nullable) - Supabase URL after sync
- `extracted_data` (TEXT) - JSON string of AI extraction results
- `created_at` (TEXT) - ISO timestamp
- `updated_at` (TEXT) - ISO timestamp
- `status` (TEXT) - Application status (LOCAL, SYNC_PENDING, PROCESSING, etc.)

### Backend Database (PostgreSQL)

**users** table:
- `id` (INT, PRIMARY KEY) - Auto-incremented ID
- `openId` (VARCHAR) - Unique OAuth identifier
- `name` (TEXT) - User full name
- `email` (VARCHAR) - User email
- `loginMethod` (VARCHAR) - Auth method (e.g., "manus-oauth")
- `role` (ENUM) - User role ("user" or "admin")
- `createdAt` (TIMESTAMP) - Account creation time
- `updatedAt` (TIMESTAMP) - Last update time
- `lastSignedIn` (TIMESTAMP) - Last login time

**loans** table:
- `id` (INT, PRIMARY KEY) - Auto-incremented ID
- `sync_id` (VARCHAR) - Mobile app application ID reference
- `status` (VARCHAR) - Processing status
- `extracted_data` (JSONB) - AI extraction results
- `created_at` (TIMESTAMP) - Record creation time
- `updated_at` (TIMESTAMP) - Last update time
- `documents` (RELATION) - Links to LoanDocument records

**loan_documents** table:
- `id` (INT, PRIMARY KEY) - Auto-incremented ID
- `loan_id` (INT, FOREIGN KEY) - Links to Loan
- `filename` (VARCHAR) - Original file name
- `storage_path` (VARCHAR) - Supabase storage URL
- `content_type` (VARCHAR) - MIME type
- `file_size` (INT) - File size in bytes
- `created_at` (TIMESTAMP) - Record creation time
- `updated_at` (TIMESTAMP) - Last update time

## 🐛 Troubleshooting

### Common Issues

#### 1. "Cannot find module" errors
```bash
# Clear cache and reinstall
rm -rf node_modules
rm pnpm-lock.yaml
pnpm install
```

#### 2. Database migration issues
```bash
# Reset database (development only)
node scripts/reset-project.js

# Verify schema
pnpm db:push
```

#### 3. OAuth configuration not working
- Verify OAuth portal and server URLs in `.env`
- Confirm bundle ID matches OAuth app configuration
- Check that redirect URI is correctly registered

**For native**: Redirect URI should be: `manus<timestamp>://oauth/callback`
**For web**: Redirect URI should be: `https://api.example.com/api/oauth/callback`

#### 4. Images not uploading
- Ensure user has camera and storage permissions
- Check that `EXPO_PUBLIC_API_BASE_URL` is correct
- Verify backend server is running (`http://localhost:8000` in dev)
- Check Supabase credentials in backend `.env`

#### 5. AI extraction not working
- Verify Nova API key is correct
- Check that image is clear and readable
- Review Nova API documentation for supported document types
- Check server logs for extraction errors

#### 6. "Unauthorized" errors on API calls
- Verify OAuth token is stored correctly
- Check token expiration
- For web: verify cookie is being sent (`credentials: "include"`)
- For native: verify token is in SecureStore

### Debugging

#### Enable verbose logging
```typescript
// In lib/_core/api.ts, the apiCall function logs all requests
// Check browser console (web) or device console (mobile)
```

#### Check local database
```typescript
// Access SQLite directly (React Native)
import * as SQLite from 'expo-sqlite';
const db = SQLite.openDatabaseSync("loanapp.db");
const rows = db.getAllSync("SELECT * FROM applications");
console.log(rows);
```

#### Verify backend connectivity
```bash
# Test API endpoint
curl http://localhost:8000/api/auth/me \
  -H "Authorization: Bearer YOUR_TOKEN"

# Test file upload
curl -X POST http://localhost:8000/upload \
  -F "sync_id=PL-12345" \
  -F "files=@/path/to/image.jpg"
```

### Getting Help

- **Frontend Issues**: Check React Native and Expo documentation
- **Backend Issues**: Review FastAPI and SQLAlchemy docs
- **OAuth Issues**: Contact Manus OAuth support
- **AI Extraction**: Check Nova API documentation

## 📚 Additional Resources

- [React Native Documentation](https://reactnative.dev)
- [Expo Documentation](https://docs.expo.dev)
- [FastAPI Documentation](https://fastapi.tiangolo.com)
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org)
- [Supabase Documentation](https://supabase.com/docs)

## 📝 License

[Add your license information here]

## 👥 Contributors

[List contributors or add contribution guidelines]

---

**Last Updated:** September 2024
**Version:** 1.0.0
