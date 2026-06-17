# Quick Start: Running RigVision-3D with Authentication

## TL;DR - Get Running in 5 Minutes

### Terminal 1: Start MongoDB & Infrastructure
```bash
docker-compose up redis postgres neo4j kafka chromadb
```

### Terminal 2: Start Auth Service
```bash
cd auth-rig
npm install
npm start
```

✅ Auth service running on http://localhost:5000

### Terminal 3: Start Backend API
```bash
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r backend/requirements.txt
python backend/main.py
```

✅ Backend running on http://localhost:8000

### Terminal 4: Start Frontend
```bash
cd frontend
npm install
npm run dev
```

✅ Dashboard available at http://localhost:5173

---

## First Login

1. Go to http://localhost:5173 (auto-redirects to login)
2. Click "Don't have an account? Register"
3. Fill in: username, email, password
4. Click Register
5. Login with your credentials
6. You're in! 🎉

### Demo Account
```
Email: test@rigvision.com
Password: TestPassword123
```

---

## Project Structure After Integration

```
RigVision/
├── auth-rig/              ← Auth microservice (Express + MongoDB)
├── backend/
│   ├── main.py           ← FastAPI app
│   ├── middleware/
│   │   └── auth.py       ← JWT verification
│   ├── .env              ← Backend config
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── AppRouter.jsx      ← Routes (NEW)
│   │   │   ├── ProtectedRoute.jsx ← Auth guard (NEW)
│   │   │   ├── LoginPage.jsx      ← Login form
│   │   │   ├── RegisterPage.jsx   ← Signup form
│   │   │   └── TopBar.jsx         ← Updated with user menu
│   │   ├── stores/
│   │   │   └── useAuthStore.js    ← Auth state
│   │   └── main.jsx               ← Updated entry point
│   ├── .env               ← Frontend config (NEW)
│   ├── .env.production    ← Prod config (NEW)
│   └── package.json       ← Added react-router-dom
└── FRONTEND_AUTH_INTEGRATION.md ← Full guide
```

---

## Common Commands

| Task | Command |
|------|---------|
| Start all infra | `docker-compose up -d` |
| Stop all infra | `docker-compose down` |
| Restart auth service | `cd auth-rig && npm start` |
| Clear auth cache | `redis-cli FLUSHALL` |
| View auth logs | `docker logs auth-rig` |
| Connect to MongoDB | `mongosh mongodb://admin:strong_password@localhost:27018/rigvision_auth?authSource=admin` |
| Create demo user via curl | `curl -X POST http://localhost:5000/api/auth/register -H "Content-Type: application/json" -d @register.json` |

---

## Verify Everything is Connected

### Check Auth Service
```bash
curl http://localhost:5000/health
# Should return: {"status":"ok"}
```

### Check Backend API
```bash
curl http://localhost:8000/health
# Should return: 200 OK
```

### Check Frontend
```bash
# Just visit http://localhost:5173
# Should redirect to login page
```

---

## What Happens When You Login

1. **Frontend** - LoginPage collects email/password
2. **Auth Service** - Validates credentials, returns JWT tokens
3. **Zustand Store** - Stores tokens in localStorage + state
4. **Protected Route** - Checks `isAuthenticated` flag
5. **Dashboard** - Renders App component with all your 3D models
6. **API Calls** - All requests include `Authorization: Bearer {token}` header
7. **Backend** - FastAPI middleware verifies token, allows/denies access

---

## Next: Protect Your Backend Routes

See the backend/middleware/auth.py file for examples. To protect an endpoint:

```python
from fastapi import Depends
from backend.middleware.auth import verify_token, TokenPayload

@app.get("/api/cv/persons")
async def get_cv_persons(current_user: TokenPayload = Depends(verify_token)):
    # This endpoint now requires valid JWT token
    return {"persons": [...]}
```

---

## Troubleshooting in 30 Seconds

| Problem | Solution |
|---------|----------|
| Blank white screen | Check browser console (F12), ensure auth service is running |
| "Auth API not responding" | Verify auth-rig is on port 5000, check CORS in .env |
| Token expired after login | Tokens are auto-refreshed, check refresh token is valid |
| Can't register | Check MongoDB is running, no duplicate email |
| Dashboard loads but no data | Ensure backend API is running, check /api/cv/* endpoints |

---

## Environment Variables

### Frontend `.env`
```
VITE_AUTH_API=http://localhost:5000/api/auth
VITE_BACKEND_API=http://localhost:8000/api
VITE_ENV=development
```

### Backend `.env`
```
DATABASE_URL=postgresql://rigvision:rigvision_dev_password@localhost:5432/rigvision
REDIS_URL=redis://localhost:6379
JWT_SECRET=your_super_secret_jwt_key
AUTH_API_URL=http://localhost:5000/api/auth
CORS_ORIGINS=http://localhost:5173,http://localhost:5174,http://localhost:5000
ENV=development
```

---

## Files Changed/Created

```
NEW:
  frontend/src/components/ProtectedRoute.jsx
  frontend/src/components/AppRouter.jsx
  frontend/.env
  frontend/.env.production
  backend/.env
  FRONTEND_AUTH_INTEGRATION.md
  FRONTEND_AUTH_QUICKSTART.md (this file)

UPDATED:
  frontend/src/main.jsx
  frontend/src/components/TopBar.jsx
  frontend/package.json

EXISTING (ready to use):
  frontend/src/components/LoginPage.jsx
  frontend/src/components/RegisterPage.jsx
  frontend/src/stores/useAuthStore.js
  backend/middleware/auth.py
```

---

## Testing Checklist

- [ ] Docker containers running (`docker ps`)
- [ ] Auth service running on :5000
- [ ] Backend API running on :8000
- [ ] Frontend dev server running on :5173
- [ ] Can access http://localhost:5173 (redirects to login)
- [ ] Can register new user
- [ ] Can login with credentials
- [ ] Can see dashboard after login
- [ ] Can see user profile in top-right corner
- [ ] Can logout and return to login page
- [ ] Token stored in localStorage (`F12 → Application → LocalStorage`)

---

✅ **Integration Complete!**

You're ready to:
- Build protected routes
- Implement role-based access
- Add user management
- Scale to production

👉 Read [FRONTEND_AUTH_INTEGRATION.md](./FRONTEND_AUTH_INTEGRATION.md) for full details.
