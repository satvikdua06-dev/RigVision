# Auth-Rig: Quick Start

Your authentication system is **fully functional and running** on port 5000.

## ✅ What's Built

- **User Registration** — Create accounts with email/username
- **JWT Authentication** — Access tokens (1h) + refresh tokens (7d)
- **Login System** — Email/password authentication  
- **Password Hashing** — Bcrypt with 12 rounds
- **Brute-Force Protection** — Account lockout after 5 failed attempts
- **Role-Based Access** — user, admin, operator roles
- **Profile Management** — Update user info & change password
- **Protected Routes** — Authorization middleware for secure endpoints

---

## 🚀 Running the System

### Start all services (MongoDB, Redis, PostgreSQL, etc.)
```bash
docker compose up -d
```

### Start the auth server
```bash
cd auth-rig
npm run dev
```

Server runs on: `http://localhost:5000`

---

## 🧪 Test the API

### 1. Register a user
```bash
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john_doe",
    "email": "john@example.com",
    "password": "SecurePass123",
    "passwordConfirm": "SecurePass123"
  }'
```

Response:
```json
{
  "success": true,
  "message": "User registered successfully",
  "token": "eyJhbGc...",
  "refreshToken": "eyJhbGc...",
  "user": {
    "id": "...",
    "username": "john_doe",
    "email": "john@example.com",
    "role": "user"
  }
}
```

### 2. Login
```bash
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "password": "SecurePass123"
  }'
```

### 3. Use the token for protected routes
```bash
curl -X GET http://localhost:5000/api/auth/me \
  -H "Authorization: Bearer <YOUR_TOKEN>"
```

---

## 🔗 Integration Points

### Frontend (React)
```javascript
// Store token after login
localStorage.setItem('token', response.token);

// Send token with requests
const headers = {
  'Authorization': `Bearer ${localStorage.getItem('token')}`
};

fetch('http://localhost:5000/api/auth/me', { headers });
```

### Backend (FastAPI)
```python
# Verify JWT from Frontend
import jwt

def verify_auth(token: str):
    decoded = jwt.decode(token, os.getenv('JWT_SECRET'), algorithms=['HS256'])
    return decoded['id']  # Get user ID
```

---

## 📋 Project Structure

```
auth-rig/
├── index.js                    # Main server
├── .env                        # Configuration
├── package.json
├── models/User.js              # MongoDB schema
├── controllers/authController.js  # Business logic
├── routes/auth.js              # API routes
└── middleware/
    ├── auth.js                 # JWT verification
    └── errorHandler.js         # Error handling
```

---

## 🔒 Security Features

✅ **Helmet.js** — HTTP security headers
✅ **CORS** — Origin validation
✅ **Rate Limiting** — Brute-force prevention
✅ **Bcrypt** — Password hashing (12 rounds)
✅ **JWT** — Token-based sessions
✅ **Input Validation** — Schema-level checks
✅ **Error Masking** — Generic error messages

---

## 📚 API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/register` | ❌ | Create new user |
| POST | `/api/auth/login` | ❌ | Login user |
| POST | `/api/auth/refresh-token` | ❌ | Get new access token |
| GET | `/api/auth/me` | ✅ | Get current user |
| PUT | `/api/auth/profile` | ✅ | Update profile |
| POST | `/api/auth/change-password` | ✅ | Change password |
| POST | `/api/auth/logout` | ✅ | Logout |
| GET | `/api/auth/users` | ✅ Admin | List all users |
| GET | `/health` | ❌ | Health check |

---

## 🛠️ Next Steps

1. **Connect Frontend**: Update React app to use auth endpoints
2. **Protect Backend Routes**: Add JWT verification to FastAPI
3. **Session Storage**: Implement Redis-backed token blacklist
4. **Email Verification**: Add email confirmation (optional)
5. **2FA**: Add two-factor authentication (optional)

---

## 📞 Troubleshooting

**MongoDB connection fails?**
```bash
docker compose ps
docker compose restart mongo
```

**Token expired?**
Use `/api/auth/refresh-token` to get a new access token.

**Port 5000 already in use?**
```bash
lsof -i :5000  # Find process
kill -9 <PID>  # Kill it
```

---

**Your authentication system is ready to go!** 🚀
