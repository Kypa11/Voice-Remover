# STEM — Vocal Remover

AI-powered vocal removal using [Demucs](https://github.com/facebookresearch/demucs).
Upload an MP3/WAV, get a clean instrumental back. Free, local, no auth.

---

## Project Structure

```
vocal-remover/
├── backend/
│   ├── app.py            # Flask API (routes, job management, cleanup)
│   ├── worker.py         # Demucs processing + progress tracking
│   ├── requirements.txt  # Python dependencies
│   ├── uploads/          # Temp input files (auto-cleaned)
│   └── outputs/          # Processed stems (auto-cleaned after 1hr)
└── frontend/
    ├── index.html
    ├── package.json
    ├── vite.config.js
    └── src/
        ├── main.jsx
        ├── index.css
        ├── App.jsx / App.module.css
        ├── UploadPanel.jsx / UploadPanel.module.css
        ├── ProcessingPanel.jsx / ProcessingPanel.module.css
        ├── DonePanel.jsx / DonePanel.module.css
        ├── api.js              # All fetch calls in one place
        └── useJobPoller.js     # Polling hook
```

---

## Prerequisites

- **Python 3.10+**
- **Node.js 18+**
- ~2GB disk space (PyTorch + Demucs model download on first run)
- No GPU required — runs on CPU (processing takes 2–5 min per track)

---

## Setup

### 1. Backend

```bash
cd vocal-remover/backend

# Create and activate a virtual environment (recommended)
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Start the Flask server
python app.py
```

Flask will start on **http://localhost:5000**.

> **First run:** Demucs will automatically download the `htdemucs` pretrained
> model (~80MB) on the first job. Subsequent runs use the cached model.

---

### 2. Frontend

```bash
cd vocal-remover/frontend

# Install dependencies
npm install

# Start the dev server
npm run dev
```

Vite will start on **http://localhost:3000**.

The Vite proxy (`vite.config.js`) forwards all `/api/*` requests to
`http://localhost:5000` — no CORS issues in development.

---

## Running the App

You need **two terminals** running simultaneously:

| Terminal | Command | URL |
|---|---|---|
| 1 — Backend  | `python app.py` (inside `backend/`) | http://localhost:5000 |
| 2 — Frontend | `npm run dev` (inside `frontend/`)  | http://localhost:3000 |

Open **http://localhost:3000** in your browser.

---

## User Flow

1. Drag & drop (or click to browse) an audio file — MP3, WAV, FLAC, M4A, OGG up to 80MB
2. Click **Remove Vocals →**
3. File uploads to Flask; a background thread starts Demucs
4. Frontend polls `/api/status/:job_id` every 2.5 seconds and shows live progress
5. When done, click **Download WAV ↓** to get `<trackname>_instrumental.wav`
6. Server deletes both the upload and output after 1 hour

---

## Configuration

Environment variables for the backend (set before running `python app.py`):

| Variable | Default | Description |
|---|---|---|
| `PORT` | `5000` | Flask port |
| `MAX_FILE_MB` | `80` | Max upload size in MB |
| `ALLOWED_ORIGINS` | `*` | CORS allowed origins (lock down for production) |

Example:
```bash
MAX_FILE_MB=120 PORT=8080 python app.py
```

---

## How It Works

Demucs `htdemucs` is a hybrid transformer model that separates audio into
4 stems: vocals, drums, bass, other. We use the `--two-stems vocals` flag
which produces only two outputs and cuts processing time in half:

- `vocals.wav` — isolated vocal track
- `no_vocals.wav` — everything else (the instrumental)

The worker streams Demucs stdout line-by-line, parses progress segments
(e.g. `3/12`), and updates the in-memory job store so the frontend can
show a real progress percentage.

---

## Troubleshooting

**`ModuleNotFoundError: No module named 'demucs'`**
→ Make sure your virtualenv is activated: `source venv/bin/activate`

**Processing stuck at 5% for a long time**
→ First run downloads the model. Check terminal for download progress.

**`FileNotFoundError: no_vocals.wav`**
→ Demucs failed silently. Check the backend terminal for the full error output.
   Usually caused by a corrupted file or unsupported codec.

**Frontend shows "Network error"**
→ Make sure Flask is running on port 5000. Check for firewall issues.

**Very slow processing**
→ Normal on CPU. A 3-minute song takes ~3–6 minutes. Add a GPU later for 5–10× speedup.

---

## Scaling Up (Post-MVP)

The architecture is modular — each layer can be upgraded independently:

| Layer | MVP | Production swap |
|---|---|---|
| Job queue | `threading` | Celery + Redis |
| Job state | in-memory dict | Redis / Postgres |
| File storage | local disk | AWS S3 / GCS |
| Processing | CPU (Demucs) | GPU instance (CUDA auto-detected) |
| Deployment | local | Docker + Gunicorn + Nginx |

### Switching to Celery (when you need multi-worker scale)

```python
# In worker.py, add the @celery.task decorator
from celery import Celery
celery = Celery("worker", broker="redis://localhost:6379/0")

@celery.task
def process_audio(job_id, input_path, output_dir, ...):
    ...

# In app.py, change:
thread = threading.Thread(...)  # remove this
process_audio.delay(job_id, input_path, OUTPUT_DIR)  # add this
```

### Docker (quick production setup)

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:5000", "app:app"]
```

---

## License

MIT. Demucs is MIT licensed by Meta Research.
