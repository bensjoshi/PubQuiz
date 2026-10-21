# Pub Quiz Host — Requirements

> An AI-powered pub quiz host: generates fact-checked quiz rounds from a source corpus, runs live multiplayer games on phones + a big screen, and marks answers fairly.
>
> **Portfolio purpose:** demonstrate production AI engineering — RAG, structured outputs, agents/tool calling, guardrails, evals in CI, LLM observability, PostgreSQL, and cloud deployment.

---

## 1. Goals

| # | Goal | Evidence in repo |
|---|------|------------------|
| G1 | Fun, playable quiz night with real people | Demo video from a live quiz |
| G2 | Grounded question generation (RAG) | Every question cites a source passage |
| G3 | Measurable AI quality | Eval suites with published scores |
| G4 | Cost/latency awareness | Model comparison + cost-per-game figures |
| G5 | Production engineering | Postgres, CI/CD, cloud deployment, tests |

### Non-goals (v1)
- User accounts / authentication for players
- Native mobile apps
- Fine-tuning models
- Monetisation

---

## 2. Users & Views

| View | Route | Device | Purpose |
|------|-------|--------|---------|
| **Host** | `/host/:gameId` | Laptop | Create game, control flow, approve questions, override marks |
| **Big Screen** | `/screen/:gameId` | TV/projector | Show questions, timer, leaderboard, reveal answers |
| **Player** | `/play/:code` | Phone | Join with team name, submit answers, see own score |

---

## 3. Functional Requirements

### 3.1 Game setup
- [ ] **FR-1** Host can create a game and receive a short join code (e.g. 4–6 chars) and QR code.
- [ ] **FR-2** Host can configure: number of rounds, questions per round, time per question, themes per round.
- [ ] **FR-3** Host can generate rounds via AI from selected themes/corpora.
- [ ] **FR-4** Host can review generated questions and approve, reject, edit, or regenerate each one before the game.
- [ ] **FR-5** Host can save a quiz and reuse it later.

### 3.2 Joining & lobby
- [ ] **FR-6** Players join by code/QR with a team name; no account needed.
- [ ] **FR-7** Duplicate team names are rejected.
- [ ] **FR-8** Big screen shows joined teams live in the lobby.
- [ ] **FR-9** Players who disconnect can rejoin and keep their team and score (session token in browser storage).

### 3.3 Gameplay
- [ ] **FR-10** Host advances: start round → show question → lock answers → reveal → leaderboard.
- [ ] **FR-11** Big screen displays question with countdown timer; question is read aloud (TTS).
- [ ] **FR-12** Players submit one free-text answer per question; can edit until timer ends or host locks.
- [ ] **FR-13** All views stay in sync in real time (WebSockets).
- [ ] **FR-14** Final round scores double points.
- [ ] **FR-15** Animated leaderboard after each round; winner screen at the end.

### 3.4 Marking
- [ ] **FR-16** Answers are marked automatically via a layered pipeline (see §4.3).
- [ ] **FR-17** Host sees each answer with its mark + reason and can override with one click.
- [ ] **FR-18** Overrides are stored and reusable as eval data.

### 3.5 Post-game
- [ ] **FR-19** Stats: hardest question, closest race, most creative wrong answer.
- [ ] **FR-20** Game history stored and viewable by host.

---

## 4. AI Requirements

### 4.1 Corpus ingestion (RAG)
- [ ] **AI-1** Ingest pipeline loads source documents (start: Wikipedia articles by category; later: local Sheffield history, music trivia).
- [ ] **AI-2** Documents are chunked, embedded, and stored in Postgres via **pgvector**, with source URL + title metadata.
- [ ] **AI-3** Retrieval supports hybrid search (vector + keyword/full-text) filtered by theme.
- [ ] **AI-4** Ingestion is idempotent and re-runnable (CLI command).

### 4.2 Question generation
- [ ] **AI-5** Generator retrieves passages for a theme and produces questions as **validated structured output** (Pydantic):
  ```json
  {
    "question": "string",
    "answer": "string",
    "accepted_aliases": ["string"],
    "difficulty": "easy | medium | hard",
    "category": "string",
    "source_chunk_ids": ["uuid"],
    "source_quote": "string"
  }
  ```
- [ ] **AI-6** Invalid model output triggers retry with error feedback (max N retries), then fails gracefully.
- [ ] **AI-7** **Round-builder agent** with tools: `search_corpus`, `check_duplicate`, `fact_check`, `balance_difficulty`. Produces a balanced round with no repeated topics.
- [ ] **AI-8** **Fact-check guardrail:** each question's answer must be supported by its cited passage (LLM-as-judge with a grounding prompt). Unsupported or ambiguous questions are rejected and logged with a reason.
- [ ] **AI-9** Rejection rate per run is recorded.

### 4.3 Answer marking pipeline
Layered to minimise cost and latency:
1. [ ] **AI-10** Normalise (case, punctuation, articles like "the", whitespace).
2. [ ] **AI-11** Exact / alias match → accept.
3. [ ] **AI-12** Fuzzy match (e.g. rapidfuzz) above threshold → accept; clearly below → reject.
4. [ ] **AI-13** Ambiguous band only → LLM judge returns `{ "correct": bool, "reason": str, "confidence": float }`.
- [ ] **AI-14** Each mark records which layer decided it.

### 4.4 Host personality (stretch)
- [ ] **AI-15** Optional short banter between rounds based on the scores. Must stay friendly; a configurable tone setting.

### 4.5 Prompt management
- [ ] **AI-16** Prompts live in versioned files (`prompts/*.md` or `.yaml`), not inline strings.
- [ ] **AI-17** Every LLM call logs the prompt version used.
- [ ] **AI-18** Model provider is abstracted behind an interface so models can be swapped/compared.

---

## 5. Evaluation Requirements

- [ ] **EV-1 Marking eval set:** ≥200 labelled cases `(question, correct_answer, player_answer, should_accept)` including typos, partial answers, aliases, and wrong-but-close answers.
- [ ] **EV-2 Question quality eval set:** ≥100 generated questions labelled for factual correctness, ambiguity, and answerability.
- [ ] **EV-3** Eval runner CLI outputs accuracy, precision, recall, and per-layer breakdown as JSON + Markdown table.
- [ ] **EV-4** Evals run in **CI**; build fails if marking accuracy drops below a threshold (e.g. 95%).
- [ ] **EV-5** **Model comparison:** run evals across 2–3 models; report accuracy, p50/p95 latency, and cost per game.
- [ ] **EV-6** Host overrides from real games can be exported into the eval set.
- [ ] **EV-7** Results summary published in README.

**Target headline metrics**
- Marking accuracy ≥ 95% on EV-1
- Fact-check guardrail catches ≥ 90% of known-bad questions
- Cost per 40-question game < £0.10
- Marking latency p95 < 1.5s

---

## 6. Observability

- [ ] **OB-1** Every LLM call logged: timestamp, feature, model, prompt version, input/output tokens, latency, cost, success/failure.
- [ ] **OB-2** Stored in Postgres (or Langfuse); queryable.
- [ ] **OB-3** Admin dashboard page: cost per game, calls by feature, latency percentiles, guardrail rejection rate, error rate.
- [ ] **OB-4** Structured application logging (JSON) with request/game IDs.

---

## 7. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NF-1 | Support ≥50 concurrent player connections per game without lag |
| NF-2 | State updates reach all clients < 300ms on a normal connection |
| NF-3 | Player view is mobile-first and works on iOS Safari + Android Chrome |
| NF-4 | No secrets in the repo; config via environment variables |
| NF-5 | Rate-limit joins and answer submissions per player |
| NF-6 | Player input sanitised; free-text answers can't inject into LLM prompts (delimited + instructed as data) |
| NF-7 | Backend test coverage ≥ 70% on core game + marking logic |
| NF-8 | Accessible: readable font sizes on big screen, colour-contrast compliant, keyboard-usable host view |

---

## 8. Tech Stack

| Layer | Choice |
|-------|--------|
| Backend | Python 3.12, **FastAPI**, WebSockets, Pydantic v2 |
| Database | **PostgreSQL 16 + pgvector**, SQLAlchemy 2 + Alembic migrations |
| Cache / pub-sub | Redis (for multi-instance WebSocket fan-out; in-memory acceptable for v1) |
| Frontend | **React + TypeScript (Vite)**, Tailwind, Framer Motion for leaderboard animation |
| LLM | Provider-agnostic client; embeddings via provider API |
| Fuzzy match | rapidfuzz |
| TTS | Browser SpeechSynthesis (v1) → cloud TTS (stretch) |
| Testing | pytest, Vitest, Playwright (one end-to-end game flow) |
| CI/CD | GitHub Actions: lint, type-check, tests, evals, build, deploy |
| Infra | Docker + docker-compose locally; **AWS** (ECS Fargate or App Runner + RDS Postgres) or Azure equivalent |
| Observability | Postgres logging table + dashboard, or Langfuse |

---

## 9. Data Model (initial)

- `games` — id, code, status, config (JSON), created_at
- `teams` — id, game_id, name, session_token, score
- `rounds` — id, game_id, index, theme, multiplier
- `questions` — id, round_id, text, answer, aliases[], difficulty, source_chunk_ids[], fact_check_status, fact_check_reason
- `answers` — id, question_id, team_id, text, is_correct, decided_by (exact/fuzzy/llm/host), reason, submitted_at
- `documents` — id, source_url, title, theme
- `chunks` — id, document_id, text, embedding (vector), tsvector
- `llm_calls` — id, feature, model, prompt_version, tokens_in, tokens_out, latency_ms, cost, status, created_at
- `eval_runs` — id, suite, model, prompt_version, metrics (JSON), created_at

---

## 10. Repo Structure

```
.
├── backend/
│   ├── app/
│   │   ├── api/            # REST + WebSocket routes
│   │   ├── game/           # game state machine, scoring
│   │   ├── ai/
│   │   │   ├── ingest/     # corpus loading, chunking, embedding
│   │   │   ├── generate/   # question generation + round-builder agent
│   │   │   ├── guardrails/ # fact-check
│   │   │   ├── marking/    # layered marking pipeline
│   │   │   └── llm/        # provider abstraction, logging
│   │   ├── db/             # models, migrations
│   │   └── observability/
│   ├── prompts/
│   └── tests/
├── frontend/
│   └── src/
│       ├── host/
│       ├── screen/
│       └── play/
├── evals/
│   ├── datasets/
│   ├── run_evals.py
│   └── results/
├── infra/                  # Dockerfiles, compose, IaC
├── .github/workflows/
├── REQUIREMENTS.md
└── README.md
```

---

## 11. Milestones

### M1 — Playable skeleton
- [ ] docker-compose with FastAPI, Postgres, React
- [ ] Create/join game, three views synced over WebSockets
- [ ] Hardcoded questions, manual marking, leaderboard
- **Done when:** a 3-phone test game runs end to end.

### M2 — RAG question generation
- [ ] Ingestion pipeline + pgvector
- [ ] Structured question generation with citations
- [ ] Host review/approve UI
- **Done when:** host can generate a themed round with sources shown.

### M3 — Guardrails & agent
- [ ] Fact-check guardrail with rejection logging
- [ ] Round-builder agent with tools
- **Done when:** known-bad questions are caught and rounds are balanced.

### M4 — Marking pipeline
- [ ] Normalise → alias → fuzzy → LLM layers
- [ ] Host override UI, overrides stored
- **Done when:** marking is automatic, with the deciding layer shown.

### M5 — Evals & observability
- [ ] Both eval sets built; eval runner CLI
- [ ] Evals in CI with a threshold gate
- [ ] Model comparison report
- [ ] LLM call logging + dashboard
- **Done when:** README shows real metrics and the dashboard shows cost per game.

### M6 — Production & demo
- [ ] Deploy to AWS/Azure with RDS/managed Postgres
- [ ] GitHub Actions deploy pipeline
- [ ] TTS, animations, polish
- [ ] Run a real quiz night and record a 2–3 minute demo video
- **Done when:** live URL + video are in the README.

### Stretch
- [ ] Picture round
- [ ] Music intro round (audio clips)
- [ ] Joker card (double one round per team)
- [ ] Host personality banter
- [ ] Cloud TTS voices

---

## 12. README Must Include (for portfolio)
- [ ] One-paragraph pitch + demo GIF/video
- [ ] Architecture diagram
- [ ] Eval results table and model comparison chart
- [ ] Cost per game and latency figures
- [ ] "Design decisions & trade-offs" section (why layered marking, why pgvector, guardrail approach)
- [ ] Known limitations and what you'd do next
- [ ] Local setup instructions
