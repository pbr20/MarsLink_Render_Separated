# MarsLink — split deployment package

Start with **[DEPLOY_RENDER.md](DEPLOY_RENDER.md)** for a complete Render walkthrough.

- `frontend/`: Render Static Site
- `backend/`: Render Python Web Service (Flask + Ridge regression forecasts)
- Render PostgreSQL: create separately in the Render dashboard
- `collector/`: local Windows serial bridge (do not deploy to Render)
- `firmware/`: original ESP32/LoRa Arduino sketches

Configure `frontend/config.js` with your backend URL and set `FRONTEND_URL` in Render backend environment with the exact frontend URL.
