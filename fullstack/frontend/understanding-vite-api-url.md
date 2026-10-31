# Understanding VITE_API_URL: A Beginner's Guide

## What is Vite?

**Vite** is a modern build tool for frontend applications (React, Vue, etc.). It's the engine that:
- Runs your development server (typically `localhost:5173`)
- Bundles your code for production
- Handles hot module reloading when you edit files

## The Problem: Connecting Frontend to Backend

When building a full-stack application, your frontend (running in the browser) needs to communicate with your backend API (running on a server). But here's the challenge:

**The backend URL changes depending on where you're running:**
- **Local development**: `http://localhost:8001`
- **Docker**: Proxied through nginx
- **Production**: A completely different domain

Hardcoding the URL in your code would mean changing it every time you deploy. That's where environment variables come in.

## What is VITE_API_URL?

`VITE_API_URL` is an **environment variable** that tells your frontend application **where to find the backend API**.

It's stored in a `.env` file and accessed in your code, allowing different values for different environments without changing your code.

## Vite's Special Rule: The `VITE_` Prefix

Vite has an important security feature:

**Only environment variables starting with `VITE_` are exposed to your browser code.**

```bash
# ✅ This works - exposed to frontend
VITE_API_URL=http://localhost:8001
VITE_APP_TITLE=My App

# ❌ These do NOT work - not exposed to browser
API_URL=http://localhost:8001
DATABASE_PASSWORD=secret123
```

**Why?** This prevents accidentally exposing sensitive variables (like database passwords, API keys) to users who can inspect your browser code.

## How to Use It

### Step 1: Create a `.env` File

In your frontend directory (where `package.json` is):

```bash
# frontend/.env
VITE_API_URL=http://localhost:8001
```

### Step 2: Access in Your Code

```javascript
// src/api.js
const API_BASE = import.meta.env.VITE_API_URL || '';

// Use it in API calls
export async function fetchData() {
  const response = await fetch(`${API_BASE}/api/data`);
  return response.json();
}
```

**Breakdown:**
- `import.meta.env.VITE_API_URL` - Gets the value from `.env`
- `|| ''` - Fallback to empty string if not set
- Template literal `${API_BASE}/api/data` - Constructs the full URL

### Step 3: Use in Components

```javascript
// src/components/UserList.jsx
import { useState, useEffect } from 'react';

function UserList() {
  const [users, setUsers] = useState([]);
  const API_BASE = import.meta.env.VITE_API_URL || '';

  useEffect(() => {
    fetch(`${API_BASE}/api/users`)
      .then(res => res.json())
      .then(data => setUsers(data));
  }, []);

  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

## Different Environments, Different Values

### Local Development

**File: `frontend/.env`**
```bash
VITE_API_URL=http://localhost:8001
```

**What happens:**
- Frontend runs on `http://localhost:5173`
- Backend runs on `http://localhost:8001`
- API calls go directly to `http://localhost:8001/api/...`

**Start both servers separately:**
```bash
# Terminal 1 - Backend
python -m uvicorn main:app --port 8001

# Terminal 2 - Frontend
cd frontend && npm run dev
```

### Docker/Production

**Environment variable in Docker:**
```bash
VITE_API_URL=""
# OR
VITE_API_URL=/
```

**What happens:**
- API calls go to `/api/...` (relative URL)
- Nginx (web server in frontend container) proxies these to the backend
- Backend runs in a separate container

**Example nginx configuration:**
```nginx
server {
  listen 80;

  # Serve frontend static files
  location / {
    root /usr/share/nginx/html;
    try_files $uri /index.html;
  }

  # Proxy API requests to backend
  location /api/ {
    proxy_pass http://backend:8001;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

### Production with Separate Domains

**Environment variable:**
```bash
VITE_API_URL=https://api.myapp.com
```

**What happens:**
- Frontend deployed to `https://myapp.com`
- Backend deployed to `https://api.myapp.com`
- API calls go to `https://api.myapp.com/api/...`

## Common Scenarios

| Scenario | VITE_API_URL | Where Requests Go |
|----------|--------------|-------------------|
| **Local dev (separate servers)** | `http://localhost:8001` | Directly to backend on port 8001 |
| **Docker Compose** | `""` (empty) | Nginx proxies `/api/*` to backend container |
| **Production (same domain)** | `""` (empty) | Nginx proxies to backend |
| **Production (different domain)** | `https://api.example.com` | Backend API domain |
| **Backend on different machine** | `http://192.168.1.100:8001` | Specific IP address |

## Building for Production

When you build your Vite app, the environment variable is **baked into the bundle**:

```bash
# Build with production API URL
VITE_API_URL=https://api.myapp.com npm run build
```

The resulting JavaScript will have the value hardcoded (not as a variable anymore).

**Important:** After building, you cannot change the API URL without rebuilding. Plan your deployment strategy accordingly.

## Debugging Tips

### Check What Value Is Being Used

In your browser console:
```javascript
console.log(import.meta.env.VITE_API_URL);
```

### Check Network Requests

Open browser DevTools → Network tab → Look at the request URLs to see where API calls are actually going.

### Common Issues

**Problem:** Changes to `.env` not taking effect
**Solution:** Restart the dev server (`npm run dev`)

**Problem:** `undefined` value
**Solution:** Make sure variable starts with `VITE_` prefix

**Problem:** CORS errors in local development
**Solution:** Backend needs CORS middleware to allow `localhost:5173`

## Example Project Structure

```
my-app/
├── backend/
│   ├── main.py              # FastAPI/Express server
│   └── .env                 # Backend secrets (DB, API keys)
│
├── frontend/
│   ├── src/
│   │   ├── api.js           # API client using VITE_API_URL
│   │   └── components/
│   ├── .env                 # VITE_API_URL=http://localhost:8001
│   ├── .env.production      # VITE_API_URL=https://api.myapp.com
│   └── package.json
│
└── docker-compose.yml       # Sets VITE_API_URL="" for containers
```

## Multiple Environment Files

Vite supports different `.env` files for different modes:

```bash
.env                  # Loaded in all cases
.env.local            # Loaded in all cases, ignored by git
.env.development      # Loaded in development mode
.env.production       # Loaded in production mode
```

**Example:**
```bash
# .env.development
VITE_API_URL=http://localhost:8001

# .env.production
VITE_API_URL=https://api.myapp.com
```

**Build commands:**
```bash
# Uses .env.development
npm run dev

# Uses .env.production
npm run build
```

## Security Best Practices

### ✅ DO:
- Use `VITE_` prefix for frontend-visible variables
- Store the `.env.local` file in `.gitignore`
- Use different values for dev/production
- Document required environment variables in README

### ❌ DON'T:
- Put secrets (API keys, passwords) in `VITE_*` variables
- Commit `.env.local` to git
- Use the same API URL for all environments
- Assume environment variables are private (they're in browser code!)

## Real-World Example

**Complete setup for a React + FastAPI app:**

**Backend (`backend/main.py`):**
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Allow frontend to connect
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  # Dev frontend
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/api/data")
async def get_data():
    return {"message": "Hello from API"}
```

**Frontend (`frontend/.env`):**
```bash
VITE_API_URL=http://localhost:8001
```

**Frontend (`frontend/src/api.js`):**
```javascript
const API_BASE = import.meta.env.VITE_API_URL || '';

export async function fetchData() {
  const response = await fetch(`${API_BASE}/api/data`);
  if (!response.ok) {
    throw new Error(`API error: ${response.status}`);
  }
  return response.json();
}
```

**Frontend (`frontend/src/App.jsx`):**
```javascript
import { useState, useEffect } from 'react';
import { fetchData } from './api';

function App() {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchData()
      .then(setData)
      .catch(setError);
  }, []);

  if (error) return <div>Error: {error.message}</div>;
  if (!data) return <div>Loading...</div>;

  return <div>{data.message}</div>;
}

export default App;
```

## Summary

- **`VITE_API_URL`** tells your frontend where the backend API is located
- **`VITE_` prefix** is required for Vite to expose variables to browser code
- **Changes require restart** - Vite reads `.env` on startup
- **Different environments use different values** - local dev vs Docker vs production
- **Security by design** - Only `VITE_*` variables are exposed (prevents leaking secrets)
- **Built into bundle** - Value is hardcoded after `npm run build`

## Further Reading

- [Vite Environment Variables Documentation](https://vitejs.dev/guide/env-and-mode.html)
- [Vite Config Reference](https://vitejs.dev/config/)
- [CORS Explanation](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

---

**Created:** 2026-02-06
**Tags:** #vite #frontend #environment-variables #api #beginner-guide
