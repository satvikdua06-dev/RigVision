# FastAPI Integration Guide

## How to Protect Your Backend Routes with Auth-Rig

This guide shows how to verify JWT tokens from the auth service in your FastAPI backend.

---

## 1. Install Dependencies

```bash
pip install pyjwt python-dotenv
```

---

## 2. Create JWT Verification Middleware

Create `backend/middleware/auth.py`:

```python
import jwt
import os
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthCredentials

JWT_SECRET = os.getenv('JWT_SECRET', 'change-me-in-production')
JWT_ALGORITHM = 'HS256'

security = HTTPBearer()

async def verify_token(credentials: HTTPAuthCredentials = Depends(security)):
    """Verify JWT token and return user ID"""
    token = credentials.credentials
    try:
        payload = jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALGORITHM])
        user_id = payload.get('id')
        user_role = payload.get('role')
        
        if not user_id:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail='Invalid token'
            )
        
        return {'id': user_id, 'role': user_role}
    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail='Token has expired'
        )
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail='Invalid token'
        )

def require_admin(user = Depends(verify_token)):
    """Verify user is admin"""
    if user['role'] != 'admin':
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail='Admin access required'
        )
    return user
```

---

## 3. Share JWT_SECRET Between Services

### Option A: Shared Environment File

In your root `.env`:
```env
JWT_SECRET=your_super_secret_jwt_key_change_this_in_production_12345
```

Load in FastAPI main.py:
```python
from dotenv import load_dotenv
import os

load_dotenv()
jwt_secret = os.getenv('JWT_SECRET')
```

### Option B: Config Service

Create `backend/config.py`:
```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    JWT_SECRET = os.getenv('JWT_SECRET')
    JWT_ALGORITHM = 'HS256'
    AUTH_API_URL = os.getenv('AUTH_API_URL', 'http://localhost:5000/api/auth')
```

---

## 4. Protect Your Routes

### Protected Endpoint
```python
from fastapi import FastAPI, Depends
from middleware.auth import verify_token

app = FastAPI()

@app.get('/api/sensors/current')
async def get_sensors(user = Depends(verify_token)):
    """Get sensor data - requires authentication"""
    user_id = user['id']
    
    # Your logic here
    return {
        'success': True,
        'data': [...],
        'requested_by': user_id
    }
```

### Admin-Only Endpoint
```python
from middleware.auth import require_admin

@app.delete('/api/users/{user_id}')
async def delete_user(user_id: str, admin = Depends(require_admin)):
    """Delete user - admin only"""
    # Your logic here
    return {'success': True, 'message': 'User deleted'}
```

### Optional Authentication (Public by Default)
```python
from typing import Optional

@app.get('/api/public-data')
async def get_public_data(user: Optional[dict] = Depends(verify_token)):
    """Optional auth - works with or without token"""
    if user:
        return {'data': [...], 'user_id': user['id']}
    else:
        return {'data': [...]}
```

---

## 5. Example: Protect CV Pipeline Endpoints

```python
# backend/main.py

from fastapi import FastAPI, Depends
from middleware.auth import verify_token, require_admin

app = FastAPI()

@app.get('/api/cv/persons')
async def get_tracked_persons(user = Depends(verify_token)):
    """Get tracked persons - authenticated users only"""
    # Get from Redis
    persons = redis_client.get('rigvision:persons')
    return {
        'success': True,
        'persons': persons,
        'accessed_by': user['id']
    }

@app.get('/api/cv/statistics')
async def get_cv_statistics(admin = Depends(require_admin)):
    """CV statistics - admin only"""
    stats = {
        'total_detections': 15234,
        'persons_tracked': 8,
        'accuracy': 0.94
    }
    return {'success': True, 'stats': stats}

@app.post('/api/cv/calibration')
async def start_calibration(config: dict, admin = Depends(require_admin)):
    """Start calibration - admin only"""
    # Trigger calibration
    return {'success': True, 'message': 'Calibration started'}
```

---

## 6. CORS Configuration for Auth Service

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        'http://localhost:5173',  # Frontend
        'http://localhost:5174',  # Sensor Console
        'http://localhost:5000'   # Auth service
    ],
    allow_credentials=True,
    allow_methods=['*'],
    allow_headers=['*']
)
```

---

## 7. Test Protected Endpoint

### Using curl:
```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "test@rigvision.com", "password": "password"}' \
  | jq -r '.token')

# Call protected endpoint
curl -X GET http://localhost:8000/api/cv/persons \
  -H "Authorization: Bearer $TOKEN"
```

### Using Python:
```python
import requests

# Login
auth_response = requests.post(
    'http://localhost:5000/api/auth/login',
    json={'email': 'test@rigvision.com', 'password': 'password'}
)
token = auth_response.json()['token']

# Call protected endpoint
headers = {'Authorization': f'Bearer {token}'}
response = requests.get(
    'http://localhost:8000/api/cv/persons',
    headers=headers
)
print(response.json())
```

---

## 8. Handle Token Expiration

```python
from fastapi import HTTPException, status

@app.get('/api/protected')
async def protected_route(user = Depends(verify_token)):
    try:
        # Your logic
        pass
    except HTTPException as e:
        if e.status_code == status.HTTP_401_UNAUTHORIZED:
            # Token expired - client should refresh using refresh token
            return {
                'success': False,
                'error': 'Token expired',
                'hint': 'Use refresh token to get new access token'
            }
        raise
```

---

## 9. Environment Variables

Add to your `backend/.env`:

```env
# Auth
JWT_SECRET=your_super_secret_jwt_key_change_this_in_production_12345
AUTH_API_URL=http://localhost:5000/api/auth

# CORS
CORS_ORIGINS=http://localhost:5173,http://localhost:5174,http://localhost:5000
```

---

## ✅ Checklist

- [ ] Install PyJWT
- [ ] Create middleware/auth.py
- [ ] Share JWT_SECRET between services
- [ ] Add @Depends(verify_token) to protected routes
- [ ] Test endpoints with curl or Postman
- [ ] Update frontend to send Authorization header
- [ ] Deploy with HTTPS in production

---

**Your backend is now protected by JWT tokens!** 🔒
