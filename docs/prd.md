# NeuroLearn AI – Product Requirements Document

> Extracted from the main PRD · Sections 1 to 4 · Do not edit here; update the main document and re-extract.

## 1. Product Requirements Document

### 1.1 Product summary

NeuroLearn AI predicts which concepts a student is about to struggle with, explains why in plain language, traces the cause back to the prerequisite concept that is really missing, and generates a personalised learning path to fix it. Faculty get a cohort view that flags at-risk students early so they can intervene before grades drop.

### 1.2 Problem

- Learning management systems serve the same content to every student.
- Teachers usually find out a student is struggling only after a poor result.
- AI chatbots answer questions but do not analyse learning behaviour or predict outcomes.

### 1.3 Goals

- G1: Maintain a Digital Learning Twin, a continuously updated model of each student's mastery and behaviour.
- G2: Predict per-concept risk of poor performance using machine learning.
- G3: Recommend a personalised, ordered learning path using the knowledge graph.
- G4: Explain every prediction and recommendation, and present them in student and faculty dashboards.

### 1.4 Non-goals (out of scope)

- Resume analyzer and AI technical support bot.
- Integration with real LMS products (LTI, Moodle) and real student data.
- Mobile apps, payments, content authoring tools, multi-language UI.
- Multiple subjects. One subject is built deeply instead.

### 1.5 Personas

- Student (Asha): wants to know what to study next and why she is falling behind.
- Faculty (Dr. Rao): wants to know which students need help this week and on which topics.

### 1.6 User stories and acceptance criteria

| ID | Story | Acceptance criteria |
| --- | --- | --- |
| US-1 | As a student I can log in and see my mastery per concept | Heatmap shows every concept with a score 0 to 100 and a colour band |
| US-2 | As a student I see which concepts I am likely to fail | Risk list sorted high to low, each with a probability |
| US-3 | As a student I see why a concept is risky | Top 3 reasons in plain English per concept |
| US-4 | As a student I see the real root gap | Root gap card names the deepest weak prerequisite and the chain to the risky concept |
| US-5 | As a student I get an ordered learning path | Path of 3 to 10 steps with resource, time estimate and a reason |
| US-6 | As a student I can log practice results and see updates | Posting an attempt updates twin, risk and path within 3 seconds |
| US-7 | As faculty I see my cohort sorted by risk | Table with risk score, trend arrow and top root gap per student |
| US-8 | As faculty I see common gaps across the class | Bar chart of root gaps ranked by number of affected students |
| US-9 | As faculty I get alerts for sudden risk rises | List of students whose risk rose 0.2 or more in 7 days |
| US-10 | As either user I can only see data I am allowed to | Students see only their own data, faculty only their cohort |

### 1.7 Functional requirements

- FR-1 Authentication with email and password, two roles (student, faculty).
- FR-2 Concept catalogue and prerequisite graph served via API and rendered visually.
- FR-3 Attempt logging: score, time spent, hints used, timestamp.
- FR-4 Twin engine: mastery per concept, velocity, consistency, engagement, snapshotted over time.
- FR-5 Prediction service: calibrated risk probability per student and concept.
- FR-6 Explanation service: SHAP based top reasons converted to sentences.
- FR-7 Root gap analysis over the prerequisite graph.
- FR-8 Recommendation engine producing an ordered path with resources and time estimates.
- FR-9 Student dashboard and faculty dashboard.
- FR-10 Fallback behaviour when the ML service is unavailable: serve the latest stored predictions with a stale marker.
- FR-11 Seed and reset scripts so the full demo can be rebuilt with two commands.

### 1.8 Non-functional requirements

| Area | Requirement |
| --- | --- |
| Performance | Cached reads under 300 ms p95, fresh prediction under 1.5 s p95 |
| Reliability | Demo path succeeds 3 runs in a row, health endpoints on every service |
| Security | Password hashing, JWT expiry, role checks on every route, input validation, rate limiting |
| Explainability | Every risk score ships with reasons, no unexplained numbers in the UI |
| Reproducibility | Fixed random seeds, versioned model artifacts and metrics file |
| Usability | Responsive from 360 px to desktop, keyboard accessible, colour contrast of AA or better |
| Maintainability | Typed API contracts, tests on core logic, one README with run steps |

### 1.9 Success metrics

| Metric | Target |
| --- | --- |
| Model AUC on held out later weeks | 0.80 or higher |
| Recall for at-risk class | 0.75 or higher at the chosen threshold |
| Calibration (Brier score) | Below 0.18 |
| Root gap accuracy against synthetic ground truth | 85 percent or higher |
| Simulated mastery gain vs syllabus order baseline | At least 20 percent higher |
| Dashboard first load | Under 2 s on localhost data |
| Demo reliability | 3 of 3 clean runs |

These targets are goals to aim for. Report what you actually achieve and be honest about it in the submission.

### 1.10 Assumptions and constraints

- Solo developer, about 6 to 8 weeks.
- Training data is synthetic and generated by you, so results show the method works and not that it generalises to real students. State this clearly.
- Free tier hosting for deployment.

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

## 3. Data model

All ids are UUIDs except concepts, which use a small integer. All timestamps are timestamptz in UTC.

| Table | Columns | Notes |
| --- | --- | --- |
| users | id, email (unique), password\_hash, name, role (student or faculty), created\_at | Role is a Postgres enum |
| cohorts | id, name, faculty\_id (users), created\_at | One faculty owns many cohorts |
| students | id, user\_id (users, unique), cohort\_id, enrolled\_at | One row per student user |
| concepts | id, code (unique), name, description, level (1 to 6), difficulty (1 to 5), subject | Nodes of the graph |
| prerequisites | concept\_id, requires\_id, strength (0 to 1, default 1) | Primary key on both ids, check concept\_id <> requires\_id |
| resources | id, concept\_id, title, type (video, article, practice), url, difficulty (1 to 5), est\_minutes | At least 2 per concept |
| attempts | id, student\_id, concept\_id, score (0 to 1), time\_spent\_sec, hints\_used, assessment\_type (quiz, assignment, practice), attempted\_at, client\_attempt\_id (uuid, unique per student) | Largest table |
| twin\_snapshots | id, student\_id, taken\_at, state (jsonb) | Append only history of the twin |
| predictions | id, student\_id, concept\_id, risk (0 to 1), model\_version, reasons (jsonb), created\_at | Keep latest per student and concept, history optional |
| recommendations | id, student\_id, generated\_at, status (active, stale, done), path (jsonb) | One active per student |
| path\_progress | id, recommendation\_id, step\_index, completed\_at | Tracks checked steps |
| model\_registry | version (pk), trained\_at, auc, recall, brier, feature\_list (jsonb), artifact\_path | Documents what is deployed |

Indexes: attempts (student\_id, concept\_id, attempted\_at desc), predictions (student\_id, created\_at desc), twin\_snapshots (student\_id, taken\_at desc), students (cohort\_id), prerequisites (requires\_id).

Integrity rules:

- Foreign keys everywhere, with ON DELETE CASCADE for student owned rows.
- Check constraints on every 0 to 1 range.
- The prerequisite graph must be a DAG. A validation script enforces this in CI, since SQL constraints cannot express it simply.
- Migrations are numbered SQL files (001\_init.sql, 002\_indexes.sql). Never edit an applied migration, add a new one.

## 4. Knowledge graph design

### 4.1 Subject choice

Use Data Structures and Algorithms foundations. It has a natural prerequisite structure, free resources are easy to find, and you can judge the graph yourself.

### 4.2 Build rules

- 36 to 40 concepts arranged in 6 levels. Each concept has 1 to 3 direct prerequisites, and level 1 concepts have none.
- Example chain: Variables and Types, Loops, Arrays, Recursion, Linked Lists, Stacks and Queues, Trees, Binary Search Trees, Graphs, Graph Traversal, Dynamic Programming.
- Store as data/concepts.json, data/prerequisites.json and data/resources.json. Then run scripts/build\_graph.py, which validates them and writes data/graph.json (nodes, edges, topological order, ancestors and descendants per concept) for the ML service. Load the three source files into the database.
- Write a validation script that fails on a cycle, an unknown id, a concept with more than 3 prerequisites, or a concept with no resources.

### 4.3 Mastery bands

- Weak: mastery below 0.5. Developing: 0.5 to 0.7. Strong: above 0.7.
- A prerequisite counts as a gap when its mastery is below 0.6.

### 4.4 Root gap algorithm

Input: student mastery map and target concept T.

1. Collect all ancestors of T using a depth first search over the prerequisite edges.
2. Keep ancestors whose mastery is below 0.6. These are gap candidates.
3. Remove candidates that still have a weak ancestor of their own. What remains are the root gaps, the deepest weak nodes.
4. Return the root gaps plus the shortest chain from each to T, ranked by (1 - mastery) multiplied by the number of descendants it blocks.
5. If no ancestor is weak, T itself is the gap.

Test with at least these cases: single chain, diamond shaped graph, two independent root gaps, no gap, and the target itself weak with strong ancestors.
