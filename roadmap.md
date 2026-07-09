# Build Roadmap: Resources & AI-Assistance Guide

Companion document to the specifications doc. Structured by the 4-week timeline. For each phase: **what to build**, **resources to learn from**, and **exactly how to use AI so you stay the one making decisions** (per the "must-understand vs. can-delegate" split we agreed on).

General rule threaded through every stage: ask AI for *explanations, tradeoffs, and reviews* on the "must-understand" items; ask it for *code directly* on boilerplate. The specific prompts below are written to reflect that.

---

## Week 1 — Foundation

### Days 1–2: Data export & cleaning

**Task:** Export Discord channels and Google Docs, define the schema, write a parsing script to structure raw exports into seed data.

**Resources:**
- Discord export tools: search for "DiscordChatExporter" (open-source, widely used, exports to JSON/HTML/CSV)
- Google Docs API (if you want programmatic export rather than manual copy-paste): `developers.google.com/docs/api`
- Python `json` and `re` modules — standard library docs at `docs.python.org/3/library/json.html`

**How to use AI here:**
- This is a "write it yourself first" zone — it's Python, your strength, and a good opportunity to practice without a safety net.
- Good prompt *after* you've written a first pass: *"Here's my parsing script for converting Discord export JSON into structured lore entries. What edge cases am I likely missing?"*
- Avoid: *"Write me a script to parse this Discord export."* You'd lose the one part of Week 1 that's squarely in your comfort zone to practice unassisted.

### Day 2 (continued): Schema design

**Task:** Finalize the `User`, `Role`, `Race`, `Item`, `NPC`, `Location`, `LoreEntry`, `Character` tables and relationships.

**Resources:**
- PostgreSQL data types: `postgresql.org/docs/current/datatype.html`
- SQLAlchemy ORM docs (if using it with FastAPI): `docs.sqlalchemy.org`
- General relational design refresher if rusty: search "database normalization basics"

**How to use AI here:**
- Design the schema yourself on paper/whiteboard first — this is exactly the kind of decision interviewers ask about.
- Good prompt: *"Here's my schema for a D&D campaign app [paste tables/relationships]. What are the tradeoffs of using a JSON column for character stats vs. a separate relational table? Poke holes in my design."*
- Avoid: *"Design a schema for a D&D campaign app."* You want to defend your schema, not someone else's.

### Day 3: Auth integration

**Task:** Set up managed auth provider (Clerk or Auth0 recommended), role-based route protection in FastAPI.

**Resources:**
- Clerk docs: `clerk.com/docs` (has a dedicated FastAPI/Python integration guide)
- Auth0 docs: `auth0.com/docs` (also has Python quickstarts)
- FastAPI security docs: `fastapi.tiangolo.com/tutorial/security/`

**How to use AI here:**
- This is security-adjacent — "must-understand" territory, but the *integration* boilerplate (SDK setup, middleware wiring) is reasonable to delegate since it's largely provider-specific plumbing, not a design decision.
- Good prompt: *"I'm integrating Clerk with FastAPI for role-based auth. Walk me through what each piece of this middleware is actually doing before I add it"* — get the explanation before you paste code in.
- Follow-up you should ask yourself, not AI: after it's wired up, try to explain out loud what happens, step by step, when a request with an invalid token hits a protected route. If you can't, go back and read the explanation again.

### Days 4–5: CRUD endpoints + tests

**Task:** Build FastAPI CRUD routes for all entities, enforce DM-only write access server-side, write pytest coverage.

**Resources:**
- FastAPI tutorial (full walkthrough): `fastapi.tiangolo.com/tutorial/`
- pytest docs: `docs.pytest.org`
- FastAPI testing guide specifically: `fastapi.tiangolo.com/tutorial/testing/`

**How to use AI here:**
- This is your strength zone and largely boilerplate — fine to move fast with AI assistance on route scaffolding.
- Good prompt: *"Generate CRUD routes for this Item model in FastAPI, following REST conventions"* — then read every route before moving on, since you're still responsible for the permission logic layered on top.
- Keep the permission-check code itself in the "write yourself, then verify" bucket: *"Here's my ownership check for Character edits — can a player edit someone else's character with this logic? Try to break it."*

---

## Week 2 — Frontend (React/Next.js)

### Days 1–2: Ramp-up (dedicated learning days, no real feature work)

**Resources:**
- Official React docs (the newer "learn" section is excellent for beginners): `react.dev/learn`
- Next.js official tutorial: `nextjs.org/learn`
- Focus specifically on: components/props, `useState`, `useEffect`, fetching data, basic App Router navigation

**How to use AI here:**
- Use AI as a *tutor*, not a code generator, for these two days.
- Good prompt: *"I just learned about useEffect for data fetching. Give me a small broken example and let me try to fix it before you show me the solution."*
- Good prompt: *"Explain the difference between server components and client components in Next.js like I already know React class components but not hooks."* (calibrate to your actual background)
- Avoid building your throwaway practice components by asking AI to write them outright — type them yourself, even copying from docs, so the syntax becomes familiar through repetition.

### Days 3–5: Real frontend build

**Task:** Browse/detail/search pages, login flow, role-aware UI, character view/edit.

**Resources:**
- Next.js data fetching docs: `nextjs.org/docs/app/building-your-application/data-fetching`
- Clerk's Next.js integration guide (if using Clerk): `clerk.com/docs/quickstarts/nextjs`

**How to use AI here:**
- Component structure and page layout: fine to ask AI directly, this is largely boilerplate at your current stage.
- Good prompt: *"Build a Next.js page that fetches and displays a list of items from this API endpoint, with a search filter"* — then trace through the code once to make sure you understand the data flow (fetch → state → render).
- Keep role-aware UI logic (showing/hiding edit controls based on role) in the "explain first" bucket, since it pairs with the server-side permission checks from Week 1 — you want to genuinely understand that the UI check is cosmetic and the API check is the real boundary, not just paste code that happens to work.

---

## Week 3 — AI Engineering

This week is your project's centerpiece — lean toward "write/design yourself, use AI to review and implement details" more than any other week.

### Days 1–2: Embedding pipeline + RAG endpoint

**Task:** Chunk lore text, embed with `sentence-transformers`, store in pgvector, build the `/ask` endpoint with local Ollama.

**Resources:**
- `sentence-transformers` docs: `sbert.net`
- `pgvector` GitHub repo and README (has SQL examples): `github.com/pgvector/pgvector`
- Ollama docs: `ollama.com` (has API docs for local generation)
- A good conceptual primer if RAG is new to you: search "RAG retrieval augmented generation explained" and read 1–2 explainer articles before implementing

**How to use AI here:**
- Design the chunking strategy yourself first (chunk size, overlap, what counts as one "document") — this is a real decision with real tradeoffs, and a common interview question.
- Good prompt: *"I'm chunking lore text at ~400 tokens with 50-token overlap for embeddings. What are the failure modes of this approach, and when would smaller/larger chunks matter more?"*
- Good prompt for the retrieval/generation glue code (more delegable): *"Show me how to write a cosine similarity query in pgvector given an embedding vector"* — implementation detail, fine to take directly, but make sure you understand what "cosine similarity" is actually measuring before moving on.
- Avoid asking AI to design the whole pipeline end-to-end in one prompt — you'll end up with someone else's architecture and won't be able to defend the choices in an interview.

### Day 3: Chat UI

**Task:** Build the "Ask the Lorekeeper" chat component.

**Resources:**
- Reuse Next.js data-fetching patterns from Week 2
- If you want streaming responses: `fastapi.tiangolo.com` streaming response docs + `react.dev` docs on handling async UI state

**How to use AI here:**
- This is mostly React boilerplate at this point — fine to build with heavier AI assistance, since the interesting decisions already happened in Days 1–2.

### Day 4: RAG evaluation

**Task:** Build a hand-written eval set, measure retrieval and answer quality.

**Resources:**
- Look up "RAG evaluation metrics" for a sense of what people measure (relevance, faithfulness/groundedness, answer correctness) — you don't need a formal framework, but understanding the vocabulary matters for talking about it later
- Anthropic's and OpenAI's docs both have brief practical guides on evaluating LLM outputs if you search their respective docs sites for "evaluation"

**How to use AI here:**
- Write your 10–20 eval questions yourself, based on lore you actually know — this is the part that proves you understand your own system.
- Good prompt: *"Here are my eval questions and the answers my RAG system gave. Help me design a simple scoring rubric to rate these consistently"* — delegate the scoring *mechanism*, not the judgment calls about what "good" looks like for your specific content.
- Write up the results and your interpretation of them yourself. This becomes your most talkable interview material — don't outsource the "so what did you learn from this" part.

### Day 5: Provider abstraction

**Task:** Build the `generate_answer()` abstraction layer, test swapping Ollama for a hosted API.

**Resources:**
- Anthropic API docs: `docs.claude.com` (Messages API)
- OpenAI API docs: `platform.openai.com/docs`

**How to use AI here:**
- Good prompt: *"I want a single function that can call either a local Ollama model or the Anthropic API depending on a config flag, without changing any calling code. What's a clean way to structure this?"* — a legitimate design-pattern question (this is basically the strategy/adapter pattern), worth understanding rather than just accepting.

---

## Week 4 — Integration, Deployment, Polish

### Days 1–2: Deploy

**Resources:**
- Vercel deployment docs (Next.js): `vercel.com/docs`
- Railway docs: `docs.railway.com`
- Render docs (alternative to Railway): `render.com/docs`

**How to use AI here:**
- Deployment configuration and debugging is a great place to lean on AI heavily — it's mostly platform-specific plumbing, not conceptual.
- Good prompt when something breaks: *"I'm getting this CORS error when my Next.js frontend calls my FastAPI backend on Railway [paste error]. What's happening and how do I fix it?"* — ask for the explanation alongside the fix so you're not just copy-pasting config blindly.

### Day 3: Stretch goals

Use judgment based on how the first three weeks went — no fixed resources here.

### Day 4: Documentation

**Task:** README, architecture diagram, resume bullets.

**How to use AI here:**
- Write the "what I learned" and "what I'd do differently" sections yourself — this is reflection, and it's also exactly what you'll be asked about in an interview, so writing it forces you to actually form the opinions now rather than fumbling for them later.
- Fine to ask AI to help with formatting/structure of the README itself, or to draft an architecture diagram in Mermaid syntax for you to refine.

### Day 5: Demo video + buffer

No specific resources needed — just record yourself walking through the app.

---

## A running practice for the whole project

Keep a `DECISIONS.md` file (mentioned in our last conversation) updated as you go — one line per significant choice and why. At the end of each week, spend 10 minutes re-reading your entries from that week. If any of them feel shaky to explain, that's a signal to go back and solidify your understanding *before* moving to the next phase, not after the whole project is done.
