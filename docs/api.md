# NeuroLearn AI – API Contract

> **Version:** 0.0 (Phase 0 stub – updated after each phase)
> **Base URL (local):** `http://localhost:3001`
> **Auth:** All protected routes require `Authorization: Bearer <jwt>` unless noted.
> **ML service:** `http://localhost:8000` (internal only, secured with `X-API-Key`).

---

## Conventions

| Field | Value |
|-------|-------|
| Content-Type | `application/json` |
| Error shape | `{ "error": "<message>", "code": "<SNAKE_CASE>" }` |
| Timestamps | ISO-8601, UTC |
| IDs | UUID v4 |

---

## Auth Routes

### `POST /auth/login`
**Role:** public
**Request:**
```json
{ "email": "student@example.com", "password": "secret" }
```
**200 Success:**
```json
{ "token": "<jwt>", "user": { "id": "uuid", "role": "student", "studentId": "uuid" } }
```
**Error cases:**
- `401` – wrong email or password (same message for both)
- `429` – rate limited (5 per minute per IP)

---

### `POST /auth/register`
**Role:** public (student self-registration)
**Request:**
```json
{ "email": "student@example.com", "password": "secret", "name": "Ada Lovelace" }
```
**201 Success:** same shape as login 200.
**Error cases:** `400` bad input, `409` email already in use.

---

### `GET /auth/me`
**Role:** any authenticated
**200 Success:**
```json
{ "id": "uuid", "email": "...", "role": "student", "studentId": "uuid" }
```

---

## Concepts

### `GET /concepts`
**Role:** any authenticated
**200 Success:** array of `{ id, code, name, level, difficulty, description }`.

### `GET /concepts/graph`
**Role:** any authenticated
**200 Success:** `{ nodes: [...], edges: [{ from, to }] }`.

---

## Student Routes (all require `canAccessStudent`)

### `GET /students/:id/twin`
Latest digital twin snapshot.
**200 Success:** `{ studentId, computedAt, masteryAll: { [conceptId]: 0-1 }, velocity, consistency, engagement, confidence }`.
**Errors:** `401`, `403`, `404`.

### `GET /students/:id/twin/history`
**Query:** `?weeks=12`
**200 Success:** array of twin snapshots ordered oldest-first.

### `GET /students/:id/risks`
Latest prediction per concept (DISTINCT ON).
**200 Success:**
```json
[{ "conceptId": "uuid", "risk": 0-1, "band": "low|medium|high", "reasons": [...], "stale": false, "computedAt": "..." }]
```

### `GET /students/:id/root-gaps`
**200 Success:** `[{ "conceptId": "uuid", "chain": ["uuid", "uuid"] }]`.

### `GET /students/:id/path`
**200 Success:** `{ steps: [{ id, conceptId, resourceId, minutes, reason, done }] }`.

### `POST /attempts`
**Role:** student (own data only)
**Request:**
```json
{
  "clientAttemptId": "uuid",
  "studentId": "uuid",
  "conceptId": "uuid",
  "score": 0.85,
  "timeSpentSeconds": 120,
  "hintsUsed": 2,
  "assessmentType": "practice|quiz|assessment"
}
```
**201 Created:** `{ summary: <twin snapshot>, computedAt }`.
**200 + `Idempotent-Replay: true`** – duplicate `clientAttemptId`.
**Errors:** `400`, `401`, `403`.

### `POST /students/:id/path/regenerate`
**Role:** student (own) or faculty
**201 Success:** new path object.

### `PATCH /path-steps/:stepId`
**Request:** `{ "done": true }`
**200 Success:** updated full path.

---

## Faculty Routes

### `GET /cohorts`
**Role:** faculty
**200 Success:** `[{ id, name, studentCount }]`.

### `GET /cohorts/:id/summary`
**200 Success:** `{ bandCounts: { low, medium, high }, averageMastery }`.

### `GET /cohorts/:id/students`
**200 Success:** `[{ studentId, name, riskScore, trend, topRootGap }]`.

### `GET /cohorts/:id/alerts`
**200 Success:** `[{ studentId, conceptId, riskDelta, detectedAt }]` – rises ≥ 0.2 in 7 days.

### `GET /cohorts/:id/gaps`
**200 Success:** `[{ conceptId, studentCount }]` – ordered by student count desc.

---

## ML Service (internal)

### `GET /ml/health`
**Auth:** none
**200 Success:** `{ status: "ok", modelVersion: "v1", graphHash: "..." }`.

### `POST /ml/analyze`
**Auth:** `X-API-Key`
**Request:** `{ studentId, attempts: [...], cohortMedianAttempts, includePath: bool }`
**200 Success:** `{ predictions, twinSummary, rootGaps, path?, timings }`.

### `GET /ml/model-info`
**Auth:** `X-API-Key`
**200 Success:** contents of `models/features.json` merged with `models/metrics.json`.

---

*This file is the contract between frontend, backend and ML service. Update it before implementing each route, and update it again from the real OpenAPI output before tagging each phase.*
