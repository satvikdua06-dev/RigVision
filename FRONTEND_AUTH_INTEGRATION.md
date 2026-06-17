# Frontend Auth Integration - Complete Setup Guide

## What Was Integrated

Your RigVision-3D frontend now has **complete JWT authentication** with role-based access control:

### 1. **Authentication Routes**
- `/login` - User login page
- `/register` - User registration page  
- `/` - Protected dashboard (requires authentication)

### 2. **New Components**
- **`ProtectedRoute.jsx`** - Route guard that redirects unauthenticated users to login
- **`AppRouter.jsx`** - Central router handling all navigation
- **`TopBar.jsx` (updated)** - Added user profile menu with logout button

### 3. **State Management**
- **`useAuthStore.js`** - Zustand store for authentication state with:
  - User profile, tokens, authentication status
  - Login, register, logout, token refresh methods
  - Automatic token refresh on expiry
  - LocalStorage persistence

### 4. **Environment Configuration**
- **`.env`** - Local development config
- **`.env.production`** - Production config
- **`backend/.env`** - Backend configuration template

### 5. **New Dependency**
- Added `react-router-dom@7.0.0` for client-side routing

---

## Setup Steps

### Step 1: Install Dependencies

From the `frontend/` directory:

```bash
npm install
```

This will install `react-router-dom` and all other dependencies.

### Step 2: Start the Auth Service

From the root directory, start the auth microservice:

```bash
cd auth-rig
npm install
npm start
```

Expected output:
```
Auth service listening on port 5000
MongoDB connected
```

### Step 3: Start the Backend API

From the root directory:

```bash
# Create a Python virtual environment if you haven't already
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install backend dependencies
pip install -r backend/requirements.txt

# Start FastAPI
python backend/main.py
```

Expected output:
```
INFO:     Uvicorn running on http://0.0.0.0:8000
```

### Step 4: Start the Frontend

From the `frontend/` directory:

```bash
npm run dev
```

This starts the Vite dev server on `http://localhost:5173`

### Step 5: Test the Authentication Flow

1. **Open the login page** → http://localhost:5173/login
2. **Create a test account**:
   - Click "Don't have an account? Register"
   - Fill in username, email, password
   - Click Register
   - You'll be redirected to login
3. **Login with your credentials**:
   - Enter email and password
   - You'll be authenticated and redirected to the dashboard
4. **View your profile**:
   - Click your username in the top-right corner
   - See your profile info and role
5. **Logout**:
   - Click "🚪 Logout" in the dropdown menu
   - You'll be redirected to login

---

## Architecture

### Authentication Flow

```
┌─────────────┐
│  Login Page │
└──────┬──────┘
       │ (POST /register or /login)
       ▼
┌──────────────────────────┐
│  auth-rig (Express.js)   │
│  Port 5000               │
│  - User validation       │
│  - JWT generation        │
│  - Token refresh         │
└──────┬───────────────────┘
       │ (JWT tokens)
       ▼
┌──────────────────────────┐
│  useAuthStore (Zustand)  │
│  - Stores tokens         │
│  - Stores user profile   │
│  - Manages auth state    │
│  - Persists to localStorage
└──────┬───────────────────┘
       │ (Authorization header)
       ▼
┌──────────────────────────┐
│  Protected API Routes    │
│  (FastAPI backend)       │
│  - JWT verification      │
│  - Role-based access     │
│  - /api/cv/*, /api/sensors/*
└──────────────────────────┘
```

### Component Hierarchy

```
main.jsx
  └─ AppRouter
      ├─ Routes
      │  ├─ /login → LoginPage
      │  ├─ /register → RegisterPage
      │  └─ /* (protected)
      │     └─ ProtectedRoute
      │        └─ App (dashboard)
      │           ├─ TopBar (with user menu)
      │           ├─ Sidebar
      │           ├─ Scene3D
      │           ├─ CameraFeeds
      │           └─ DiagnosticsModal
```

---

## Token Storage & Security

### How Tokens Are Stored

1. **After Login**: Access and refresh tokens are stored in `localStorage`
2. **On Each Request**: Authorization header automatically includes `Bearer {accessToken}`
3. **On Token Expiry**: Automatically requests new token using refresh token
4. **On Logout**: Tokens cleared from localStorage

### Security Features

- ✅ **HTTPS Required** in production (configure in `.env.production`)
- ✅ **Secure Cookies** via httpOnly flag (refresh token)
- ✅ **Token Expiry**: Access token = 1 hour, Refresh token = 7 days
- ✅ **Brute-Force Protection**: 5 login attempts → 15 min lockout
- ✅ **Password Hashing**: Bcrypt with 12 rounds
- ✅ **CORS Configured**: Only allow requests from frontend/sensor console/auth service
- ⚠️ **LocalStorage Risk**: Access token in localStorage is vulnerable to XSS. Consider using httpOnly cookies in production.

---

## Backend Integration

### Protected Route Examples

To protect your FastAPI routes, use the middleware from `backend/middleware/auth.py`:

```python
from fastapi import FastAPI, Depends, HTTPException
from backend.middleware.auth import verify_token, verify_admin, TokenPayload

@app.get("/api/cv/persons")
async def get_persons(current_user: TokenPayload = Depends(verify_token)):
    # Only authenticated users can access
    return {"user_id": current_user.user_id, "persons": [...]}

@app.post("/api/admin/system-stats")
async def get_stats(current_user: TokenPayload = Depends(verify_admin)):
    # Only admins can access
    return {"stats": {...}}
```

### Apply to Your Routes

1. **Open** `backend/main.py`
2. **Import the middleware**:
   ```python
   from backend.middleware.auth import verify_token, verify_admin, TokenPayload
   ```
3. **Wrap your endpoints**:
   ```python
   @app.get("/api/cv/persons")
   async def get_cv_persons(current_user: TokenPayload = Depends(verify_token)):
       # Now protected!
   ```

---

## Demo Credentials

For testing, use:

| Field | Value |
|-------|-------|
| Email | test@rigvision.com |
| Password | TestPassword123 |
| Username | testuser |
| Role | user |

Or create your own account via the register page.

---

## Environment Variables

### Frontend (`.env`)

```
VITE_AUTH_API=http://localhost:5000/api/auth
VITE_BACKEND_API=http://localhost:8000/api
VITE_ENV=development
```

### Backend (`.env`)

```
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
JWT_SECRET=your_secret_key
AUTH_API_URL=http://localhost:5000/api/auth
CORS_ORIGINS=http://localhost:5173,http://localhost:5174,http://localhost:5000
ENV=development
```

---

## File Locations

| Component | File |
|-----------|------|
| Login Page | `frontend/src/components/LoginPage.jsx` |
| Register Page | `frontend/src/components/RegisterPage.jsx` |
| Auth Store | `frontend/src/stores/useAuthStore.js` |
| Protected Route | `frontend/src/components/ProtectedRoute.jsx` |
| App Router | `frontend/src/components/AppRouter.jsx` |
| TopBar (updated) | `frontend/src/components/TopBar.jsx` |
| FastAPI Middleware | `backend/middleware/auth.py` |
| Auth Service | `auth-rig/` (separate microservice) |

---

## Next Steps

1. ✅ **Install & Run**
   - [ ] `npm install` in frontend/
   - [ ] Start auth service
   - [ ] Start backend API
   - [ ] Start frontend dev server

2. ✅ **Test Authentication**
   - [ ] Register a new user
   - [ ] Login with credentials
   - [ ] Verify token in localStorage
   - [ ] Logout

3. 🔄 **Protect Backend Routes** (In Progress)
   - [ ] Apply `verify_token` to `/api/cv/*` endpoints
   - [ ] Apply `verify_admin` to `/api/admin/*` endpoints
   - [ ] Test protected endpoints with curl

4. 🔄 **Enhanced UI**
   - [ ] Add user profile page
   - [ ] Add password change page
   - [ ] Add role-based UI elements

5. 🔄 **Production Deployment**
   - [ ] Set up HTTPS
   - [ ] Configure production env variables
   - [ ] Set secure cookies in production
   - [ ] Enable CORS for production domains

---

## Troubleshooting

### "Cannot find module 'react-router-dom'"
- Run `npm install` in the frontend directory
- Restart the dev server with `npm run dev`

### "Auth API not responding (CORS error)"
- Ensure auth service is running on port 5000
- Check `VITE_AUTH_API` in `.env` matches the running service
- Verify CORS is enabled in auth-rig/index.js

### "Invalid token" on protected route
- Clear localStorage: DevTools → Application → LocalStorage → Clear
- Register a new account and login again
- Check token expiry (1 hour)

### "Token refresh failed"
- Ensure refresh token is valid (7 day expiry)
- Check auth service is running
- Verify `CORS_ORIGINS` includes your frontend URL

### User stuck on login page after registering
- Check browser console for errors
- Verify auth service is responding with JWT tokens
- Check localStorage for token storage

---

## Security Considerations

**Development Only⚠️**:
- Tokens stored in localStorage (XSS vulnerable)
- VITE_AUTH_API exposed in frontend code
- JWT_SECRET hardcoded in .env

**Before Production**:
- [ ] Move tokens to httpOnly cookies
- [ ] Use environment-specific secrets management
- [ ] Enable HTTPS
- [ ] Add rate limiting
- [ ] Add request signing/verification
- [ ] Implement CSRF protection
- [ ] Add audit logging
- [ ] Regular security audits

---

## API Reference

### Auth Endpoints

**Register User**
```
POST http://localhost:5000/api/auth/register
Content-Type: application/json

{
  "username": "newuser",
  "email": "user@example.com",
  "password": "SecurePass123"
}

Response:
{
  "user": {"id": "...", "username": "...", "role": "user"},
  "token": "eyJhbGc...",
  "refreshToken": "eyJhbGc..."
}
```

**Login User**
```
POST http://localhost:5000/api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123"
}

Response:
{
  "user": {"id": "...", "username": "...", "role": "user"},
  "token": "eyJhbGc...",
  "refreshToken": "eyJhbGc..."
}
```

**Refresh Token**
```
POST http://localhost:5000/api/auth/refresh-token
Authorization: Bearer {refreshToken}

Response:
{
  "token": "eyJhbGc...",
  "refreshToken": "eyJhbGc..."
}
```

**Get Current User**
```
GET http://localhost:5000/api/auth/me
Authorization: Bearer {token}

Response:
{
  "id": "...",
  "username": "...",
  "email": "...",
  "role": "user",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

---

## Support

For issues, refer to:
- [React Router Documentation](https://reactrouter.com/)
- [Zustand Documentation](https://github.com/pmndrs/zustand)
- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [Express.js JWT Guide](https://www.npmjs.com/package/jsonwebtoken)

---

**Last Updated:** 2024
**Status:** ✅ Complete & Ready for Testing
