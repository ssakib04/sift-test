# Requirement Clarification Document (P1.2)

**Product:** `<PROJECT_NAME>` | **Phase 1** | **Status:** Draft — import remaining Q&As

The full clarification set contains 49 Q&As. Entries below are those provided so far, grouped by feature area. Add the remaining entries in the same format; do not paraphrase trainer answers.

## A. Ambiguities Identified in the BID

| # | Ambiguity / conflict | Impact | Resolution / assumption |
|---|---|---|---|
| A-1 | Duration stated as both "17 working days" and "20 working days" (3 + 17). | Planning | Plan on 20 days total; confirm dates with trainer. |
| A-2 | "Job Post" text appears inside the In-Scope table while job posting is out of scope (§4.2). | Scope | Treat as informational job-post content used as sample input; no publishing functionality. |
| A-3 | Auth scope: "simple role separation acceptable" but mechanism unspecified. | Security | Firebase Auth for roles only (assumption AS-1). |
| A-4 | Speaker attribution: diarisation *or* channel separation. | F4 | Channel separation chosen (see C). |
| A-5 | Weighting: "configurable" vs. who configures. | F2 | AI-determined dynamically; inspectable and frozen per job. |

## B. Pre-Interview (F1, F2)

| Ref | Topic | Clarified rule |
|---|---|---|
| Q-unnumbered | Hard-requirement gate | Candidates failing any mandatory requirement (e.g., minimum years of experience) are excluded. Filter node sets `EXCLUDED_HARD_FAIL` and halts scoring. |
| Q-unnumbered | Score scale | Overall and per-dimension scores strictly 0–100, enforced via Pydantic `ge=0, le=100`. |
| Q-unnumbered | Weighting | AI determines dimension weights from job context or recruiter prompt via a dynamic weight-extraction node before aggregation. |

## C. During-Interview (F3, F4)

| Ref | Topic | Clarified rule |
|---|---|---|
| Q-unnumbered | Speaker attribution | Deterministic, no mixed-audio ML diarization. WebRTC dual tracks: Channel 0 = interviewer, Channel 1 = candidate, mapped to metadata tags. |
| Q-unnumbered | Media retention | No raw audio/video stored (N16). In-memory chunks streamed to Google Cloud STT gRPC; buffers flushed immediately after transcription. |
| Q39, Q40 | Scheduling & links | LLM generates unique candidate interview links; future date/time scheduling supported. Session endpoint stores `scheduled_at` and generates the token. |

## D. Post-Interview (F5, F6)

| Ref | Topic | Clarified rule |
|---|---|---|
| Q10 | Answer verification | Technical accuracy evaluated against authoritative documents via RAG. Evaluation node queries ChromaDB of verified frameworks/docs. |

## E. Platform, Privacy & Reliability

| Ref | Topic | Clarified rule |
|---|---|---|
| Q45 | Crash recovery | LangGraph workflows resume after crash without re-running completed steps. `AsyncPostgresSaver` attached to every compiled graph; invoked with `thread_id`. |
| Q15 / N19 | Data deletion | Admin capability to purge candidate data. `DELETE /api/v1/candidates/{id}` cascades across candidate, scores, transcripts. |

## F. Remaining Q&As (to import)

| Ref | Feature area | Question | Trainer response | Assumption if unanswered |
|---|---|---|---|---|
| Q__ | | | | |

## G. Open Assumptions

| # | Assumption | Confirm with trainer? |
|---|---|---|
| AS-1 | Firebase used for role auth only. | Yes |
| AS-2 | Candidates join by expiring signed link, no account. | Yes |
| AS-3 | Scanned CVs use OCR fallback; failures skipped with `PARSE_FAILED`. | No |
| AS-4 | Dynamic weights frozen after first generation for reproducibility. | Yes |
| AS-5 | 20-day total plan. | Yes |
| AS-6 | Deletion also purges LangGraph checkpoints and Chroma vectors for the candidate. | No |
