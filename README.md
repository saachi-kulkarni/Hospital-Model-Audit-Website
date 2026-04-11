# 🏥 Hospital AI Model Audit — Web Application

A full-stack web application for auditing clinical AI model performance. Enter your model's metrics manually or upload a JSON file, and the app sends them to a locally-running fine-tuned LLM ([PhantomAjusshi/phi3-auditor-merged](https://huggingface.co/PhantomAjusshi/phi3-auditor-merged)) which returns a structured health classification and detailed explanation.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Setup & Installation](#setup--installation)
- [Running the App](#running-the-app)
- [API Reference](#api-reference)
- [Environment Variables](#environment-variables)
- [Tech Stack](#tech-stack)
- [Related Repository](#related-repository)

---

## Overview

The app provides a clean UI where users can:
1. Enter clinical ML model metrics (AUC, accuracy, ECE, drift, etc.) manually or via JSON upload
2. Submit them to a FastAPI backend
3. Receive a verdict label (e.g. *"Calibration Failure"*, *"Major Drift"*) and a ~300-word structured analysis from the Phi-3 auditor model

The model is served locally via **llama.cpp** using a quantized GGUF file (`phi3-auditor-q4.gguf`), keeping inference entirely on-device.

---

## Architecture

```
Browser (Next.js :3000)
        │
        │  POST /analyze  { metrics JSON }
        ▼
FastAPI Backend (:8000)
        │
        │  POST /completion  { prompt }
        ▼
llama.cpp server (:8080)
        │
        │  Loads
        ▼
models/phi3-auditor-q4.gguf  (~2.2 GB)
```

---

## Project Structure

```
Hospital-Model-Audit-Website/
│
├── app/                            # Next.js App Router
│   ├── page.tsx                    # Main UI — metrics input + model output
│   ├── layout.tsx                  # Root layout with Vercel Analytics
│   └── globals.css                 # Global styles
│
├── backend/
│   ├── main.py                     # FastAPI server — /analyze endpoint
│   ├── model.py                    # HuggingFace model loader (alternative inference path)
│   ├── requirements.txt            # Python dependencies
│   ├── start.sh                    # Launches llama.cpp server then FastAPI
│   └── llama_server.log            # Runtime log from llama.cpp (auto-generated)
│
├── components/
│   ├── theme-provider.tsx
│   └── ui/                         # shadcn/ui component library (40+ components)
│
├── hooks/                          # Custom React hooks
├── lib/utils.ts                    # Tailwind utility helpers
├── public/                         # Static assets & icons
│
├── models/                         # ← Place phi3-auditor-q4.gguf here (not in repo)
├── llama.cpp/                      # ← Clone & build separately (not in repo)
│
├── next.config.mjs
├── package.json
├── tsconfig.json
└── SETUP.md                        # New-machine setup guide
```

---

## Features

- **Two input modes** — manual field-by-field entry or JSON file upload
- **Live backend status indicator** — checks `/health` on load; shows a warning banner if the backend is unreachable
- **Structured AI output** — the model returns a label + a 4-section explanation (Observations → Diagnosis → Impact → Recommendation)
- **Mock mode** — set `USE_MOCK=true` in the backend to get instant placeholder responses without loading the model (useful for frontend development)
- **Dark mode** support via `next-themes`
- **5-minute request timeout** with user-friendly timeout and CORS error messages
- **JSON schema enforcement** — llama.cpp is prompted with a JSON schema so the output is always parseable

---

## Prerequisites

| Tool | Version | Notes |
|---|---|---|
| Node.js | 18+ | For the Next.js frontend |
| Python | 3.10+ | For the FastAPI backend |
| C++ Compiler | — | To build llama.cpp (`gcc` / `clang` / `cmake`) |
| Git | — | To clone llama.cpp |
| RAM | 8 GB+ | For running the 4-bit quantized model |
| GPU (optional) | — | Speeds up inference via `-ngl 99` flag in `start.sh` |

---

## Setup & Installation

### Step 1 — Clone the repository

```bash
git clone <your-repo-url>
cd Hospital-Model-Audit-Website
```

### Step 2 — Build llama.cpp

The llama.cpp binary is excluded from the repo (system-specific build). Compile it from source:

```bash
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make
# OR with cmake:
# mkdir build && cd build && cmake .. && cmake --build . --config Release
cd ..
```

> The `start.sh` script expects the binary at `llama.cpp/build/bin/llama-server`. Adjust the path in `backend/start.sh` if your build places it elsewhere.

### Step 3 — Download the model

The quantized GGUF model (`phi3-auditor-q4.gguf`, ~2.2 GB) is not included in the repo. Create the `models/` directory and place it there:

```bash
mkdir -p models
# Transfer phi3-auditor-q4.gguf into models/
# The model was quantized from: https://huggingface.co/PhantomAjusshi/phi3-auditor-merged
```

### Step 4 — Backend setup

```bash
cd backend
python3 -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Step 5 — Frontend setup

```bash
# From the project root
npm install --legacy-peer-deps
```

---

## Running the App

### Option A — All-in-one (recommended)

The `start.sh` script kills any existing processes on ports 8000/8080, starts the llama.cpp inference server, waits for it to load, then starts FastAPI:

```bash
chmod +x backend/start.sh
./backend/start.sh
```

Then open a separate terminal for the frontend:

```bash
npm run dev
```

Visit [http://localhost:3000](http://localhost:3000).

### Option B — Manual (three terminals)

**Terminal 1 — llama.cpp server:**
```bash
./llama.cpp/build/bin/llama-server \
    -m ./models/phi3-auditor-q4.gguf \
    --port 8080 \
    -c 2048 \
    -ngl 99 \
    --host 127.0.0.1
```

**Terminal 2 — FastAPI backend:**
```bash
cd backend
source venv/bin/activate
python3 main.py
```

**Terminal 3 — Next.js frontend:**
```bash
npm run dev
```

### Option C — Mock mode (no model required)

Useful during frontend development when you don't want to load the full model:

```bash
cd backend
USE_MOCK=true python3 main.py
```

---

## API Reference

Base URL: `http://localhost:8000`

### `GET /health`
Returns backend and llama.cpp server status.

**Response:**
```json
{ "status": "online" }
// or if the model server is down:
{ "status": "backend_online_but_model_server_down" }
```

### `POST /analyze`
Accepts model performance metrics and returns an audit verdict.

**Request body** (all fields optional — send at least one):
```json
{
  "auc": 0.8523,
  "accuracy": 0.7865,
  "precision": 0.8234,
  "recall": 0.7432,
  "f1Score": 0.7823,
  "ece": 0.0452,
  "brier": 0.1234,
  "drift": 0.0523,
  "missingRate": 0.0145,
  "labelShift": 0.0832,
  "positiveRate": 0.4521,
  "dataIntegrity": 2
}
```

**Response:**
```json
{
  "label": "Needs Review",
  "explanation": "1. Observations: ...\n2. Diagnosis: ...\n3. Impact: ...\n4. Recommendation: ..."
}
```

**Generation settings:** `n_predict: 1024`, `temperature: 0.5`, JSON schema enforced via llama.cpp grammar.

---

## Environment Variables

### Frontend — `.env.local`

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
```

Defaults to `http://localhost:8000` if not set. Update this to your production API URL when deploying.

### Backend

| Variable | Default | Description |
|---|---|---|
| `USE_MOCK` | `False` | Return mock responses without loading the model |
| `HF_TOKEN` | — | Optional HuggingFace token (needed only if using the `model.py` HF inference path) |

---

## Tech Stack

### Frontend

| Package | Version | Role |
|---|---|---|
| Next.js | 16.0.7 | React framework (App Router) |
| React | 19.2.0 | UI library |
| TypeScript | 5.x | Type safety |
| Tailwind CSS | 4.x | Styling |
| shadcn/ui + Radix UI | — | Accessible component library |
| Recharts | 2.15.4 | Chart components |
| Vercel Analytics | 1.3.1 | Usage analytics |
| next-themes | 0.4.6 | Dark mode |
| Zod | 3.25.76 | Schema validation |

### Backend

| Package | Role |
|---|---|
| FastAPI | REST API server |
| Uvicorn | ASGI server |
| Requests | HTTP calls to llama.cpp `/completion` endpoint |
| llama.cpp | Local LLM inference engine (GGUF format) |
| `phi3-auditor-q4.gguf` | 4-bit quantized Phi-3 auditor model (~2.2 GB) |

---

## Related Repository

The fine-tuned model powering this app was trained in a separate project:

> **Hospital-Audit-Trained-Model** — LoRA fine-tuning of `microsoft/Phi-3-mini-4k-instruct` on 5,000 synthetic clinical audit reports using 8-bit quantization and PEFT.
>
> Merged model on HuggingFace: [PhantomAjusshi/phi3-auditor-merged](https://huggingface.co/PhantomAjusshi/phi3-auditor-merged)
