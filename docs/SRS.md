# Software Requirements Specification (SRS)

**Product:** `<PROJECT_NAME>` — AI-Powered Recruitment & Interview Intelligence Platform
**Version:** 0.1 (Draft) | **Programme:** AI Trainee Developer — Final Evaluation
**Status:** DRAFT SCAFFOLD. Per BID §13.2 (NA2), the SRS must be authored by the trainee without AI support. Review, rewrite and own every statement before submission.

---

## 1. Introduction

### 1.1 Purpose
Defines the functional, non-functional, and architectural requirements of a platform that supports hiring end to end: CV screening and ranking (pre-interview), live video interview with speaker-attributed transcription (during), and Q&A analysis with a consolidated assessment (post). The platform **augments** human decisions; every AI output must be explainable and evidence-traceable.

### 1.2 Scope
In scope and out of scope follow BID §4. Out of scope: production deployment, enterprise SSO, ATS/HRIS integration, job posting/sourcing, calendaring integrations, multi-tenancy/billing, native mobile, model training, WCAG certification.

### 1.3 Definitions
| Term | Meaning |
|---|---|
| Hard requirement | A mandatory job criterion (e.g., minimum years of experience). Failing it excludes the candidate. |
| Channel 0 / Channel 1 | Interviewer audio / Candidate audio, kept as separate streams. |
| Checkpointer | LangGraph persistence layer (`AsyncPostgresSaver`) keyed by `thread_id`. |
| RAG | Retrieval-Augmented Generation against ChromaDB verified technical documents. |

### 1.4 Actors
| Actor | Capabilities |
|---|---|
| Interviewer / Recruiter | Manage jobs, upload CVs, view rankings/scores, schedule and run interviews, view assessments. |
| Candidate | Join a scheduled interview via unique link only. **No access** to scoring, ranking, or assessment data (N9). |
| Administrator | Everything above plus candidate data deletion (N19). |

---

## 2. Overall Description

### 2.1 Architecture summary
React-based web frontend → FastAPI (ASGI) backend → PostgreSQL (app data + LangGraph checkpoints), ChromaDB (vectors/RAG), OpenAI (LLM, embeddings), Google Cloud STT (streaming gRPC), Firebase (role auth). WebRTC for peer video; per-participant PCM audio streamed over WebSocket for STT. See `architecture_diagram.md`.

### 2.2 Constraints
Python 3.10+, LangGraph, FastAPI + Pydantic, Docker Compose, GitHub (BID §8.1); local Docker execution only; sample/synthetic data only; solo delivery; English transcription (A7).

### 2.3 Assumptions
| # | Assumption |
|---|---|
| AS-1 | Firebase Authentication is used only for simple role separation (interviewer/admin/candidate), not enterprise SSO. *(Inferred from the Firebase placeholder; confirm.)* |
| AS-2 | Candidate interview access is by a signed, unguessable, expiring link — not by candidate account creation. |
| AS-3 | Scanned/image-only CVs are handled by OCR fallback; if OCR fails the CV is flagged `PARSE_FAILED` and skipped without aborting the batch. |
| AS-4 | Dynamic weights are computed once per job/prompt and **frozen** (persisted) so re-scoring is reproducible. |
| AS-5 | The BID states both "17 development days" and "20 working days"; plan uses 20 days (3 + 17). Confirm with trainer. |

---

## 3. Functional Requirements

Priority: **M** Must (Core), **S** Stretch (bonus), **C** Creative.

### Stage 1 — Pre-Interview

**F1 — CV Screening & Requirement-Based Ranking (M)**
| ID | Requirement |
|---|---|
| F1.1 | Accept a job requirement profile: required/preferred skills, qualifications, minimum years of experience, extra criteria, and a flag marking each criterion as *hard* or *soft*. |
| F1.2 | Batch upload of CVs (PDF mandatory, DOCX supported); validate type and size before processing (N8). |
| F1.3 | Extract structured data: contact, education, work history, skills, certifications, projects. OCR fallback for scanned PDFs. |
| F1.4 | **Hard-requirement gate:** a candidate failing any hard requirement receives status `EXCLUDED_HARD_FAIL`, the reason is recorded, and no further scoring occurs (`cv_screening_graph` filter node). |
| F1.5 | Semantic matching (embeddings + LLM), not keyword-only (e.g., "REST services in Django" ↔ "Python web development"). |
| F1.6 | Produce an ordered ranking of non-excluded candidates; excluded candidates listed separately with reasons. |
| F1.7 | Per candidate, show each requirement as Met / Partially Met / Unmet with CV evidence snippets. |
| F1.8 | Failures on individual CVs (bad layout, corrupt file) are logged and skipped; the batch continues. |

**F2 — AI-Based Candidate Scoring (M)**
| ID | Requirement |
|---|---|
| F2.1 | Aggregate score and every per-dimension score are on a strict **0–100** scale (Pydantic `Field(ge=0, le=100)`; LLM output is structured JSON validated by Pydantic). |
| F2.2 | Minimum dimensions: technical skills, relevant experience, qualifications. Additional dimensions allowed. |
| F2.3 | **Dimension weights are determined dynamically by the AI** from job context or a recruiter prompt (dedicated weight-extraction node before aggregation). Weights sum to 1.0, are persisted with the job, and may be inspected. |
| F2.4 | Each dimension score carries a short natural-language justification with evidence. |
| F2.5 | Reproducibility: temperature 0, fixed seed, cached results keyed by hash(CV text + job profile + frozen weights). |
| F2.6 | Scores are persisted and retrievable for consolidation (F6). |
| F2.7 | Scoring inputs exclude protected characteristics (N20). |

### Stage 2 — During-Interview

**F3 — AI-Powered Video Interview (M)**
| ID | Requirement |
|---|---|
| F3.1 | Interviewer creates a session linked to one candidate and one job profile. |
| F3.2 | **Scheduling & links:** session endpoint supports a future `scheduled_at` timestamp and triggers generation of a unique, expiring candidate interview link/token (LLM-assisted generation per clarification Q39/Q40; token validity enforced server-side). |
| F3.3 | Browser WebRTC video/audio; lifecycle: create/schedule → join → in-progress → ended. |
| F3.4 | Controls: mute, camera toggle, end session. |
| F3.5 | Signalling via FastAPI WebSocket (`webrtc_signaling.py`); STUN configured, optional TURN. |
| F3.6 | Graceful degradation: on disconnect, auto-reconnect; already-captured transcript turns are retained (persisted incrementally). |
| F3.7 | Session record persisted: participants, start/end times, status. |

**F4 — Real-Time Speech Recognition & Transcription (M)**
| ID | Requirement |
|---|---|
| F4.1 | **Deterministic speaker attribution without ML diarization:** each participant's browser captures its own microphone and streams 16 kHz PCM chunks over WebSocket tagged Channel 0 (interviewer) or Channel 1 (candidate). Backend runs one streaming STT recognition per channel. |
| F4.2 | Streaming transcription via Google Cloud STT gRPC; interim results shown live, final results persisted. |
| F4.3 | Each turn stored as `{session_id, speaker, start_ms, end_ms, text, confidence}` in order. |
| F4.4 | **Zero raw media retention:** audio exists only as in-memory chunks; buffers are flushed immediately after transcription; no audio/video file is written to disk or DB (N16). |
| F4.5 | Handle silence, accents and overlap without failing; STT stream timeouts (~5 min) auto-rotate to a new stream without losing turns. |
| F4.6 | Transcript retrievable after session end. |

**F7 — Candidate Identification & Real-Time Profile Overview (S)**
Optional. If implemented: explicit visible consent, low-confidence handling (never silently assume identity), profile panel (name, headline experience, skills, pre-interview score, probe areas), biometric storage/retention stated in README (N17).

### Stage 3 — Post-Interview

**F5 — Interview Q&A Analysis (M)**
| ID | Requirement |
|---|---|
| F5.1 | Segment transcript into question–answer pairs, handling follow-ups, clarifications and multi-part answers. |
| F5.2 | Evaluate each answer on relevance, technical accuracy, completeness, articulation (each 0–100) with justification citing transcript turn IDs. |
| F5.3 | **RAG verification:** technical accuracy is assessed by retrieving authoritative documents from ChromaDB (verified frameworks/docs) and grounding the evaluation on retrieved passages, which are cited (Q10). |
| F5.4 | Map each Q&A pair to job requirements; report coverage and **requirements never explored**. |

**F6 — Consolidated Candidate Assessment (M)**
| ID | Requirement |
|---|---|
| F6.1 | Merge CV screening, scores, and Q&A analysis into one report with component scores (no single opaque number). |
| F6.2 | Strengths, weaknesses, evidence gaps, areas for further verification. |
| F6.3 | Export/print as PDF or structured HTML. |
| F6.4 | Side-by-side comparison of multiple assessed candidates for one job. |
| F6.5 | Every conclusion links to a CV snippet or transcript turn. |

**F8 — CV Claim Verification (S)** Flags as *contradiction / unverified / not discussed*, framed as human follow-up prompts.
**F9 — Comprehensive Interview Scoring (S)** Separate interview score; six dimensions per BID §5.4.2.
**F10 — Interview Quality Assessment (S)** Question effectiveness, coverage, talk-time balance, recommendations.

### Creative

**F11 — Innovation Feature (C)** One original, integrated, documented feature (`api/innovation.py`); rationale in `innovation_feature_rationale.md` (300–500 words). *Concept: TBD by trainee.*

### Cross-cutting

| ID | Requirement |
|---|---|
| F-X1 | **LangGraph persistence:** all graphs compile with `AsyncPostgresSaver` and are invoked with a `thread_id`; after a crash/restart, workflows resume from the last checkpoint without re-running completed nodes (Q45). |
| F-X2 | **Data deletion:** `DELETE /api/v1/candidates/{id}` (admin only) cascades to CV data, vector chunks, scores, sessions, transcripts, Q&A, assessments, and LangGraph checkpoints for that candidate's threads (N19, Q15). |
| F-X3 | Long-running AI tasks run asynchronously with pollable status/progress (N4). |

---

## 4. Non-Functional Requirements (N1–N26)

| ID | Category | Requirement | Design response |
|---|---|---|---|
| N1 | Performance | 20-CV batch parsed, screened, scored, ranked in a documented time; measure and publish in README. | Async concurrent per-CV graph runs; embedding/LLM caching. |
| N2 | Performance | Minimal transcription lag. | Streaming gRPC, interim results over WS. |
| N3 | Performance | Stable video/audio for a typical interview. | WebRTC P2P, STUN/TURN, reconnect logic. |
| N4 | Performance | Fast non-AI endpoints; AI ops async with status. | Background tasks + job status endpoints. |
| N5 | Performance | One live interview alongside background CV processing. | Async I/O, separate worker concurrency limits. |
| N6 | Reliability | External AI failure yields clear error state, no crash/data loss. | Retry/backoff edges in graphs; `FAILED` status with user-safe message. |
| N7 | Security | Secrets only via env vars; `.env.example` provided. | pydantic-settings; `.gitignore`. |
| N8 | Security | Validate all inputs; check upload type/size. | Pydantic; MIME sniffing + extension + size cap. |
| N9 | Security | Candidate cannot access scores/rankings/assessments. | Role-based dependencies on every router. |
| N10 | Security | Secure transport; document local exceptions. | WSS/HTTPS in principle; localhost HTTP documented (getUserMedia allowed on localhost). |
| N11 | Security | Pin dependencies. | Exact versions in `requirements.txt` / lockfile. |
| N12 | Security | No stack traces/paths/prompts in errors. | Global exception handler with generic messages + correlation ID. |
| N13 | Privacy | Mock/anonymised data only. | 20 sample CVs + synthetic. |
| N14 | Privacy | No personal data in repo. | `.gitignore` for `test_data/`, media, dumps. |
| N15 | Privacy | Anonymise demo material. | Redaction checklist before slides/screenshots. |
| N16 | Privacy | Data minimisation; no raw audio/video retained. | In-memory streaming only; transcript stored. |
| N17 | Privacy | Biometric caution (if F7). | Explicit consent, documented retention. |
| N18 | Privacy | Document third-party data disclosure. | README table: OpenAI, Google STT, Firebase. |
| N19 | Privacy | Deletion capability. | Admin cascade-purge endpoint. |
| N20 | Fairness | Score on job-relevant criteria only; no protected attributes. | Redact/strip protected fields pre-LLM; prompt constraints; fairness limitations in README. |
| N21 | Reproducibility | Clean-machine start via Docker only. | Single `docker compose up --build`. |
| N22 | Observability | Structured logs; no PII/secrets. | JSON logging, ID-only references, redaction filter. |
| N23 | Quality | Clean structure, no dead code. | Layered: api → services → agents. |
| N24 | Docs | README + OpenAPI. | FastAPI auto docs at `/docs`. |
| N25 | Usability | Self-navigable UI, progress, friendly errors. | Progress indicators; error mapping. |
| N26 | Persistence | Data survives restart. | Named Docker volumes for Postgres and Chroma. |

---

## 5. Clarification Register (Integrated)

> **Source status:** The entries below are those supplied in the clarified business rules matrix. The full set is 49 Q&As; entries without a Q-number in the source are marked *unnumbered*. The remainder must be imported (see 5.2).

### 5.1 Known clarified rules
| Ref | Area | Clarified rule | Requirement(s) |
|---|---|---|---|
| Q-unnumbered | Hard requirement gate | Candidates failing any mandatory requirement (e.g., min. years of experience) are excluded from screening. | F1.4 |
| Q-unnumbered | Scoring scale | Overall and per-dimension scores strictly 0–100. | F2.1 |
| Q-unnumbered | Score weighting | AI determines dimension weights dynamically from job context or recruiter prompt. | F2.3 |
| Q-unnumbered | Speaker attribution | Deterministic; dual WebRTC audio channels (0 interviewer, 1 candidate) instead of mixed-audio diarization. | F4.1 |
| Q-unnumbered | Media retention | Zero storage of raw audio/video; in-memory streaming to Google STT gRPC; immediate buffer flush. | F4.4, N16 |
| Q10 | RAG verification | Technical accuracy evaluated against authoritative documents via RAG (ChromaDB). | F5.3 |
| Q15 | Data deletion | Admin capability to purge candidate personal data. | F-X2, N19 |
| Q39, Q40 | Scheduling & links | LLM-generated unique interview links; future date/time scheduling supported; `scheduled_at` stored. | F3.2 |
| Q45 | LangGraph persistence | Workflows survive crashes and resume without re-running completed steps; `AsyncPostgresSaver` + `thread_id`. | F-X1 |
| N19 | Deletion API | `DELETE /api/v1/candidates/{id}` cascading purge. | F-X2 |

### 5.2 Remaining Q&As to import
Paste the remaining entries here in this format, then link to requirement IDs:

| Ref | Area | Question | Trainer answer | Requirement(s) |
|---|---|---|---|---|
| Q__ | | | | |

---

## 6. Data Model (Logical)

`users(id, firebase_uid, role)` · `jobs(id, title, profile_json, weights_json, weights_frozen_at)` · `candidates(id, job_id, status, parse_status, exclusion_reason)` · `cv_documents(id, candidate_id, structured_json, text_hash)` · `screening_results(id, candidate_id, requirement_matches_json)` · `scores(id, candidate_id, aggregate, dimensions_json, justification_json)` · `interview_sessions(id, candidate_id, job_id, scheduled_at, started_at, ended_at, status, link_token_hash, link_expires_at)` · `transcript_turns(id, session_id, speaker, start_ms, end_ms, text, confidence)` · `qa_pairs(id, session_id, question_turn_ids, answer_turn_ids, evaluation_json, requirement_ids)` · `assessments(id, candidate_id, session_id, report_json)` · LangGraph checkpoint tables (managed by `AsyncPostgresSaver`).

## 7. API Surface (v1)

| Router | Representative endpoints |
|---|---|
| `auth` | `POST /auth/verify`, `GET /auth/me` |
| `jobs` | `POST/GET /jobs`, `POST /jobs/{id}/weights:generate` |
| `candidates` | `POST /jobs/{id}/cvs` (batch), `GET /jobs/{id}/ranking`, `GET /candidates/{id}`, `DELETE /candidates/{id}` |
| `sessions` | `POST /sessions`, `GET /sessions/{id}`, `POST /sessions/{id}/end`, `WS /ws/signaling/{id}`, `WS /ws/audio/{id}/{channel}` |
| `analysis` | `POST /sessions/{id}/analyze`, `GET /analysis/{task_id}`, `GET /candidates/{id}/assessment`, `GET /jobs/{id}/compare` |
| `innovation` | TBD |

## 8. Traceability

| BID Feature | SRS | Graph / Service |
|---|---|---|
| F1, F2 | F1.x, F2.x | `cv_screening_graph`, `cv_parser`, `rag_engine` |
| F3 | F3.x | `webrtc_signaling`, `sessions` |
| F4 | F4.x | `stt_stream` |
| F5 | F5.x | `qa_analysis_graph`, `rag_engine` |
| F6 | F6.x | `assessment_graph` |
| F11 | F11 | `innovation` |
