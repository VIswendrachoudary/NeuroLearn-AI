# NeuroLearn AI: PRD, Architecture and Build Plan

Adithya (24BDS0220) · Version 1.1 · 12 October 2026 · Solo build

## How to use this document

Sections 1 to 4 define what you are building and how the data and graph are shaped. Sections 5 to 12 are the engineering specs. Sections 13 to 16 are the step by step plan, risks and submission checklist. Build in the order of Section 13 and do not start a phase until the previous phase meets its Definition of Done (DoD).

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

## 5. Digital Learning Twin

The twin is a computed model of one student. It is rebuilt from the attempt history every time something changes, and a snapshot is stored so you can show progress over time.

### 5.1 Twin state (stored in twin\_snapshots.state)

```json
{
  "student_id": "uuid",
  "as_of": "2026-10-12T10:00:00Z",
  "mastery": { "c07": 0.42, "c08": 0.61 },
  "confidence": { "c07": "low", "c08": "high" },
  "velocity": { "c07": -0.05, "c08": 0.03 },
  "consistency": 0.72,
  "engagement": 0.55,
  "risk_summary": { "high": 4, "medium": 7, "low": 25 }
}
```

### 5.2 Update rules

- Prior mastery with no attempts: 0.40 for concepts with no prerequisites, otherwise 0.20 + 0.40 x mean(prerequisite mastery).
- Mastery update on each attempt: m\_new = m\_old + alpha x (score - m\_old). Use alpha = 0.35 for the first 3 attempts on a concept, then 0.25. Multiply alpha by 0.7 for practice attempts, 1.0 for quizzes and assignments.
- Forgetting, applied when reading and not stored: m\_effective = prior + (m - prior) x exp(-0.015 x days\_since\_last\_attempt).
- Velocity: slope of a straight line fitted to the last 5 scores on the concept (0 when fewer than 3 attempts).
- Consistency: 1 - min(1, std(gaps) / max(mean(gaps), 0.5)), where gaps are the days between consecutive attempts across all concepts. It is defined only with 3 or more attempts (2 or more gaps). With fewer, store 0.5 and treat it as low confidence, so there is never a division by zero.
- Engagement: attempts in the 7 days before the prediction time divided by the cohort median for that same 7 day window, capped at 1, with the median floored at 1. Compute the median only from attempts before the prediction time. A median over the whole dataset would leak future behaviour into training rows.
- Confidence: low when fewer than 3 attempts, medium for 3 to 5, high above 5. The UI must show low confidence clearly.
- Optional upgrade later: replace the moving average with Bayesian Knowledge Tracing (p\_init 0.3, p\_learn 0.15, p\_slip 0.1, p\_guess 0.2) and compare the two in the report.

### 5.3 Twin tests

- Repeated high scores raise mastery and never exceed 1.
- Repeated low scores lower mastery and never go below 0.
- Forgetting moves mastery toward the prior and never past it.
- A student with no attempts gets the prior and low confidence on every concept.

## 6. Machine learning pipeline

### 6.1 Synthetic data generator (training/generate\_data.py)

Use seed 42 so every run is identical.

- 300 students across 3 cohorts, 38 concepts, 12 simulated weeks, 5 to 25 attempts per student per concept they touch.
- Each student has a latent ability theta drawn from a normal distribution with mean 0 and standard deviation 1, and an engagement style drawn from Beta(2, 2) that sets how often they practise.
- True mastery m per concept starts at a low prior. Each practice event does: m = m + learn\_rate x readiness x (1 - m), where readiness is the mean true mastery of the prerequisites (clipped to 0.2 to 1) and learn\_rate = sigmoid(theta) x 0.25.
- Forgetting: m decays 0.5 percent per day without practice.
- Score = clip(m + noise(0, 0.1) - 0.05 x (difficulty - 3), 0, 1).
- Time spent = base time for the difficulty x (1.6 - m) x lognormal noise. Hints used = Poisson with mean 3 x (1 - m).
- Students mostly follow syllabus order, but some skip topics.
- Hidden root gaps: for 25 percent of students, set one level 2 or level 3 concept to mastery 0.15 and block its improvement. Their downstream concepts then stay weak. Save the true root gap per student to data/generated/ground\_truth.csv. This is what lets you measure root gap accuracy.
- Outputs: students.csv (id, name, email, cohort, is\_demo, with no password column), attempts.csv (with a generated client\_attempt\_id per row), ground\_truth.csv.

Five scripted demo students (fixed ids, always generated):

1. Hidden root gap in Arrays, then fails Linked Lists and Trees. This is the main demo story.
2. Steady strong student, low risk everywhere.
3. Strong at first, then a sudden engagement drop, risk climbs in the last 3 weeks.
4. Cramming pattern, long gaps then bursts, low consistency.
5. New student with only 2 attempts, shows the cold start low confidence behaviour.

### 6.2 Features (training/features.py)

Leakage rule: every feature is computed only from attempts strictly before the prediction time. Build training rows by moving a prediction time through each student's history, one row per student, concept and prediction time.

| Feature | Definition |
| --- | --- |
| avg\_score\_last3 | Mean of the last 3 scores on the concept |
| avg\_score\_all | Mean of all earlier scores on the concept |
| score\_slope\_5 | Slope of the last 5 scores |
| attempts\_count | Number of earlier attempts on the concept |
| days\_since\_last\_attempt | Days since the last attempt on the concept |
| time\_ratio | Mean time spent divided by the typical time for the concept |
| hints\_rate | Mean hints used per attempt |
| prereq\_mean\_mastery | Mean twin mastery of direct prerequisites |
| prereq\_min\_mastery | Lowest twin mastery among all ancestors |
| n\_weak\_prereqs | Count of ancestors below 0.6 |
| concept\_difficulty | Difficulty 1 to 5 |
| concept\_level | Level 1 to 6 |
| student\_mean\_mastery | Mean mastery across all concepts |
| student\_consistency | Twin consistency |
| student\_engagement | Twin engagement |

Handle missing values explicitly: no earlier attempt means attempts\_count 0 and the scores filled with the concept prior. XGBoost can also take missing values natively.

### 6.3 Label

y = 1 when the student's next attempt on that concept within 14 days scores below 0.6, otherwise 0. Rows with no next attempt in that window are dropped.

### 6.4 Splits

- Time based: train on weeks 1 to 8, validate on weeks 9 to 10, test on weeks 11 to 12. Never shuffle across time.
- Use GroupKFold with student id as the group for cross validation, so one student never appears in both train and validation folds.
- Touch the test set once, at the very end.

### 6.5 Models and training

1. Baseline: logistic regression with standardised features.
2. Main model: XGBoost classifier. Search ranges: max\_depth 3 to 6, learning\_rate 0.03 to 0.1, min\_child\_weight 3 to 10, subsample 0.8, colsample\_bytree 0.8, up to 600 trees with early stopping after 30 rounds. Use 30 to 50 random search trials.
3. Class imbalance: set scale\_pos\_weight to negatives divided by positives.
4. Calibration: fit isotonic calibration on the validation set so the output is a trustworthy probability.
5. Threshold: choose the lowest threshold that reaches recall 0.75 on validation. Risk bands: low below 0.30, medium 0.30 to 0.60, high 0.60 and above.

### 6.6 Evaluation (training/evaluate.py)

- Metrics on the test set: ROC AUC, PR AUC, precision and recall at the threshold, Brier score.
- Plots saved to docs/: calibration curve, confusion matrix, ROC curve, SHAP summary plot.
- Comparison table: logistic regression vs XGBoost.
- Ablation: train again without the prerequisite features (prereq\_mean\_mastery, prereq\_min\_mastery, n\_weak\_prereqs). The drop in score is your evidence that the knowledge graph adds value, which is a core claim of the project.
- Root gap accuracy: for students with a ground truth root gap, check whether the algorithm returns it in its top 1 and top 3.
- Error analysis: list the 10 worst false negatives and explain them in the model card.

### 6.7 Explanations with SHAP

- Use shap.TreeExplainer on the uncalibrated XGBoost model. Sort features by their positive SHAP values (the ones pushing risk up), keep the top 3, and merge near duplicates such as the three prerequisite features into one reason.
- Convert each feature into a sentence with a template, then fill in the real numbers.

| Feature | Reason template |
| --- | --- |
| score\_slope\_5 (negative) | Your scores on this topic dropped about {x} points over your last 5 attempts |
| days\_since\_last\_attempt (high) | You have not practised this topic for {n} days |
| prereq\_min\_mastery (low) | A prerequisite, {name}, is still weak at {m} percent |
| avg\_score\_last3 (low) | Your last 3 scores average {m} percent |
| hints\_rate (high) | You needed {h} hints per attempt on average |
| attempts\_count (low) | Only {n} attempts so far, so this estimate is less certain |

- Test that every feature in the model has a template, so no raw feature name ever reaches the UI.

### 6.8 Artifacts and versioning

- Save models/model\_v1.joblib (the model and calibrator), models/features.json (ordered feature list), models/metrics.json, and insert a row in model\_registry.
- The ML service refuses to start if features.json does not match the model. Every prediction response carries model\_version.
- Retraining produces model\_v2 and never overwrites v1.

### 6.9 Model card (docs/model-card.md)

Write one page covering: purpose, data (synthetic, with the generator rules), features, label, metrics, ablation result, limitations (synthetic data, one subject, no fairness testing on real groups), and intended use (support for teachers, never an automatic decision about a student).

## 7. Recommendation engine

### 7.1 Algorithm

Input: twin state, predictions, graph, resources. Output: an ordered learning path.

1. Targets: concepts with risk 0.5 or above, or mastery below 0.5.
2. For each target, run the root gap algorithm. The union of root gaps and targets forms the candidate set.
3. Score each candidate: priority = 0.4 x severity + 0.3 x impact + 0.2 x readiness - 0.1 x effort.
   - severity = 1 - mastery.
   - impact = number of at-risk descendants it unblocks divided by the largest such number among candidates.
   - readiness = fraction of its prerequisites with mastery 0.6 or above.
   - effort = estimated minutes of its best resource divided by 120, capped at 1.
4. Order: topological order first, so a prerequisite always comes before the concept that needs it. Break ties inside the same level by priority.
5. Resource choice: from the concept's resources, pick the one whose difficulty is closest to mastery x 5 rounded up, preferring a practice item when mastery is below 0.3 and a harder item when above 0.6.
6. Cap the path at 10 steps. Each step includes concept, resource, estimated minutes and a one line reason (for example: Arrays is blocking Linked Lists and Trees).
7. Regenerate the path whenever the twin changes, and mark the old one stale.

The weights above are starting values. Tune them using the simulation in 7.2 and record the final values.

### 7.2 Evaluation by simulation

Build a small simulator that reuses the generator's learning rule. Take 100 held out students with weak spots, give each one 10 study steps in a fixed order, and measure the result.

| Strategy | Study order |
| --- | --- |
| NeuroLearn | Output of the algorithm in 7.1 |
| Syllabus order | Concepts in curriculum order |
| Weakest first | Lowest mastery first, ignoring prerequisites |
| Random | Random weak concepts |

Metrics: mean mastery gain across at-risk concepts, number of at-risk concepts cleared after 10 steps. Run 20 seeds and report mean and standard deviation. State plainly that this is a simulation, not a classroom trial.

## 8. API specification

### 8.1 Conventions

- Base path /api/v1, JSON bodies, UTF-8, ISO 8601 timestamps in UTC.
- Authentication: Authorization: Bearer token. Tokens last 8 hours.
- Validation with zod on every body, query and path parameter.
- Pagination for lists: limit (default 50, max 200) and offset.
- Error shape on every failure:

```json
{ "error": { "code": "FORBIDDEN", "message": "You cannot access this student", "details": null } }
```

- Status codes: 200, 201, 400 (bad input), 401 (no or bad token), 403 (not allowed), 404, 409 (conflict), 422 (valid shape but invalid meaning), 429 (rate limited), 500, 503 (ML service down and no stored data).
- Authorisation: a student can only read or write their own data. Faculty can only read students in their own cohorts.

### 8.2 Public endpoints (frontend to Express)

| Method | Path | Role | Purpose |
| --- | --- | --- | --- |
| POST | /auth/register | public | Create account (faculty accounts are seeded, not self registered) |
| POST | /auth/login | public | Returns token and user |
| GET | /auth/me | any | Current user and role |
| GET | /concepts | any | All concepts |
| GET | /concepts/graph | any | Nodes and edges for the graph view |
| GET | /students/:id/twin | student, faculty | Current twin state |
| GET | /students/:id/twin/history | student, faculty | Snapshots for the timeline chart |
| GET | /students/:id/risks | student, faculty | Risk per concept with reasons and root gaps |
| GET | /students/:id/root-gaps | student, faculty | Ranked root gaps |
| GET | /students/:id/path | student, faculty | Active learning path |
| POST | /students/:id/path/regenerate | student | Force a new path |
| PATCH | /path-steps/:id | student | Mark a step complete |
| POST | /attempts | student | Log an attempt and trigger recompute |
| GET | /cohorts | faculty | Cohorts owned by the faculty user |
| GET | /cohorts/:id/summary | faculty | Counts per risk band, average mastery |
| GET | /cohorts/:id/students | faculty | Students sorted by risk, with filters |
| GET | /cohorts/:id/gaps | faculty | Common root gaps with student counts |
| GET | /cohorts/:id/alerts | faculty | Students whose risk rose 0.2 or more in 7 days |
| GET | /health | public | Liveness plus database and ML service status |

### 8.3 Sample response: GET /students/:id/risks

```json
{
  "student_id": "uuid",
  "model_version": "v1",
  "stale": false,
  "generated_at": "2026-10-12T10:00:00Z",
  "risks": [
    {
      "concept_id": 14,
      "concept_name": "Linked Lists",
      "risk": 0.82,
      "band": "high",
      "mastery": 0.41,
      "confidence": "high",
      "reasons": [
        "Your scores dropped about 18 points over your last 5 attempts",
        "A prerequisite, Arrays, is still weak at 38 percent",
        "You have not practised this topic for 12 days"
      ],
      "root_gaps": [ { "concept_id": 6, "name": "Arrays", "mastery": 0.38, "chain": [6, 14] } ]
    }
  ]
}
```

### 8.4 Internal endpoints (Express to ML service)

All internal calls send the header X-API-Key. The ML service never accepts calls from the browser.

| Method | Path | Input | Output |
| --- | --- | --- | --- |
| POST | /ml/twin/update | attempts for one student, as\_of | Twin state |
| POST | /ml/predict | twin state, attempts, concept ids | Risk per concept, band, model\_version |
| POST | /ml/explain | same as predict | Risk plus top 3 reasons per concept |
| POST | /ml/root-gap | mastery map, target concept ids | Root gaps and chains |
| POST | /ml/recommend | twin, predictions | Ordered path |
| GET | /ml/health | none | Status, model\_version, graph size |
| GET | /ml/model-info | none | Metrics and feature list |
| POST | /ml/analyze | attempts for one student, as\_of, optional concept ids, include\_path | Twin state, risk with top 3 reasons and root gaps per concept, optional path, model\_version |

Express timeout for ML calls: 5 seconds, one retry. After that, serve stored data and set stale to true.

Define request and response models once with Pydantic in the ML service and with zod in Express, and keep them identical. Export the OpenAPI document from FastAPI into docs/api.md so the contract is always current.

## 9. Frontend design

### 9.1 Pages and routes

| Route | Role | Content |
| --- | --- | --- |
| /login | public | Email and password form, error messages, redirect by role |
| /student | student | Summary cards, mastery heatmap, risk list, root gap card, path preview |
| /student/concept/:id | student | One concept: risk, reasons, score trend, prerequisites, resources |
| /student/path | student | Full learning path with checkable steps |
| /student/graph | student | Interactive prerequisite graph coloured by mastery |
| /student/timeline | student | Twin history chart (mastery and risk over time) |
| /faculty | faculty | Cohort summary, risk distribution, alerts, student table |
| /faculty/students/:id | faculty | Same views as the student pages, read only |
| /faculty/gaps | faculty | Common root gaps ranked by affected students |
| \* | any | Not found page |

### 9.2 Component list

- Layout: AppShell, NavBar, RoleGuard (redirects when the role does not match).
- Data display: SummaryCard, MasteryHeatmap, RiskList, RiskBadge, ConfidenceChip, ReasonList, RootGapCard, TrendChart, TwinTimeline, PrerequisiteGraph, PathStepper, ResourceCard, CohortTable, GapBarChart, AlertList.
- State components: Skeleton, EmptyState, ErrorState, StaleBanner (shown when the API returns stale true).
- Each component takes plain props and has no fetching inside, so you can test it with mock data. Pages do the fetching.

### 9.3 State and data fetching

- TanStack Query for all server data. Query keys follow the pattern \[resource, id\], for example \[risks, studentId\].
- After POST /attempts resolves (the backend has finished the recompute by then), cancel in-flight queries for that student, write the response into the cache with setQueryData, then invalidate the twin, risks, root gaps and path queries. Ignore any response whose computed\_at is older than the cached one (Section 17.10).
- An api/ folder holds one typed function per endpoint. Types are copied from the OpenAPI document so frontend and backend never drift.
- Store the token in sessionStorage and attach it in one fetch wrapper. On a 401, clear it and go to /login. This is a simple choice for a student project. In a real product you would use httpOnly cookies.

### 9.4 Visual design

- Mastery and risk colours: strong green, developing amber, weak red. Always pair colour with a text label or icon so colour blind users are not excluded.
- One typeface family, 4 sizes, an 8 px spacing grid, cards with consistent padding, and a maximum content width of 1200 px.
- Wireframe, student dashboard: row 1 has four summary cards (overall mastery, high risk count, current streak, next step). Row 2 has the heatmap on the left and the risk list on the right. Row 3 has the root gap card and the path preview.
- Wireframe, faculty dashboard: row 1 has risk distribution and alerts. Row 2 has the sortable student table. Row 3 has the common gaps chart.
- Dark mode is optional. Skip it until everything else is finished.

### 9.5 UX states every screen must handle

- Loading: skeletons, never a blank page.
- Empty: a new student sees a friendly message and a prompt to log a first attempt.
- Error: a readable message with a retry button.
- Stale: a banner saying predictions may be out of date.
- Low confidence: a chip next to any estimate based on fewer than 3 attempts.

### 9.6 Accessibility

- Everything reachable by keyboard, visible focus rings, labels on all inputs, alt text or aria labels on charts, text alternatives (a table view) for the heatmap and graph, and AA colour contrast.

## 10. Security and privacy

| Area | Requirement |
| --- | --- |
| Passwords | bcrypt with cost 12, minimum length 8, never logged or returned |
| Tokens | JWT HS256, 8 hour expiry, secret from environment, verify on every protected route |
| Authorisation | Middleware checks role, then checks ownership (student id matches token, or faculty owns the cohort). Test every route for a wrong user |
| Input | zod validation, parameterised SQL only (never string concatenation), body size limit 100 kb |
| Transport and headers | helmet, strict CORS allowlist, HTTPS in production |
| Abuse | Rate limit login to 5 attempts per minute per IP and the API to 100 requests per minute per user |
| Internal calls | ML service reachable only with X-API-Key and not exposed publicly in production |
| Secrets | Only in environment variables, .env in .gitignore, rotate if ever committed |
| Dependencies | npm audit and pip-audit in CI, fix high severity findings |
| Logging | Request id on every log line, never log passwords, tokens or full request bodies |
| Data | Only synthetic data. If you ever use real data, anonymise it and get consent first |

Responsible use rules for the product itself:

- Predictions are probabilities that support a teacher. They are never a final judgement about a student.
- Every score has reasons and a confidence marker.
- Show a short notice on the faculty dashboard that predictions come from a model trained on synthetic data and need human review.

## 11. Testing strategy

### 11.1 Test layers

| Layer | Tool | What it covers | Target |
| --- | --- | --- | --- |
| Unit, ML | pytest | Twin math, features, root gap, recommender scoring, reason templates | 85 percent of core logic |
| Unit, backend | Vitest | Validation schemas, auth helpers, cache logic | 70 percent |
| Integration, backend | Vitest and supertest, real test database | Every route: success, 401, 403, 404, validation error | Every route |
| Integration, ML | pytest and FastAPI TestClient | Every endpoint against the real model file | Every endpoint |
| Component | Vitest and Testing Library | Key components with mock data | Main components |
| End to end | Playwright | Login, view risks, log an attempt, see update, faculty view | 1 full demo path |

### 11.2 Specific cases you must cover

- Root gap: chain, diamond, two roots, no gap, target itself weak.
- Twin: bounds, forgetting, cold start.
- Features: no feature uses data from the future (write a test that adds a future attempt and checks the features do not change).
- Reasons: every model feature has a template.
- Auth: a student cannot read another student, faculty cannot read another faculty's cohort, expired token returns 401.
- Resilience: stop the ML service, then check the API returns stored predictions with stale true. Start it again and check recovery.
- Edge data: student with no attempts, student with perfect scores, a concept with one attempt, a very long history.
- Idempotency: sending the same attempt twice with the same client\_attempt\_id returns the original result and writes no second row.

### 11.3 Acceptance test script (run before every milestone)

1. Reset and seed: ./scripts/reset.sh then ./scripts/seed.sh.
2. Log in as demo student 1. Confirm the heatmap shows Arrays weak, Linked Lists and Trees at high risk, and the root gap card names Arrays.
3. Open Linked Lists. Confirm three readable reasons.
4. Log three good Arrays attempts. Confirm mastery rises, risk falls and the path changes within 3 seconds.
5. Log in as faculty. Confirm demo student 1 appears near the top, and the common gaps chart lists Arrays.
6. Stop the ML service. Reload. Confirm the stale banner and that pages still load.

### 11.4 Performance check

Run autocannon or k6 for 30 seconds against GET /students/:id/risks with 20 concurrent users on the cached path. Record the p95 latency in the docs.

## 12. DevOps and delivery

### 12.1 Local environment

- docker-compose.yml with four services: db (postgres), ml-service, backend, frontend. Each has a healthcheck, and backend waits for db and ml-service to be healthy.
- One command to start: docker compose up --build. Two commands to reset the demo: ./scripts/reset.sh and ./scripts/seed.sh.
- Document the exact commands in the README and test them on a clean clone.

### 12.2 Git workflow

- main is always working. Create short lived branches per task (feature/twin-engine, fix/cors). Merge through pull requests, even when you review your own, because CI runs on them.
- Commit messages: type and short summary, for example feat: add root gap algorithm.
- Tag milestones (v0.1 to v1.0) so you can always return to a working demo.

### 12.3 Continuous integration (GitHub Actions)

- Job 1: validate graph files (cycles, unknown ids, missing resources).
- Job 2: ML service, install, lint, pytest.
- Job 3: backend, install, lint, type check, tests against a Postgres service container.
- Job 4: frontend, install, lint, type check, tests, build.
- Job 5: dependency audit.
- Rule: do not merge with a red pipeline.

### 12.4 Deployment

| Part | Suggested host | Notes |
| --- | --- | --- |
| Frontend | Cloudflare Pages or Vercel | Set VITE\_API\_URL at build time, always as an https:// URL |
| Backend | Render or Railway | Run migrations on start, set all environment variables |
| ML service | Render or Railway | Private if the host supports it, otherwise protected by API key |
| Database | Neon or Supabase | Run migrations then the seed script once |

Free tiers often sleep after inactivity. Before any demo, open every service URL and the health endpoints 10 minutes ahead to warm them up.

### 12.5 Observability and backups

- Structured logs with pino and Python logging, a request id passed from Express to the ML service.
- /health on backend and /ml/health report database, ML service and model version.
- Keep a pg\_dump of the seeded demo database in the repo or storage, so you can restore the demo in minutes if a deploy breaks.

## 13. Project plan

### 13.1 Schedule and milestones

| Phase | What | Duration |
| --- | --- | --- |
| 0 | Setup and contract | 2 to 3 days |
| 1 | Knowledge graph and content | 3 to 4 days |
| 2 | Database | 2 days |
| 3 | Data generation, twin and model | 8 to 10 days |
| 4 | ML service | 3 days |
| 5 | Backend | 7 to 8 days |
| 6 | Recommender and simulation | 4 to 5 days |
| 7 | Student frontend | 7 to 8 days |
| 8 | Faculty frontend | 4 days |
| 9 | Integration and hardening | 5 days |
| 10 | Deployment | 3 days |
| 11 | Documentation and pitch | 4 to 5 days |

Total is about 7 weeks of work. Add 20 percent buffer, because solo projects always run over. If you have more time, spend it on Phase 3 and Phase 6 quality, not extra features.

| Milestone | Target | Proof |
| --- | --- | --- |
| M1 | End of week 1 | Graph validated, database migrated and seeded |
| M2 | End of week 3 | Model meets metrics, ML service passes tests |
| M3 | End of week 4 | Backend complete with tests and fallback |
| M4 | End of week 5 | Student dashboard and recommender working end to end |
| M5 | End of week 6 | Faculty dashboard done, acceptance script passes, feature freeze |
| M6 | End of week 7 | Deployed, documents done, pitch rehearsed |

### 13.2 Daily working agreement

- Start the day by picking 1 to 3 tasks from the board. End it by committing, pushing and writing 3 lines in a build log (done, blocked, next).
- Do one thing at a time. Finish and test it before the next.
- Build the interface against mock data until the real endpoint exists.
- If a task takes more than twice its estimate, stop and cut scope for that task.

### Phase 0: Setup and contract

Goal: a clean repo and one agreed contract, so nothing is invented later.

1. Install Node LTS, Python 3.11, Docker Desktop and Git. Create the repo: git init neurolearn-ai, add .gitignore (node\_modules, **pycache**, .env, .venv, dist).
2. Create the folders from Section 2.5 and an empty README.
3. Copy Sections 1 to 4 of this document into docs/prd.md and docs/architecture.md.
4. Create docs/api.md from Section 8 and keep it as the source of truth.
5. Write docker-compose.yml with only the db service. Run docker compose up db and connect with psql to confirm.
6. Create a task board (GitHub Projects) with one card per task in this plan.
7. Create .env.example from Section 2.6. Then add .github/workflows/ci.yml with one job that validates the graph files (skeleton in Section 17.1), and add one job each time a service gets its first tests.

DoD: a fresh clone starts the database with one command, and the contract is committed.

### Phase 1: Knowledge graph and content

Goal: a validated graph with resources, the foundation of every later phase.

1. List 38 concepts in a spreadsheet: code, name, level, difficulty, description.
2. Define 1 to 3 prerequisites per concept. Sketch the graph on paper first and check that every level 2 or higher concept has a sensible parent.
3. Collect at least 2 resources per concept (official docs, free video, practice problems). Record url, type, difficulty, minutes. Open every link to check it works.
4. Export to data/concepts.json, data/prerequisites.json, data/resources.json. Then write scripts/build\_graph.py to validate them and produce data/graph.json (Section 17.7), and run it in the ML Dockerfile and in CI so the ML service always has its graph file.
5. Write scripts/validate\_graph.py using networkx: is a DAG, no unknown ids, at most 3 prerequisites, every concept has resources, levels are consistent (a prerequisite has a lower level).
6. Implement ml-service/app/services/graph.py with ancestors, descendants, topological order and the root gap algorithm from 4.4.
7. Write the 5 root gap tests from 4.4.

DoD: validation passes, root gap tests pass, the graph is committed and tagged v0.1.

### Phase 2: Database

Goal: a reproducible schema and seed.

1. Write db/migrations/001\_init.sql from Section 3: enums, tables, foreign keys, checks.
2. Write 002\_indexes.sql.
3. Write a small migration runner (or use node-pg-migrate) that records applied files in a schema\_migrations table.
4. Write db/seed that loads concepts, prerequisites, resources, one faculty user, three cohorts, the demo student accounts and the generated students. Hash the passwords of the 5 demo accounts and the faculty account with bcrypt. Give every generated student one shared pre-computed hash, since they never log in, so seeding 300 users takes seconds (Section 17.4).
5. Write scripts/reset.sh (drop, migrate) and scripts/seed.sh.
6. Test: reset and seed from nothing, then run a few sanity queries (row counts, no orphan rows).

DoD: reset plus seed finishes in under a minute on a clean machine.

### Phase 3: Data generation, twin and model

Goal: a trained, explained, evaluated model. This is the heart of the project, so take your time.

1. Implement generate\_data.py exactly as in 6.1, with seed 42. Add the 5 scripted demo students.
2. Sanity check the data: plot score distribution by level, attempts per student, and the share of label 1 rows. Aim for 15 to 35 percent at-risk. If it is outside that range, adjust the generator parameters.
3. Implement twin.py (5.2) and its tests (5.3).
4. Implement features.py (6.2) and the future data leakage test.
5. Train the logistic regression baseline and record its metrics.
6. Train and tune XGBoost with GroupKFold, early stopping and class weighting. Calibrate and pick the threshold.
7. Run evaluate.py (6.6) including the ablation and root gap accuracy. Save plots and metrics.json.
8. Implement reasons.py with the templates (6.7) and the test that every feature has one.
9. Write docs/model-card.md (6.9).

Sanity rules: if AUC is above 0.97 suspect leakage or a generator that is too easy, so add noise and recheck. If AUC is below 0.70 the generator signal is weak or a feature has a bug, so inspect feature distributions before tuning.

DoD: model\_v1 saved and registered, metrics meet Section 1.9 targets or the gap is explained in the model card, explain(student, concept) returns a risk and 3 readable reasons for the demo students.

### Phase 4: ML service

Goal: the model and logic behind a stable HTTP interface.

1. Create the FastAPI app (app/main.py) with a lifespan function that loads the model, calibrator, features.json and the graph once. Refuse to start on a feature mismatch.
2. Define Pydantic models for every request and response (Section 8.4).
3. Add routers for analyze (the single call Express uses for twin, risk, reasons, root gaps and path), twin, predict, explain, root-gap, recommend, health and model-info.
4. Add the X-API-Key dependency, request id middleware and structured error responses.
5. Write pytest tests using TestClient, with the real model file and the 5 demo students.
6. Write the Dockerfile (slim Python image, non-root user) and add the service to docker-compose.
7. Measure latency of /ml/explain for one student with 38 concepts. Target under 300 ms.

DoD: all endpoints pass tests, Swagger UI works, the service starts in Docker and reports healthy.

### Phase 5: Backend

Goal: the secure orchestration layer.

1. npm init, add TypeScript, Express, pg, zod, jsonwebtoken, bcrypt, helmet, cors, express-rate-limit, pino, Vitest, supertest. Create the folder layout from 2.5.
2. config.ts: read and validate environment variables with zod, exit on a missing one.
3. db/pool.ts with a connection pool, a query helper and a transaction helper.
4. Auth: register, login, me, plus middleware for authentication, role and ownership checks. Add rate limits.
5. Read routes: concepts, graph, twin, twin history, risks, root gaps, path.
6. mlClient.ts: calls with the API key, a 5 second timeout and one retry. Wrap errors into a typed result.
7. Cache and fallback logic (Flow B): fresh predictions come from the database, stale ones are recomputed, ML failure returns stored data with stale true.
8. POST /attempts (Flow A) in a transaction, requiring a client\_attempt\_id and replaying the stored result on a duplicate (Section 17.5).
9. Make predictions append only and read the latest per student and concept with DISTINCT ON. Faculty alerts and the timeline need this history.
10. Write scripts/backfill\_history in Python that imports the ML code directly instead of calling it over HTTP, computes weekly twin snapshots and predictions in batches, loads them with COPY and stores explanations only for the latest week (Section 17.2). Run it once and save the result as a pg\_dump.
11. Cohort routes (summary, students, gaps, alerts) using SQL aggregation.
12. Central error handler, request id, pino logging.
13. Integration tests for every route: success, 401, 403, 404, validation failure. Add the ML down test.
14. Generate the API document from the code and keep it in sync with docs/api.md.

DoD: all routes tested, the fallback test passes, the backend and ML service run together in compose.

### Phase 6: Recommender and simulation

Goal: personalised paths that you can prove are better than simple baselines.

1. Implement recommender.py following 7.1 with unit tests (prerequisite always before dependent, path capped at 10, reasons present, no resource for a concept missing).
2. Expose /ml/recommend and call it from the backend. Store paths in recommendations and steps in path\_progress. Add PATCH /path-steps/:id.
3. Write simulator.py (7.2) with the four strategies and 20 seeds.
4. Tune the weights on a tuning set of students, then report on a separate evaluation set.
5. Save the results table to docs/ and keep the final weights in config.

DoD: results table exists. If NeuroLearn does not beat the syllabus baseline, report it honestly and investigate why before the final submission.

### Phase 7: Student frontend

Goal: US-1 to US-6 working against the real API.

1. npm create vite@latest frontend with the react-ts template. Add Tailwind, React Router, TanStack Query, Chart.js and Cytoscape.js.
2. Create the typed api client, the auth context, RoleGuard and the login page.
3. Build components from 9.2 on a hidden /dev page with mock data first.
4. Build /student (dashboard), /student/concept/:id, /student/path, /student/graph and /student/timeline.
5. Add the attempt logging dialog (concept, score, time, hints) and wire it to POST /attempts, then invalidate the queries.
6. Add all five UX states from 9.5 to every page.
7. Run Lighthouse and an accessibility pass: keyboard, contrast, labels.

DoD: acceptance criteria of US-1 to US-6 pass by hand, and the demo student 1 story is clear on screen.

### Phase 8: Faculty frontend

Goal: US-7 to US-9.

1. Build the cohort summary with a risk distribution chart and the alert list.
2. Build the sortable and filterable student table (risk, trend arrow, top root gap).
3. Reuse the student components inside /faculty/students/:id in read only mode.
4. Build /faculty/gaps with the ranked bar chart.
5. Add the model and synthetic data notice from Section 10.

DoD: acceptance criteria of US-7 to US-9 pass, and faculty cannot open another faculty's cohort.

### Phase 9: Integration and hardening

Goal: boring reliability.

1. Run the acceptance script in 11.3 from a clean reset. Fix every failure and run it again.
2. Write the Playwright end to end test for the demo path.
3. Cover the edge cases listed in 11.2.
4. Run the load check in 11.4 and record the numbers.
5. Walk through the security table in Section 10 line by line and tick each item.
6. Fix the top 10 bugs, then declare feature freeze.
7. Make CI green on main.

DoD: the acceptance script passes 3 times in a row, and the CI pipeline is green.

### Phase 10: Deployment

1. Create the production database, run migrations, then restore the pre-built seeded dump with pg\_restore. Do not run the backfill on a free tier host.
2. Deploy the ML service, then the backend with all environment variables, then the frontend with VITE\_API\_URL.
3. Set CORS\_ORIGIN to the exact https:// frontend origin with no trailing slash and verify login from the live site.
4. Run the acceptance script against production.
5. Save a pg\_dump of the seeded production data and note how to restore it.
6. Write down all URLs and credentials for demo accounts in a private note (never in the repo).

DoD: the full demo path works on the public URL.

### Phase 11: Documentation and pitch

1. README: what it is, architecture diagram, setup in 5 steps, demo accounts, test commands, deployment notes.
2. Final docs: prd.md, api.md, architecture.md, model-card.md, simulation results.
3. Build the slides (outline in Section 15) and the 3 minute demo script.
4. Record a backup demo video of the full flow.
5. Rehearse twice with a timer, once with the deliberately broken ML service to practise the stale banner explanation.
6. Complete the mentor-facing submission documents and fill every open placeholder: mentor details, team members as confirmed, and the budget breakdown.

DoD: everything is submitted, and the repo can be cloned and run by a stranger using only the README.

## 14. Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Synthetic data makes the model look unrealistically good | Credibility loss | Say clearly it is synthetic, add noise, show the ablation and the limits in the model card |
| Data leakage in features | Inflated metrics | Time based split, GroupKFold, the future data test |
| Scope creep | Missed deadline | Non-goals in Section 1.4, feature freeze at end of Phase 9 |
| ML service slow or down during demo | Failed demo | Cache, stored predictions fallback, warm up services, backup video |
| Free tier cold starts | Slow first request | Warm up 10 minutes before, keep a local copy running too |
| Root gap logic wrong on edge graphs | Wrong advice | Five required tests, ground truth accuracy measurement |
| Integration surprises late | Rework | Contract first, mock data, integrate every week not at the end |
| Burnout or lost days | Schedule slip | 20 percent buffer, one rest day each week, cut scope not sleep |
| Lost work or broken deploy | Lost days | Daily commits, tags, database dump |

## 15. Demo and submission

### 15.1 Three minute demo script

| Time | Action | Message |
| --- | --- | --- |
| 0:00 | Slide: the problem | Students and teachers find gaps too late |
| 0:20 | Student dashboard for demo student 1 | The twin shows Linked Lists and Trees at high risk |
| 0:50 | Open Linked Lists | Three plain reasons from SHAP, not a black box |
| 1:10 | Root gap card | The real cause is Arrays, found through the knowledge graph |
| 1:30 | Learning path | Ordered steps to fix the root cause first |
| 1:50 | Log three good Arrays attempts | Twin, risk and path update live |
| 2:15 | Faculty dashboard | At-risk students and common gaps, so teachers act early |
| 2:40 | Results slide | AUC, recall, ablation, simulated path gain |
| 2:55 | Close | Proactive, explainable, personalised |

### 15.2 Slide outline (10 slides)

1. Title. 2. Problem. 3. Solution overview. 4. Architecture diagram. 5. Digital Learning Twin. 6. Prediction and explanation (SHAP). 7. Knowledge graph and root gaps. 8. Recommendation and simulation results. 9. Live demo. 10. Impact, limits and next steps.

### 15.3 Submission checklist

- Working deployed link and a backup video.
- Repository with a clear README.
- PRD, architecture document, API document, model card, simulation results.
- Mentor-facing documents complete, with mentor details, team members and the budget breakdown filled in.
- Honest statement of limits: synthetic data, one subject, no real classroom trial.

## 16. Final definition of done

- All 10 user stories pass their acceptance criteria.
- All metrics in 1.9 are reported, whether or not each target was met.
- Every route has an integration test, core logic has unit tests, CI is green.
- The demo path works 3 times in a row on the deployed site.
- A stranger can set up the project from the README in under 30 minutes.
- Nothing secret is in the repository.

## 17. Amendments in version 1.1: gaps found in review

Ten gaps were raised against version 1.0. All ten are valid as design gaps, though a few were smaller than first stated. The affected sentences in Sections 2, 5, 6, 8, 9, 11, 12 and 13 were edited in place. This section holds the full specs.

| Gap | Verdict | Fixed in |
| --- | --- | --- |
| G1 No ci.yml | Valid, Section 12.3 described jobs but no step created the file | 17.1, Phase 0 |
| G2 Backfill too slow | Valid. 300 students x 12 weeks is 3,600 student-weeks, tens of minutes over HTTP and far worse on a free tier | 17.2, Phases 5 and 10 |
| G3 Engagement leakage | Valid, a cohort median over the whole dataset uses future data | 17.3, Section 5.2 |
| G4 Password pipeline | Valid, and bcrypt cost 12 for 300 users also breaks the one minute reset target | 17.4, Phase 2 |
| G5 No idempotency key | Valid. The twin is rebuilt from history, so a retry double counts an attempt rather than corrupting stored state | 17.5, Section 3, Flow A |
| G6 Serial ML calls | Valid as redundancy. Summed latency was within 3 seconds on paper, but three calls add failure points and cold start cost | 17.6, Section 8.4 |
| G7 graph.json never built | Valid, the ML service would fail at startup | 17.7, Phase 1 |
| G8 Consistency division by zero | Valid for fewer than 3 attempts or same day attempts | 17.8, Section 5.2 |
| G9 Mixed HTTP and HTTPS | Valid as a configuration risk. The browser only calls the backend, so the ML service is not affected | 17.9, Phase 10 |
| G10 Refetch before ML finishes | Valid in edge cases. POST /attempts is synchronous, so the race comes from background refetches and late responses | 17.10, Section 9.3 |

### 17.1 CI workflow skeleton (.github/workflows/ci.yml)

Create this file in Phase 0 with only the graph job. Add each other job when its folder gets tests. Commit the small model\_v1.joblib so the ML tests run against the real model.

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
jobs:
  graph:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install networkx
      - run: python scripts/validate_graph.py
      - run: python scripts/build_graph.py --check
  ml-service:   # add after Phase 3
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11', cache: pip }
      - run: pip install -r ml-service/requirements.txt
      - run: pytest ml-service/tests -q
  backend:      # add after Phase 5
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: neurolearn_test }
        ports: ['5432:5432']
        options: >-
          --health-cmd 'pg_isready -U postgres' --health-interval 5s --health-retries 10
    env:
      DATABASE_URL: postgres://postgres:test@localhost:5432/neurolearn_test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: lts/*, cache: npm, cache-dependency-path: backend/package-lock.json }
      - run: npm ci && npm run lint && npm run typecheck && npm test
        working-directory: backend
  frontend:     # add after Phase 7: same shape as backend, ending with npm run build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: lts/*, cache: npm, cache-dependency-path: frontend/package-lock.json }
      - run: npm ci && npm run lint && npm run typecheck && npm test && npm run build
        working-directory: frontend
```

Add a fifth job with npm audit and pip-audit in Phase 9. Turn on branch protection for main so a red pipeline blocks the merge.

### 17.2 Backfill design

- Do not call the ML service over HTTP. Write scripts/backfill\_history.py inside ml-service so it imports twin.py, features.py and the model directly.
- Build one feature matrix for all student, concept and week rows (300 x 38 x 12 is about 137,000 rows), then call model.predict\_proba once per batch. This takes seconds, not minutes.
- Compute SHAP and reasons only for the latest week and for students whose risk rose 0.2 or more, since those are the only rows the UI shows reasons for.
- Load the output with COPY into twin\_snapshots and predictions, not row by row inserts.
- Target: the whole backfill in under 5 minutes locally.
- Run it once, save a pg\_dump (db/seed/demo.dump), and restore that dump in every environment. Production never runs the backfill.

### 17.3 Time windowed cohort statistics

- Any statistic that uses other students (the engagement median) is computed per prediction time t from attempts strictly before t, over the window from t minus 7 days to t.
- Precompute these as a small table keyed by cohort and week boundary, built with the same before-t rule.
- Extend the leakage test in 11.2: append future attempts for the student and for other students in the cohort, then assert that every feature, including the cohort based ones, is unchanged.

### 17.4 Seed data and passwords

- generate\_data.py writes no passwords. students.csv carries id, name, email, cohort and is\_demo only.
- The seed script creates one user row per student. The 5 demo students and the faculty user get real passwords from environment variables, hashed with bcrypt cost 12.
- All other generated students get one shared pre-computed hash that nobody knows the password for. They never log in, so this keeps seeding fast.
- Generated emails use a reserved test domain (student001@demo.neurolearn.test).
- Add a seed test that counts users, students and attempts and checks no student lacks a user row.

### 17.5 Idempotency for POST /attempts

- The request body requires client\_attempt\_id (uuid). The frontend creates it with crypto.randomUUID() when the attempt dialog opens, so a retry or double click reuses it.
- The attempts table has a unique index on (student\_id, client\_attempt\_id). The insert uses ON CONFLICT DO NOTHING RETURNING id.
- On a conflict, skip the recompute, return the current twin summary and top risks with status 200 and the header Idempotent-Replay: true. A new attempt returns 201.
- Tests: same id twice gives one row and identical responses; two different ids give two rows.

### 17.6 One analyze call

- POST /ml/analyze takes the student's attempts, as\_of, optional concept ids and include\_path (default true). It returns the twin state, risk with band and top 3 reasons per concept, root gaps, the optional path and model\_version.
- Express uses it in Flow A and Flow B. The separate /ml/predict, /ml/explain, /ml/twin/update and /ml/recommend stay for tests and debugging only.
- Latency budget: analyze under 600 ms p95 on localhost. POST /attempts under 1.5 s p95 locally and under 3 s against warm free tier hosts.
- Log the timing of each stage (twin, features, predict, SHAP, recommend) in the response so you can see where time goes.

### 17.7 graph.json

- scripts/build\_graph.py reads concepts.json, prerequisites.json and resources.json, runs every check from validate\_graph.py, then writes data/graph.json with nodes, edges, topological order, and the ancestors and descendants of every concept.
- The flag --check rebuilds in memory and exits with an error if graph.json differs from the file on disk. CI runs it.
- The ML Dockerfile copies graph.json, and features.json stores a hash of it. The ML service refuses to start if the hash does not match, the same way it refuses on a feature mismatch.

### 17.8 Rules for thin data

| Value | Rule when data is thin |
| --- | --- |
| consistency | Fewer than 3 attempts: 0.5 and flagged low confidence. Mean gap floored at 0.5 days |
| velocity | Fewer than 3 attempts: 0 |
| score\_slope\_5 | Fewer than 3 scores: missing for XGBoost, imputed 0 for logistic regression |
| avg\_score\_last3, avg\_score\_all | No attempts: the concept prior |
| time\_ratio | No attempts: 1.0, typical time floored at 1 second |
| hints\_rate | No attempts: 0 |
| engagement | Cohort median floored at 1 |
| days\_since\_last\_attempt | No attempts: 60, with attempts\_count 0 |

Add one unit test per row, plus a test that every feature is finite for a student with no attempts.

### 17.9 HTTPS and CORS rules

- Every public URL is https. A page served over https cannot call an http API, because the browser blocks it as mixed content. VITE\_API\_URL must start with https:// in production.
- CORS\_ORIGIN is the exact origin, scheme and host, with no trailing slash.
- The browser never calls the ML service, so its URL only needs to work from the backend. Prefer the host's private network, otherwise use https plus the API key.
- In production the backend exits at startup if CORS\_ORIGIN is not https. Add scripts/smoke.sh, which runs curl -fsS against /health of the frontend, backend and ML service, and run it before every demo.

### 17.10 Frontend cache after an attempt

- POST /attempts only returns after the twin, predictions and path are stored, so the mutation's onSuccess runs on finished data.
- In onMutate call cancelQueries for that student, so an earlier refetch cannot land after the new data.
- In onSuccess write the response with setQueryData, then invalidate twin, risks, root gaps and path. Set refetchOnWindowFocus to false for these keys while the mutation is pending.
- Every response carries computed\_at. The fetch wrapper drops a response older than the cached one.
- Show a small Updating state on the cards while the mutation runs, so the user never sees silent stale numbers.
