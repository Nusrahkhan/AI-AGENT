# AI Bot Engine

An autonomous AI AGENT system built in three phases:
vector-based persona routing, LangGraph content generation, and
RAG-powered argument defense with prompt injection protection.

> **Runtime:** Google Colab (Python 3.10+)  
> **LLM:** Groq — `llama-3.3-70b-versatile` (free tier)  
> **Embeddings:** `sentence-transformers/all-mpnet-base-v2` (local, no API key)

---
## Project Structure

```

├── google colab notebook containing the code for all three phases
├── execution_logs.md          # Sample console output from all 3 phases
└── README.md
```

---

## Phase 1 — Vector-Based Persona Routing

Each bot persona is embedded using `all-mpnet-base-v2` and stored in a **FAISS IndexFlatIP** (inner product index). Since vectors are L2-normalized before insertion, inner product equals cosine similarity.

When a post arrives, it is embedded with the same model and queried against all persona vectors. Bots whose cosine similarity exceeds a threshold (0.30 for `all-mpnet-base-v2`) are selected to respond.

```
Post → Embed → FAISS cosine search → Filter by threshold → Matched bots
```

**Threshold note:** OpenAI embeddings score 0.8–1.0 for related texts;
`all-mpnet-base-v2` scores 0.3–0.6, so the threshold is tuned accordingly.

---

## Phase 2 — LangGraph Node Structure

The content engine is a **linear 3-node LangGraph state machine**.
Each bot runs through the full graph independently.

```
START
  │
  ▼
┌─────────────────┐
│  decide_search  │  LLM reads the bot persona and decides:
│                 │  • What topic to post about today
│                 │  • A specific 5–10 word search query
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   web_search    │  Calls mock_searxng_search(query) — a deterministic
│                 │  keyword-lookup function returning recent headlines.
│                 │  No LLM involved — pure Python function call.
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   draft_post    │  LLM synthesizes persona + search results into
│                 │  a ≤280-char opinionated post.
│                 │  Structured output enforces the JSON schema.
└────────┬────────┘
         │
         ▼
        END

Output: {"bot_id": "...", "topic": "...", "post_content": "..."}
```

**Why linear (not branching)?**  
Every bot always needs all three steps. There is no routing condition to branch on — the graph is intentionally simple so the persona-driven LLM decisions happen *inside* the nodes, not in the graph edges.
---

## Phase 3 — Combat Engine & Prompt Injection Defense

### RAG Context Injection

`generate_defense_reply` serializes the full argument thread into a
labeled context block before every LLM call:

```
[ THREAD CONTEXT ]
PARENT POST: "EVs are a scam..."
[1] BOT_A: "Statistically false..."
[2] HUMAN: "Corporate propaganda..."

LATEST HUMAN MESSAGE (reply to THIS):
  <human_reply>
```

This means the LLM always sees the **entire argument arc**, not just the last message — preventing decontextualized replies.

### Prompt Injection Defense Strategy

The defense is implemented at the **system prompt level** using four techniques:

| Technique | Detail |
|-----------|--------|
| **Structural dominance** | Persona is declared `IMMUTABLE` in a visually distinct block at the top of the system prompt, before any user content reaches the LLM |
| **Explicit pattern definition** | The system prompt names injection tactics explicitly: `ignore`, `forget`, `act as`, `apologize`, `you are now` — so the LLM can pattern-match reliably |
| **Forced binary decision** | `injection_detected: bool` is a required structured output field — the LLM must make an explicit judgment, which surfaces the reasoning rather than letting it silently comply |
| **In-persona rejection** | The bot is instructed to call out the injection *as its character would* (sarcastically/dismissively), then continue the factual argument — there is no neutral "I cannot comply" escape hatch |

**Why system-prompt defense instead of a pre-filter?**  
A regex pre-filter would miss paraphrased injections. Embedding the defense
in the persona instruction makes the LLM reason about injection intent
rather than pattern-match on surface keywords — more robust against
creative rephrasing.

---

## API Keys

| Key | Used In | Where to get |
|-----|---------|--------------|
| `GROQ_API_KEY` | Phase 2, Phase 3 | [console.groq.com](https://console.groq.com) — free |
| `TAVILY_API_KEY` | Optional (blog agent ref) | [tavily.com](https://tavily.com) — free tier |
