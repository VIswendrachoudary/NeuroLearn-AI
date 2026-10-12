# NeuroLearn AI – Architecture

> Extracted from the main PRD · Section 2 · Do not edit here; update the main document and re-extract.

## 2. System architecture

### 2.1 Overview

```
Browser (React + Vite + TypeScript)
        |  HTTPS, JSON, JWT
        v
Node.js / Express API  (auth, validation, orchestration, caching)
   |                  \
   | SQL               \ HTTP (internal, API key header)
   v                    v
PostgreSQL 16        Python FastAPI ML service
(system of record)   (twin, predict, explain, root gap, recommend)
                        |
                        v
              model artifacts + graph.json (read only files)
```

### 2.2 Responsibilities

| Component | Owns | Never does |
| --- | --- | --- |
| Frontend | Rendering, routing, client state, charts | Business rules, scoring |
| Express API | Auth, authorisation, validation, persistence, orchestration, caching, fallback | ML computation |
| ML service | Feature building, model inference, SHAP, twin math, graph traversal, recommendations | Authentication, direct database writes |
| PostgreSQL | All persistent data | Logic beyond constraints and indexes |

Rule: the ML service is stateless. Express sends it the data it needs (attempt history, graph slice) and stores what comes back. This keeps the ML service easy to test and replace.

### 2.3 Technology stack

| Layer | Choice |
| --- | --- |
| Frontend | React 18, Vite, TypeScript, React Router, TanStack Query, Tailwind CSS, Chart.js, Cytoscape.js |
| Backend | Node.js LTS, Express, pg, zod, jsonwebtoken, bcrypt, helmet, cors, express-rate-limit, pino |
| ML service | Python 3.11, FastAPI, uvicorn, pydantic v2, pandas, numpy, scikit-learn, xgboost, shap, networkx, joblib |
| Database | PostgreSQL 16, plain SQL migrations |
| Testing | pytest, Vitest, supertest, Playwright (one end to end test) |
| DevOps | Docker, docker compose, GitHub Actions |

Pick the current stable version of each when you install it, then pin it in package.json and requirements.txt.

### 2.4 Key request flows

Flow A, student logs a new attempt (POST /attempts):

1. Express validates the body and the JWT, and checks the student owns the record.
2. Express inserts the attempt in a transaction, using the client\_attempt\_id from the request so a retried request cannot create a second row (unique index, ON CONFLICT DO NOTHING).
3. Express loads the last 60 days of attempts for that student and the graph slice around the concept.
4. Express makes one call to ML /analyze, which returns the updated twin, the risk and top 3 reasons per changed concept, the root gaps and the new path together.
5. Express stores a twin snapshot, the new predictions and the new recommendation, and marks the old recommendation stale.
6. Express returns the updated twin summary, the top risks and a computed\_at timestamp, and only after steps 3 to 5 have finished, so the response itself is the fresh state.

Flow B, student opens the dashboard (GET risks, twin, path):

1. Express checks for predictions newer than the cache window (10 minutes).
2. If fresh, return them. If stale, recompute through the ML service.
3. If the ML service fails, return the stored predictions with stale set to true.

Flow C, faculty opens the cohort page: Express reads stored predictions for every student in the cohort in one query and aggregates in SQL, with no ML calls.

### 2.5 Repository structure

```
neurolearn-ai/
  README.md
  docker-compose.yml
  .env.example
  docs/            prd.md, api.md, architecture.md, model-card.md, demo-script.md
  data/            concepts.json, prerequisites.json, resources.json, generated/
  db/              migrations/, seed/
  ml-service/
    app/           main.py, routers/, schemas.py, services/
    training/      generate_data.py, features.py, train.py, evaluate.py
    models/        model_v1.joblib, features.json, metrics.json
    tests/
  backend/
    src/           app.ts, routes/, middleware/, services/, db/
    tests/
  frontend/
    src/           pages/, components/, api/, hooks/, lib/
    tests/
  scripts/         seed.sh, reset.sh, demo.sh
```

### 2.6 Configuration

| Variable | Used by | Purpose |
| --- | --- | --- |
| DATABASE\_URL | backend | Postgres connection |
| JWT\_SECRET, JWT\_EXPIRES\_IN | backend | Token signing and lifetime |
| ML\_SERVICE\_URL, ML\_API\_KEY | backend, ML service | Internal call target and shared secret |
| CORS\_ORIGIN | backend | Allowed frontend origin |
| PREDICTION\_CACHE\_MINUTES | backend | Freshness window |
| MODEL\_PATH, GRAPH\_PATH | ML service | Artifact locations |
| VITE\_API\_URL | frontend | Backend base URL |

Never commit real secrets. Commit .env.example only.
