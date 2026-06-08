# Resilient Real-Time Geospatial Telemetry

This project demonstrates a minimal but complete real-time geospatial telemetry pipeline:

- **Ingestion layer**: FastAPI WebSocket server accepts continuous client telemetry.
- **State + spatial layer**: Redis GEO stores latest device coordinates and enables fast geofence queries.
- **Computation + alerting**: A background worker detects geofence entry events and pushes live alerts to connected clients.
- **Simulator**: A concurrent fleet simulator generates many clients, simulates network drops, queues unsent telemetry locally, and bulk-syncs on reconnect to avoid data loss.
- **UI**: Streamlit dashboard starts/stops the simulator and visualizes telemetry/drop/queue/bulk-sync metrics.

## Demo Video

Watch the simulator in action:

https://github.com/DivyanshiDetroja/Real-time-Geospatial-Telemetry-Engine/raw/main/videos/Geospatial%20Telemetry%20Engine.mp4

## What the project achieves

The system proves the core reliability loop needed for mobile geospatial telemetry:

1. Clients send frequent location updates over WebSocket.
2. Server persists location to Redis GEO in near real-time.
3. When clients enter the Fairfax demo geofence, the server emits `geofence_entered` alerts.
4. Under simulated unstable connectivity, clients do not lose points:
   - points are queued while offline,
   - replayed via `bulk_sync` when connection is restored.

## How it works (high level)

- **Backend**
  - `backend/app/main.py`: WebSocket endpoint + app lifecycle wiring.
  - `backend/app/services/telemetry_service.py`: handles `telemetry` and `bulk_sync` writes to Redis GEO.
  - `backend/app/workers.py`: polling worker checks geofence membership and emits entry alerts.
  - `backend/app/connection_manager.py`: maps `device_id` to active WebSocket for server push.
  - `backend/app/models.py`: message schemas for inbound/outbound socket payloads.
- **Simulator**
  - `simulator/client_simulator.py`: runs `N` concurrent async clients.
  - `simulator/network_simulation.py`: probabilistic drop/reconnect behavior.
  - `simulator/queueing.py`: in-memory buffering + bulk drain.
  - `simulator/paths.py`: deterministic client movement paths.
  - `simulator/streamlit_app.py`: control panel + live metrics chart.

## Installation and run guide (separate venvs)

Use separate virtual environments for backend and simulator.

### 1) Backend setup (FastAPI + Redis client)

From repo root:

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

Start Redis (in another terminal):

```bash
redis-server
```

Start backend:

```bash
cd backend
source .venv/bin/activate
uvicorn app.main:app --reload
```

Backend endpoint:
- WebSocket: `ws://127.0.0.1:8000/ws`
- Health: `http://127.0.0.1:8000/health`

### 2) Simulator setup (Streamlit + websockets)

From repo root:

```bash
cd simulator
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

Run Streamlit UI:

```bash
cd simulator
source .venv/bin/activate
streamlit run streamlit_app.py
```

Streamlit default URL:
- `http://localhost:8501`

## Streamlit parameters explained

These controls in `simulator/streamlit_app.py` shape simulator behavior:

- **WS URL**
  - Backend WebSocket endpoint used by all simulated clients.
  - Default: `ws://127.0.0.1:8000/ws`.

- **Fleet size**
  - Number of concurrent simulated clients (async tasks).
  - Higher values increase load and message volume.

- **Drop probability per tick**
  - Probability that a connected client intentionally drops during each loop iteration.
  - Example: `0.05` means ~5% chance of drop on each tick.

- **Tick seconds**
  - Interval between generated telemetry points per client while running.
  - Smaller value = more frequent telemetry.

- **Reconnect delay (seconds)**
  - Wait time before retrying connection after a drop/send failure.
  - Also governs how long an offline client keeps queueing before retry.

- **Bulk sync batch size**
  - Max number of queued points sent in one `bulk_sync` message upon reconnect.
  - Larger batches drain queue faster but create larger payloads.

## Interpreting UI metrics

- **Sent telemetry**: live `telemetry` messages accepted in connected state.
- **Queued**: points buffered locally while offline or after send failure.
- **Bulk synced**: queued points replayed successfully via `bulk_sync`.
- **Drops**: intentional network drop events.
- **Reconnects**: successful socket reconnect attempts.
- **ACKs**: successful backend acknowledgments (`telemetry_ack` accepted).
- **Errors**: send/ack/reconnect failures observed by simulator.

## Minimal local test flow

1. Start Redis.
2. Start backend.
3. Start Streamlit simulator UI.
4. Set non-zero drop probability, click **Start**.
5. Confirm:
   - `Drops` increases,
   - `Queued` grows during outages,
   - `Bulk synced` rises after reconnects,
   - backend remains responsive.
