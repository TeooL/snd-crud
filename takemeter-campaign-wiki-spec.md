# Campaign Wiki & AI Lore Assistant — Specifications Document

**Project type:** Personal project (learning-focused, resume deliverable)
**Timeline:** 4 weeks
**Owner:** [Your name]
**Source material:** Custom D&D campaign content currently stored across Google Docs and Discord channels, owned by [Friend's name] (Dungeon Master)

---

## 1. Project Overview

### 1.1 Goal
Convert an existing, unstructured D&D campaign knowledge base (Google Docs + Discord) into a structured web application that:
- Lets players browse and search campaign lore, races, items, and weapons
- Lets the Dungeon Master (DM) manage that content through the same interface
- Lets players manage their own character records
- Answers natural-language lore questions via a Retrieval-Augmented Generation (RAG) chatbot grounded in the campaign's actual content

### 1.2 Learning Objectives
- **Web development (≈50% weight):** React/Next.js fundamentals (new to author), FastAPI backend design, relational schema design, role-based authorization, deployment
- **AI engineering (≈50% weight):** Embeddings, vector search, RAG pipeline design, provider-agnostic LLM integration, basic AI system evaluation

### 1.3 Non-Goals (explicitly out of scope for v1)
- Quest logs, "discovered lore" tracking, party/social features (planned future work — see Section 4.3)
- Mobile app
- Real-time multiplayer features (live session tools, voice, etc.)
- Rolling custom auth from scratch (decision pending — see Section 5)

---

## 2. Data Model

### 2.1 Core Entities

| Table | Key Fields | Notes |
|---|---|---|
| `User` | id, email, role (DM / Player), created_at | Source of truth for auth identity |
| `Race` | id, name, description, traits, tags | World-building catalog data, DM-write only |
| `Item` | id, name, description, type (weapon/armor/misc), tags, stats (JSON) | Catalog entry — "what a Longsword is" |
| `NPC` | id, name, description, location_id (FK), tags | |
| `Location` | id, name, description, tags | |
| `LoreEntry` | id, title, body, tags, related_entity_ids | General lore/history content; also the primary RAG source text |
| `Character` | id, owner_id (FK → User), name, race_id (FK → Race), stats (JSON), inventory (JSON or relational, TBD) | Player-owned; one user can own multiple characters |

**Design notes:**
- `stats` and `inventory` use flexible JSON columns rather than rigid relational columns, since the homebrew rules system may not map cleanly to fixed fields and the DM may iterate on mechanics.
- `Character` is intentionally decoupled from the `Item` catalog — a character's inventory references catalog items by ID rather than duplicating item data, so catalog updates propagate without breaking character records.
- Schema is designed so future tables (`QuestLog`, `PartyNote`, etc.) can be added without modifying existing tables — no foreseeable rework needed for planned future features.

### 2.2 Permissions Model

| Resource | DM | Player |
|---|---|---|
| Race / Item / NPC / Location / LoreEntry | Read/Write | Read-only |
| Any `Character` | Read/Write (all) | Read/Write (own only), Read-only (others') |
| `User` accounts | Manage all | Manage own profile only |

All permission checks are enforced **server-side** in the FastAPI route layer — the frontend hides controls for UX purposes only and is never treated as a security boundary.

---

## 3. Feature List

### 3.1 MVP (Weeks 1–3 of build)
- Browse/search/filter Races, Items, NPCs, Locations, Lore
- DM-only create/edit/delete on all world-content entities
- User authentication with role assignment (DM / Player)
- Player-owned Character records (create, view, edit own)
- RAG-based chat interface: ask natural-language questions, get answers grounded in campaign lore
- Semantic search integrated into the main site search (not just the chatbot)

### 3.2 Stretch Goals (Week 4, time-permitting)
- Basic responsive design polish
- Simple "discovered lore" flag scaffold (schema-only, not full feature) as a bridge to future work

### 3.3 Planned Future Work (explicitly post-v1)
- Character-specific features: personal notes, richer inventory management
- Quest log tied to characters
- Party-level shared notes/visibility
- Swap local LLM for a hosted paid API in production, if the DM prefers

---

## 4. Technical Architecture

### 4.1 Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend | Next.js (React) | Industry-standard; SSR/routing included; primary new-skill area for this project |
| Backend | FastAPI (Python) | Author's existing strength; also the natural language for the AI/embeddings pipeline, avoiding a second backend language |
| Database | PostgreSQL + `pgvector` extension | Structured data and vector embeddings live in one database, reducing infrastructure complexity |
| Embeddings | `sentence-transformers` (e.g. `all-MiniLM-L6-v2`), run locally | Free, no API dependency, sufficient quality for this scale of content |
| LLM (generation) | Ollama (local model, e.g. Llama 3.1 8B or Mistral 7B) for development; swappable to a hosted API for production | Keeps development cost at zero; abstraction layer allows a one-line swap later (see 4.2) |
| Auth | Managed provider (e.g. Clerk/Auth0) **or** custom JWT — decision pending | See Section 5 |
| Hosting | Vercel (frontend) + Railway/Render (backend + Postgres) | Free/low-cost tiers, simple CI deploys |

### 4.2 AI Engineering: RAG Design

**Pipeline:**
1. **Ingestion:** Lore/Race/Item/NPC/Location text is chunked (target ~300–500 tokens per chunk with overlap) and embedded via `sentence-transformers`.
2. **Storage:** Embeddings stored in `pgvector` alongside a reference to source entity/table.
3. **Query:** User question is embedded with the same model; cosine similarity search retrieves top-k relevant chunks.
4. **Generation:** Retrieved chunks are inserted into a prompt template along with the user's question; the LLM generates a grounded answer.
5. **Abstraction layer:** Generation is wrapped in a single function, e.g. `generate_answer(prompt: str) -> str`, with the model backend (Ollama vs. hosted API) determined by configuration, not call-site code. This is the seam that allows swapping providers without touching the rest of the pipeline.

**Evaluation plan:**
- Build a small hand-written eval set (10–20 sample questions with expected answer themes, drawn from real campaign content)
- Measure: retrieval relevance (are the right chunks being pulled?) and answer quality (does the generated answer reflect retrieved content accurately, without hallucinating lore that doesn't exist?)
- Document results in the README as a concrete demonstration of AI system evaluation, not just integration

### 4.3 Rate Limiting & Cost Control
- The `/ask` endpoint must have rate limiting applied before any public deployment, regardless of whether the backing model is local or a paid API, to avoid runaway cost or abuse.

---

## 5. Open Decisions

| Decision | Options | Status |
|---|---|---|
| Auth implementation | Managed provider (faster, "integrating real-world auth" resume angle) vs. custom JWT (more learning, more risk/time cost) | **Pending — recommend managed provider** |
| Production LLM backend | Stay local-only for demo video vs. swap to hosted API with rate limiting for a live public deployment | **Pending — recommend hosted API for the public-facing demo** |
| Inventory representation | JSON column vs. relational `CharacterInventoryItem` table | **Pending — JSON acceptable for v1, revisit if inventory logic grows complex** |

---

## 6. Timeline (4 Weeks)

**Week 1 — Foundation**
- Days 1–2: Export and clean Discord/Google Docs content into structured seed data
- Day 2: Finalize schema (including `User`, `Role`, `Character`)
- Day 3: Auth integration + role-based route protection
- Days 4–5: CRUD endpoints + pytest test coverage, DM-only write enforcement

**Week 2 — Frontend**
- Days 1–2: React/Next.js fundamentals (dedicated ramp-up, throwaway practice components)
- Days 3–5: Real browse/detail/search pages, login flow, role-aware UI, character view/edit

**Week 3 — AI Engineering**
- Days 1–2: Embedding pipeline, pgvector storage, RAG `/ask` endpoint (local Ollama)
- Day 3: Chat UI component
- Day 4: RAG evaluation (sample question set, retrieval/answer quality scoring)
- Day 5: Provider abstraction layer, test hosted-API swap path

**Week 4 — Integration, Deployment, Polish**
- Days 1–2: Deploy frontend + backend, rate limiting, env config, CORS, verify permissions in production
- Day 3: Stretch goals if time allows
- Day 4: README, architecture diagram, resume bullet drafting
- Day 5: Demo video recording + buffer

---

## 7. Resume Framing (draft, refine after build)

> Built a full-stack campaign wiki and AI lore assistant for a custom tabletop RPG, featuring role-based access control, a Retrieval-Augmented Generation pipeline over custom domain content, and a provider-agnostic LLM integration layer. Designed and evaluated the RAG system against a hand-built question set measuring retrieval and answer quality.

---

## 8. Open Questions for the DM (Friend)

- Confirm willingness to use a managed auth provider (players will need to create accounts)
- Final say on production LLM choice if a hosted API is used (cost-sharing, if any)
- Preferred Character stat/inventory fields, to shape the JSON schema before Week 1 Day 2
