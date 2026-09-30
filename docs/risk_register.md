# Risk Register (P1.5)

Scale: Likelihood (L) / Impact (I) = Low, Medium, High.

| ID | Risk | Category | L | I | Mitigation | Contingency | Owner / Checkpoint |
|---|---|---|---|---|---|---|---|
| R1 | **STT latency**: streaming transcription lags or interim results stutter. | Technical | M | H | Send 100 ms PCM chunks at 16 kHz; one gRPC stream per channel; show interim results; measure p95 latency early. | Show "transcribing…" state; fall back to final-only display. | Days 12–13 (M3) |
| R2 | **STT stream limit**: Google streaming sessions cap at ~5 min; stream drops mid-interview. | Technical | H | H | Auto-rotate streams before the limit with small overlap; persist final turns incrementally. | Reconnect and continue; mark gap in transcript. | Days 12–13 |
| R3 | **WebRTC disconnects** / NAT traversal failure. | Technical | H | H | Prototype on Days 4–5; STUN + optional TURN; ICE restart and auto-rejoin; persist turns incrementally so no transcript loss. | Audio-only mode; manual rejoin link. | Days 4–5, 9–11 (M3) |
| R4 | **Audio-only WS drops** while video stays up (or vice versa). | Technical | M | M | Client ring-buffer of last ~2 s of chunks (in memory) resent on reconnect; heartbeat pings. | Flag missing segment. | Days 12–13 |
| R5 | **PDF/OCR parsing failure** on multi-column, scanned, or image-heavy CVs. | Technical | H | M | PyMuPDF/pdfplumber first, OCR fallback (Tesseract); per-CV isolation with `PARSE_FAILED` status; batch never aborts. | Manual re-upload or text paste. | Days 4–8 (M2) |
| R6 | **LangGraph crash recovery** fails or re-runs completed nodes. | Technical | M | H | `AsyncPostgresSaver` on all graphs; deterministic `thread_id` (e.g., `cv:{candidate_id}`); idempotent node side-effects; kill-and-resume test. | Manual re-trigger from last persisted state. | Days 4–5, 19 |
| R7 | **Checkpoint state bloat / serialisation errors** with large CV text. | Technical | M | M | Store references/IDs in graph state, not raw documents; keep state Pydantic-typed. | Trim state; prune old checkpoints. | Days 6–8 |
| R8 | **LLM score instability** (non-reproducible). | Technical | M | M | Temperature 0, fixed seed, structured output, hash-keyed cache, frozen weights. | Show score variance note. | Days 6–8 |
| R9 | **LLM/API rate limits, cost caps, outages** (OpenAI, Google STT). | External | M | H | Backoff + retry edges; cache; batch; dev-time budget alarms. | Clear `FAILED` state, retry button; queue work. | Throughout |
| R10 | **Structured-output failures** (invalid JSON, out-of-range scores). | Technical | M | M | Pydantic validation node with bounded repair-retry loop. | Mark dimension "unscorable". | Days 6–8 |
| R11 | **RAG knowledge base too thin** to verify technical answers. | Technical | M | M | Curate verified docs for the roles in the sample CVs; return "insufficient evidence" instead of guessing. | Fall back to LLM-only with lowered confidence flag. | Days 14–15 |
| R12 | **Zero-media-retention violated** accidentally (temp files, logs, debug dumps). | Privacy | L | H | No disk writes in audio path; code-review checklist; `.gitignore` media patterns; log redaction. | Purge and rotate. | Day 18 hardening |
| R13 | **Personal data or secrets committed** to repo. | Compliance | M | H | `.gitignore`, pre-commit secret scan, review before tagging. | History rewrite and key rotation. | Every commit; Day 19 |
| R14 | **Fairness/bias** in AI scoring. | Ethical | M | H | Strip protected attributes pre-LLM; job-relevant criteria only; document limitations. | Human-review disclaimer in reports. | Days 6–8, 19 |
| R15 | **Over-engineering** early features; core pipeline unfinished. | Schedule | H | H | Thin end-to-end slice first; daily WBS tracking. | Cut stretch/innovation depth. | Ongoing |
| R16 | **Docker works locally, not on clean machine.** | Delivery | M | H | Clean-machine test by Day 19; pinned image tags; healthchecks; `depends_on` conditions. | Reduce services; document prerequisites. | Day 19 (M6) |
| R17 | **Innovation feature scope creep** or weak integration. | Schedule | M | M | Decide by Day 15; time-box to Days 16–17. | Ship smallest integrated version. | Day 15 |
| R18 | **Dual-channel misuse**: a participant's mic sent on the wrong channel. | Technical | L | M | Server assigns channel from authenticated role, ignoring client claims. | Correct mapping in transcript UI. | Days 12–13 |
| R19 | **Solo-delivery bandwidth / illness.** | Schedule | M | M | Buffer in Day 18; strict prioritisation. | Drop stretch. | Ongoing |
