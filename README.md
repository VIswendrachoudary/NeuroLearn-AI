# NeuroLearn AI

> **An Intelligent Learning Operating System Using Machine Learning and Knowledge Graphs**

[![Python](https://img.shields.io/badge/Python-3.11-blue.svg)](https://www.python.org/)
[![Node](https://img.shields.io/badge/Node-LTS-green.svg)](https://nodejs.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue.svg)](https://www.postgresql.org/)
[![License](https://img.shields.io/badge/License-MIT-purple.svg)](LICENSE)
[![CI](https://github.com/VIswendrachoudary/NeuroLearn-AI/actions/workflows/ci.yml/badge.svg)](https://github.com/VIswendrachoudary/NeuroLearn-AI/actions)

NeuroLearn AI predicts which concepts a student is about to struggle with, explains why in plain language, traces the root cause back to the prerequisite that is really missing, and generates a personalised learning path to fix it. Faculty get a cohort view that flags at-risk students early so they can intervene before grades drop.

---

## Quick start (local)

```bash
# 1. Clone and enter the project
git clone https://github.com/VIswendrachoudary/NeuroLearn-AI.git
cd NeuroLearn-AI

# 2. Set up environment
cp .env.example .env          # then fill in JWT_SECRET and ML_API_KEY

# 3. Start the database
docker compose up -d db

# 4. Verify the database is running
docker compose exec db psql -U neurolearn -c 'select version();'
```

Full setup instructions (ML service, backend, frontend) will be added after each phase is complete.

---

## Repository layout

```
neurolearn-ai/
  README.md
  docker-compose.yml
  .env.example
  docs/            prd.md, api.md, architecture.md, model-card.md
  data/            concepts.json, prerequisites.json, resources.json, generated/
  db/              migrations/, seed/
  ml-service/      app/, training/, models/, tests/
  backend/         src/, tests/
  frontend/        src/, tests/
  scripts/         seed.sh, reset.sh
```

---

## Demo accounts

> Available after Phase 2 seeding. Documented here before v0.2 tag.

| Role | Email | Password |
|------|-------|----------|
| Student (main demo) | demo1@neurolearn.ai | `demo` |
| Faculty | faculty@neurolearn.ai | `demo` |

---

## Team

| Name | Reg. No. |
|------|----------|
| NARNE ADITHYA HANUMAN | 24BDS0220 |
| TANGUDU RAAJIV | 24BDS0242 |
| GARIKAPATI CHARAN DEEP | 24BDS0260 |
| BHAMIDIPATY SAI SANTOSH | 24BDS0213 |

---

## Build status

| Phase | Description | Tag |
|-------|-------------|-----|
| ✅ Phase 0 | Setup, repo structure, DB compose, env, CI skeleton | `v0.0` |
| ⬜ Phase 1 | Knowledge graph + content | `v0.1` |
| ⬜ Phase 2 | Database migrations + seed | `v0.2` |
| ⬜ Phase 3 | Data generation, Digital Twin, ML model | `v0.3` |
| ⬜ Phase 4 | ML FastAPI service | `v0.4` |
| ⬜ Phase 5 | Express backend | `v0.5` |
| ⬜ Phase 6 | Recommender + simulation | `v0.6` |
| ⬜ Phase 7 | Student dashboard | `v0.7` |
| ⬜ Phase 8 | Faculty dashboard | `v0.8` |
| ⬜ Phase 9 | Integration & hardening | `v0.9` |
| ⬜ Phase 10 | Deployment | `v1.0-rc` |
| ⬜ Phase 11 | Docs & pitch | `v1.0` |
