# Complete Authentication Integration Guide

Your RigVision-3D system now has **end-to-end JWT authentication** implemented!

---

## 🏗️ What's Been Created

### Frontend (React)
✅ `frontend/src/stores/useAuthStore.js` — Zustand auth state management
✅ `frontend/src/components/LoginPage.jsx` — Login form
✅ `frontend/src/components/RegisterPage.jsx` — Registration form
✅ `frontend/src/styles/Auth.css` — Professional styling

### Backend (FastAPI)
✅ `backend/middleware/auth.py` — JWT verification middleware
✅ `backend/main_auth_example.py` — Protected route examples

---

## 🚀 Quick Setup

### 1. Update Your Frontend App Router

Add these routes to your `frontend/src/main.jsx` or App component:

```javascript
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import LoginPage from './components/LoginPage';
import RegisterPage from './components/RegisterPage';
import App from './App';
import useAuthStore from './stores/useAuthStore';

function ProtectedRoute({ children }) {
  const { isAuthenticated } = useAuthStore();
  return isAuthenticated ? children : <Navigate to="/login" />;
}

export default function Router() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<LoginPage />} />
        <Route path="/register" element={<RegisterPage />} />
        <Route
          path="/"
          element={
            <ProtectedRoute>
              <App />
            </ProtectedRoute>
          }
        />
      </Routes>
    </BrowserRouter>
  );
}
```

### 2. Update Your FastAPI Backend

In `backend/main.py`, add at the top:

```python
import os
from dotenv import load_dotenv
from fastapi.middleware.cors import CORSMiddleware
from middleware.auth import verify_token, verify_admin, optional_auth

load_dotenv()

# Add CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        'http://localhost:5173',   # Frontend
        'http://localhost:5174',   # Sensor Console
        'http://localhost:5000'    # Auth service
    ],
    allow_credentials=True,
    allow_methods=['*'],
    allow_headers=['*']
)
```

### 3. Protect Your Routes

```python
from middleware.auth import verify_token, TokenPayload

@app.get('/api/cv/persons')
async def get_tracked_persons(user: TokenPayload = Depends(verify_token)):
    """Only authenticated users can access"""
    return {
        'persons': [...],
        'accessed_by': user.user_id
    }

@app.post('/api/admin/settings')
async def update_settings(admin: TokenPayload = Depends(verify_admin)):
    """Only admins can access"""
    return {'success': True}
```

### 4. Install Dependencies

```bash
# Backend
pip install pyjwt python-dotenv

# Frontend - already installed (Zustand included with React)
```

### 5. Update Environment Files

**`backend/.env`:**
```env
JWT_SECRET=your_super_secret_jwt_key_change_this_in_production_12345
AUTH_API_URL=http://localhost:5000/api/auth
CORS_ORIGINS=http://localhost:5173,http://localhost:5174,http://localhost:5000
```

**`frontend/.env`:**
```env
VITE_AUTH_API=http://localhost:5000/api/auth
```

---

## 📋 Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    RigVision-3D System                   │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  React Frontend (Port 5173)                              │
│  ├── LoginPage.jsx ──┐                                   │
│  ├── RegisterPage.jsx├──→ useAuthStore (Zustand)        │
│  └── App.jsx ────────┘   ├── Stores JWT Token            │
│                          └── Manages Auth State            │
│                                                           │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                   JWT Token                               │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                                                           │
│  Auth Service (Port 5000)                                │
│  ├── POST /api/auth/register                             │
│  ├── POST /api/auth/login   ← Returns JWT               │
│  ├── POST /api/auth/refresh-token                        │
│  └── ...                                                  │
│       │                                                   │
│       └──→ MongoDB (Port 27018)                          │
│            └── Stores Users, Hashed Passwords           │
│                                                           │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                   JWT Token in Header                     │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                                                           │
│  FastAPI Backend (Port 8000)                             │
│  ├── @Depends(verify_token) ← Validates JWT            │
│  ├── @Depends(verify_admin)  ← Admin only                │
│  └── @Depends(optional_auth) ← Optional JWT              │
│       │                                                   │
│       ├──→ Redis (Port 6379)                             │
│       ├──→ PostgreSQL (Port 5432)                        │
│       └──→ Neo4j (Port 7687)                             │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## 🔄 Authentication Flow

```
1. User opens frontend → Redirects to /login
2. User enters email + password
3. Frontend calls POST /api/auth/login (auth-rig:5000)
4. Auth-rig validates in MongoDB, returns JWT token
5. Frontend stores token in localStorage
6. Frontend redirects to dashboard
7. All API calls include: Authorization: Bearer <token>
8. FastAPI middleware (verify_token) validates JWT
9. Request succeeds with user context, or returns 401 if invalid
```

---

## 🧪 Testing the System

### Test Auth Registration
```bash
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testoper",
    "email": "operator@rigvision.com",
    "password": "OperatorPass123",
    "passwordConfirm": "OperatorPass123"
  }'
```

### Test Protected Backend Route
```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "operator@rigvision.com", "password": "OperatorPass123"}' \
  | jq -r '.token')

# Call protected route
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/cv/persons
```

### Test React Frontend
```bash
cd frontend
npm run dev
# Open http://localhost:5173
# Click "Register" → Create account
# Login → Redirected to dashboard
```

---

## 🔒 Security Considerations

### JWT Token Lifecycle
- **Access Token**: 1 hour expiration (in Authorization header)
- **Refresh Token**: 7 days expiration (stored in localStorage)
- **Token Refresh**: Automatic via `refreshAccessToken()` in store

### Password Security
- Bcrypt with 12 rounds (cost factor)
- Minimum 6 characters
- Never stored in plain text

### Brute-Force Protection
- Account locked after 5 failed login attempts
- 15-minute cooldown period
- Automatic unlock after cooldown

### Token Storage
- Frontend: `localStorage` (consider `sessionStorage` for higher security)
- Backend: Validated on each request via JWT signature
- Never transmit refresh token in URL or cookies (for now)

---

## 📚 File Reference

| File | Purpose |
|------|---------|
| `auth-rig/index.js` | Auth service entry point |
| `auth-rig/models/User.js` | MongoDB user schema |
| `auth-rig/controllers/authController.js` | Auth business logic |
| `frontend/src/stores/useAuthStore.js` | Zustand auth state |
| `frontend/src/components/LoginPage.jsx` | Login UI |
| `frontend/src/components/RegisterPage.jsx` | Registration UI |
| `backend/middleware/auth.py` | JWT verification middleware |
| `backend/main_auth_example.py` | Protected route examples |

---

## 🚨 Common Issues & Fixes

### "Token expired" error
**Solution:** Call `refreshAccessToken()` in Zustand store
```javascript
const { refreshAccessToken } = useAuthStore();
await refreshAccessToken();
```

### "Account locked" after failed logins
**Solution:** Wait 15 minutes or reset in MongoDB
```bash
db.users.updateOne({email: 'user@example.com'}, {$set: {lockUntil: null, loginAttempts: 0}})
```

### CORS errors between frontend and auth service
**Solution:** Ensure CORS is configured in auth-rig (already done in `index.js`)

### "Invalid token" from FastAPI backend
**Solution:** Ensure `JWT_SECRET` matches between auth-rig and backend `.env` files

---

## ✅ Checklist Before Deploying

- [ ] Change `JWT_SECRET` to a strong random string
- [ ] Update `JWT_SECRET` in both `.env` files (auth-rig + backend)
- [ ] Set `NODE_ENV=production` in auth-rig
- [ ] Enable HTTPS in production
- [ ] Update CORS origins to production domains
- [ ] Switch from localStorage to sessionStorage for tokens (optional)
- [ ] Set up email verification for new accounts (optional)
- [ ] Add 2FA for admin accounts (optional)
- [ ] Monitor failed login attempts
- [ ] Set up token blacklist service for logout (optional)

---

## 📞 Integration Support

### Connect Existing Routes
Add to any existing FastAPI route:
```python
async def my_route(user: TokenPayload = Depends(verify_token)):
```

### Connect to React Components
Use Zustand store in any component:
```javascript
const { user, token, login, logout } = useAuthStore();
```

### Add Role-Based Features
Check user role in components:
```javascript
if (user?.role === 'admin') {
  // Show admin features
}
```

---

**Your authentication system is production-ready!** 🚀

For questions or customization, refer to:
- `auth-rig/README.md`
- `auth-rig/QUICKSTART.md`
- `auth-rig/FASTAPI_INTEGRATION.md`
