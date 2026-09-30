# `Sift`

> **Positioning:** `<one line: what it does, for whom>`

**Name rationale:** `<2–4 sentences>`

*(Skeleton per BID requirement G2. Replace every `<placeholder>`; state limitations honestly.)*

---

## 1. Features

| Class | Feature | Status |
|---|---|---|
| Core | F1 CV Screening & Ranking (with hard-requirement gate) | ☐ |
| Core | F2 AI Candidate Scoring (0–100, dynamic weights) | ☐ |
| Core | F3 Video Interview (WebRTC, scheduling + links) | ☐ |
| Core | F4 Real-Time Transcription (dual-channel, Google STT) | ☐ |
| Core | F5 Q&A Analysis (RAG-verified) | ☐ |
| Core | F6 Consolidated Assessment | ☐ |
| Stretch | F7 / F8 / F9 / F10 — `<list attempted>` | ☐ |
| **Innovation** | F11 — `<feature name>` (see `docs/innovation_feature_rationale.md`) | ☐ |

## 2. Architecture

See `docs/architecture_diagram.md` and `docs/architecture_diagram.png`.

- **Backend:** FastAPI (ASGI), Pydantic, SQLAlchemy async
- **Agents:** LangGraph (`cv_screening_graph`, `qa_analysis_graph`, `assessment_graph`) with `AsyncPostgresSaver`
- **Data:** PostgreSQL, ChromaDB
- **AI services:** OpenAI (LLM, embeddings), Google Cloud Speech-to-Text (gRPC streaming)
- **Auth:** Firebase (role separation only)
- **Video:** WebRTC with FastAPI WebSocket signalling

## 3. Technology Stack & Justification

| Layer | Choice | Why |
|---|---|---|
| Backend | Python 3.11, FastAPI | `<justify>` |
| Orchestration | LangGraph | Stateful multi-step graphs with checkpointing |
| Frontend | `<framework>` | `<justify — required by BID §8.1>` |
| DB | PostgreSQL | App data + LangGraph checkpoints |
| Vector store | ChromaDB | Semantic matching + RAG |
| Container | Docker Compose | Single-command startup |

## 4. Prerequisites

- Docker Engine 24+ and Docker Compose v2
- OpenAI API key; Google Cloud project with Speech-to-Text enabled and a service-account JSON; Firebase project
- `<ports free: 3000, 8000>`

## 5. Setup & Run

```bash
git clone <repo-url> && cd ai-recruitment-platform
cp .env.example .env            # fill in real values (never commit .env)
mkdir -p secrets                # place GCP / Firebase JSON here (git-ignored)
docker compose up --build       # single-command startup
```

| Action | Command |
|---|---|
| Build | `docker compose build` |
| Run | `docker compose up` |
| Stop | `docker compose down` |
| Full reset (deletes data) | `docker compose down -v` |
| Logs | `docker compose logs -f backend` |

- Frontend: http://localhost:3000
- API docs (OpenAPI): http://localhost:8000/docs

## 6. Configuration Reference

See `.env.example`; document each variable here in a table (`Variable | Purpose | Required`).

## 7. Usage Walkthrough

1. Create a job requirement profile (mark hard vs. soft criteria).
2. Upload CVs → view ranking, exclusions, and score breakdown.
3. Schedule an interview → share candidate link.
4. Conduct interview → watch live transcript.
5. Run analysis → view consolidated assessment, compare candidates, export.
6. `<Innovation feature walkthrough>`

## 8. Performance

| Metric | Measured |
|---|---|
| 20 CVs parsed + screened + scored + ranked (N1) | `<X s>` |
| Transcription latency (N2) | `<X ms>` |

## 9. Privacy & Data Handling

- **Data used:** provided sample/synthetic CVs only; nothing personal committed.
- **Raw media:** never stored. Audio is streamed in memory to Google STT and flushed immediately; only text transcripts are persisted.
- **Third-party disclosure (N18):**

| Service | Data sent | Purpose |
|---|---|---|
| OpenAI | CV text (protected attributes stripped), transcript excerpts | Scoring, analysis, embeddings |
| Google Cloud STT | Live audio chunks | Transcription |
| Firebase | Auth identity | Role authentication |

- **Deletion (N19):** `DELETE /api/v1/candidates/{id}` (admin) purges candidate data across all stores.
- **Biometrics (N17):** `<N/A or consent + retention details if F7>`
- **Fairness (N20):** scoring uses job-relevant criteria only; `<known limitations>`

## 10. Repository Layout

```
docs/  backend/  frontend/  test_data/  docker-compose.yml  .env.example
```

## 11. Documentation Index

`docs/SRS.md` · `docs/clarification_document.md` · `docs/risk_register.md` · `docs/architecture_diagram.md` · `docs/WBS_actual.xlsx` · `docs/innovation_feature_rationale.md`

## 12. Known Limitations

- `<be honest: what does not work, and why>`
