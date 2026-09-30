# Architecture Diagrams (P1.4)

> Draft scaffold. Per BID §13.2 (NA3) the workflow diagram must be your own work: use this as a starting point, then redraw and export `architecture_diagram.png` in your own style.

## 1. System Boundary (ASCII)

```
                        ┌───────────────────────────── Docker Compose network ─────────────────────────────┐
                        │                                                                                  │
 ┌──────────────┐  HTTPS │  ┌────────────┐   REST /api/v1     ┌───────────────────────────────────────┐   │
 │ Interviewer  │◄──────►│  │  Frontend  │◄──────────────────►│        FastAPI (ASGI / Uvicorn)       │   │
 │   Browser    │        │  │ (React SPA)│  WS: signalling    │  routers: auth jobs candidates        │   │
 └──────┬───────┘        │  └────────────┘  WS: audio ch0/1   │           sessions analysis innovation │   │
        │ WebRTC P2P     │                                     │  services: cv_parser stt_stream        │   │
        │ (video+audio)  │  ┌────────────┐                     │            rag_engine webrtc_signaling │   │
 ┌──────┴───────┐        │  │  Frontend  │◄───────────────────►│  agents (LangGraph):                   │   │
 │  Candidate   │◄──────►│  │(same SPA)  │                     │   cv_screening / qa_analysis /         │   │
 │   Browser    │        │  └────────────┘                     │   assessment                           │   │
 └──────────────┘        │                                     └───┬───────────┬───────────┬───────────┘   │
                         │                                         │           │           │               │
                         │                                 ┌───────▼──────┐ ┌──▼────────┐ │               │
                         │                                 │ PostgreSQL   │ │ ChromaDB  │ │               │
                         │                                 │ app data +   │ │ CV chunks │ │               │
                         │                                 │ LangGraph    │ │ + verified│ │               │
                         │                                 │ checkpoints  │ │ tech docs │ │               │
                         │                                 └──────────────┘ └───────────┘ │               │
                         └────────────────────────────────────────────────────────────────┼───────────────┘
                                                                                          │ outbound only
                              ┌──────────────────┬────────────────────┬───────────────────┘
                              ▼                  ▼                    ▼
                     Google Cloud STT       OpenAI API          Firebase Auth
                     (gRPC streaming,     (LLM + embeddings)    (role identity)
                      audio in memory)
```

## 2. System Context & Containers (Mermaid)

```mermaid
flowchart LR
    subgraph Users
        I[Interviewer Browser]
        C[Candidate Browser]
    end

    subgraph Compose["Docker Compose Network"]
        FE[Frontend SPA]
        subgraph API["FastAPI ASGI Backend"]
            R[Routers: auth, jobs, candidates, sessions, analysis, innovation]
            S1[cv_parser]
            S2[stt_stream]
            S3[rag_engine]
            S4[webrtc_signaling]
            subgraph AG["LangGraph Agents"]
                G1[cv_screening_graph]
                G2[qa_analysis_graph]
                G3[assessment_graph]
            end
        end
        PG[(PostgreSQL: app data + checkpoints)]
        CH[(ChromaDB: CV chunks + verified docs)]
    end

    STT[Google Cloud STT gRPC]
    LLM[OpenAI LLM + Embeddings]
    FB[Firebase Auth]

    I --> FE
    C --> FE
    FE -->|REST| R
    FE <-->|WS signalling| S4
    FE -->|WS PCM audio ch0/ch1| S2
    I <-->|WebRTC P2P media| C
    R --> S1
    R --> AG
    AG <-->|AsyncPostgresSaver| PG
    R --> PG
    S1 --> CH
    S3 <--> CH
    G2 --> S3
    S2 -->|gRPC stream per channel| STT
    S2 -->|final turns| PG
    AG --> LLM
    S1 --> LLM
    FE --> FB
    R -->|verify token| FB
```

## 3. Dual-Channel WebRTC + STT Sequence

Video/audio between participants flows peer-to-peer. Independently, each browser taps its **own local microphone**, downsamples to 16 kHz PCM via an AudioWorklet, and streams chunks to the backend. The backend derives the channel from the authenticated role, never from client claims (risk R18). No audio is written to disk.

```mermaid
sequenceDiagram
    participant IB as Interviewer Browser
    participant CB as Candidate Browser
    participant SG as FastAPI Signalling WS
    participant AS as FastAPI Audio WS
    participant ST as stt_stream
    participant G as Google STT (gRPC)
    participant DB as PostgreSQL

    IB->>SG: join(session_id, role=interviewer)
    CB->>SG: join(session_id, link_token)
    SG-->>IB: peer-joined
    IB->>SG: SDP offer + ICE
    SG-->>CB: relay offer
    CB->>SG: SDP answer + ICE
    SG-->>IB: relay answer
    Note over IB,CB: P2P video/audio established (STUN/TURN)

    par Channel 0 (Interviewer)
        IB->>AS: PCM chunks (ws /audio/{sid}/0)
        AS->>ST: chunk + speaker=INTERVIEWER
        ST->>G: StreamingRecognize (stream A)
    and Channel 1 (Candidate)
        CB->>AS: PCM chunks (ws /audio/{sid}/1)
        AS->>ST: chunk + speaker=CANDIDATE
        ST->>G: StreamingRecognize (stream B)
    end

    G-->>ST: interim/final results
    ST-->>IB: live interim transcript (WS)
    ST->>DB: persist final turn {speaker, start_ms, end_ms, text}
    ST->>ST: flush audio buffer immediately
    Note over ST: stream rotation before ~5 min limit
    IB->>SG: end-session
    SG->>DB: status=ENDED, ended_at
```

## 4. LangGraph Workflows

All graphs compile with `AsyncPostgresSaver` and are invoked with `config={"configurable": {"thread_id": ...}}`. On restart, invoking with the same `thread_id` resumes from the last checkpoint.

### 4.1 `cv_screening_graph` (F1 + F2) — thread_id: `cv:{candidate_id}`

```mermaid
flowchart TD
    A[load_cv_and_job] --> B[parse_and_extract_structured]
    B -->|parse fail| X[mark_parse_failed]
    B --> C[hard_requirement_gate]
    C -->|any hard fail| E[mark_EXCLUDED_HARD_FAIL]
    C -->|pass| D[load_or_generate_frozen_weights]
    D --> F[semantic_match_requirements<br/>embeddings + Chroma]
    F --> G[score_dimensions<br/>0-100 + justification]
    G --> H[validate_scores<br/>Pydantic]
    H -->|invalid, retries left| G
    H -->|valid| I[aggregate_weighted_score]
    I --> J[persist_results]
    E --> Z((END))
    X --> Z
    J --> Z
```

State (`state.py`): `candidate_id, job_id, cv_text_ref, structured_cv, hard_gate_result, weights, requirement_matches, dimension_scores, aggregate, status, errors, retry_count`.

### 4.2 `qa_analysis_graph` (F5) — thread_id: `qa:{session_id}`

```mermaid
flowchart TD
    A[load_transcript] --> B[segment_qa_pairs<br/>follow-ups, multi-part]
    B --> C[map_to_requirements]
    C --> D{next pair?}
    D -->|yes| E[rag_retrieve<br/>ChromaDB verified docs]
    E --> F[evaluate_answer<br/>relevance, accuracy, completeness, articulation]
    F --> G[validate_and_cite<br/>turn IDs + doc refs]
    G -->|invalid, retries left| F
    G --> D
    D -->|done| H[coverage_gap_analysis<br/>unexplored requirements]
    H --> I[persist_qa_analysis]
    I --> Z((END))
```

### 4.3 `assessment_graph` (F6) — thread_id: `assess:{candidate_id}:{session_id}`

```mermaid
flowchart TD
    A[load_screening_scores_qa] --> B[merge_components]
    B --> C[summarise_strengths_weaknesses]
    C --> D[identify_evidence_gaps_and_followups]
    D --> E[link_evidence<br/>CV snippets + transcript turns]
    E --> F[compose_report]
    F --> G[persist_assessment]
    G --> H[render_export_PDF_HTML]
    H --> Z((END))
```

## 5. Deployment Topology (Compose services)

| Service | Image / build | Ports | Volume | Depends on |
|---|---|---|---|---|
| `frontend` | `./frontend` | 3000 | – | backend |
| `backend` | `./backend` | 8000 | – | postgres (healthy), chroma |
| `postgres` | `postgres:16-alpine` | internal | `pgdata` (named) | – |
| `chroma` | `chromadb/chroma` (pinned) | internal | `chroma_data` (named) | – |

## 6. Key Design Decisions

| Decision | Rationale |
|---|---|
| Per-participant mic capture, one STT stream per channel | Deterministic attribution; avoids diarization error (Q clarified rule). |
| Audio never touches disk | Meets N16; only transcript text persisted. |
| Frozen dynamic weights per job | AI-derived weights yet reproducible scoring (B2, F2.5). |
| Hard gate as an explicit graph node with conditional edge | Visible, testable, halts scoring cleanly. |
| Postgres checkpointer on every graph | Crash-resume without re-running completed nodes (Q45). |
| RAG cited in evaluation | Evidence-backed accuracy verification (Q10). |
