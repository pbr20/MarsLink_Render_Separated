# MarsLink — Separate Render Deployment (manual setup)

This package is structured for **two separate Render services** plus a **Render PostgreSQL database**. The local serial collector is NOT deployed to Render.

```
MarsLink_Render_Separated/
  backend/         Flask API + ML + requirements + .python-version
  frontend/        Static HTML/CSS/JS and images
  collector/       Runs on Windows PC connected to ESP32 master
  firmware/        Original Arduino sketches
```

## Step 1 — Upload to GitHub

Create a GitHub repository. Upload the CONTENTS of `MarsLink_Render_Separated/` to the repository root (so `backend/` and `frontend/` are at the top level). Do not upload `.env` or secrets.

## Step 2 — Create PostgreSQL on Render

Dashboard > New > Postgres. Choose the region you will use for your backend. Create it, then copy the **Internal Database URL** from the database connection page. This is a secret. Free Render Postgres expires after 30 days, so use a paid plan or another persistent database if needed.

## Step 3 — Create backend Web Service on Render

Dashboard > New > Web Service > connect the GitHub repo.

- Name: `marslink-api` (or any available name)
- Language/Runtime: Python 3
- Branch: `main`
- **Root Directory: `backend`**
- Build Command: `pip install -r requirements.txt`
- Start Command: `gunicorn --bind 0.0.0.0:$PORT --workers 1 --threads 2 --timeout 120 app:app`
- Instance: Free for demos (or a paid plan for reliability)
- Health Check Path: `/health`
- Python version: backend/.python-version pins Python 3.11; if needed set `PYTHON_VERSION` to a specific Python 3.11.x version in Environment.

Environment variables on backend:

- `DATABASE_URL`: **Internal Database URL** copied from Render Postgres
- `INGEST_TOKEN`: generate a strong random secret (never commit to GitHub)
- `FRONTEND_URL`: leave unset for the first deploy, then set to the exact public URL of your frontend (e.g. `https://marslink-ui.onrender.com`, no trailing slash) and redeploy

Deploy and test `https://YOUR-BACKEND.onrender.com/health`. Expected response: `{"status":"ok","database":"connected"}`. Root `/` shows API status. `/api/data` returns JSON even when no nodes are online.

## Step 4 — Configure frontend URL

Open `frontend/config.js` and replace the placeholder with the actual backend URL, e.g.

```
window.MARSLINK_API_BASE_URL = "https://marslink-api-xxxx.onrender.com";
```

This is a public URL, NOT a database password or ingestion token. Commit this change to GitHub.

## Step 5 — Create frontend Static Site on Render

Dashboard > New > Static Site > connect the SAME GitHub repository.

- Name: `marslink-ui` (or any available name)
- Branch: `main`
- **Root Directory: `frontend`**
- Build Command: `echo "Static files ready"`
- Publish Directory: `.` (the `frontend` directory itself, because Root Directory is `frontend`)

Deploy. Copy the exact frontend public URL (e.g. `https://marslink-ui-xxxx.onrender.com`).

## Step 6 — Finish CORS configuration

Go back to backend Web Service > Environment. Set:

```
FRONTEND_URL=https://marslink-ui-xxxx.onrender.com
```

Use the exact frontend URL, no trailing slash. Save and redeploy backend. CORS only controls browser access from origins; it is NOT authentication. The backend's GET APIs are public.

## Step 7 — Connect the ESP32 master from Windows

On the Windows PC connected by USB to the master ESP32:

```
cd collector
py -m pip install -r requirements.txt
$env:MARSLINK_SERIAL_PORT="COM6"
$env:MARSLINK_SERVER="https://YOUR-BACKEND.onrender.com"
$env:MARSLINK_TOKEN="YOUR-SAME-INGEST_TOKEN"
py collector.py
```

Run the PowerShell `$env:` lines in PowerShell (not CMD). Set the actual COM port visible in Windows Device Manager or Arduino IDE. Leave the collector running to forward telemetry. A Render cloud service cannot read the PC's USB serial port.

## Step 8 — Verify

- Backend `/` responds with API status.
- Backend `/health` says database connected.
- Backend `/api/data` returns a `nodes` object with BB/CC/DD.
- Backend `/api/forecast` shows collecting/stale until sufficient continuous real readings are received (usually about 80–100 minutes).
- Frontend loads images, charts, and calls the backend URL; check browser Developer Tools > Network for failed API requests.
- Collector shows `Uploaded:` messages. `401` means wrong/missing token; `400` can mean unsupported node or invalid payload; `503` from forecast means backend/database issue.

## Common Render errors

- `ModuleNotFoundError: No module named 'forecast'`: Root Directory must be `backend`.
- `ModuleNotFoundError: No module named 'flask'`: ensure Python 3 runtime and correct build command.
- `gunicorn: command not found`: install requirements; ensure successful build.
- Python/scikit-learn wheel build errors: ensure `.python-version` was committed under `backend/` and Python 3.11 is used.
- `Database initialization warning`: check `DATABASE_URL` and that Postgres is live; `/health` is the DB diagnostic.
- `No open ports detected`: start command must bind `0.0.0.0:$PORT`.
- CORS errors: `FRONTEND_URL` must exactly match the frontend origin, including `https://`, without trailing slash.
- `Failed to fetch` in browser: check `frontend/config.js`, backend URL, CORS, and service availability.
- Dashboard empty: no serial data forwarded, incorrect COM port, or wrong token; frontend can be working while readings are empty.
- Render Free web service can sleep after 15 minutes idle; it may take around a minute to wake. Render Free PostgreSQL expires after 30 days.

This is a deployment-ready layout, not a guarantee of successful cloud deployment: no live Render account or production database was available for end-to-end verification.
