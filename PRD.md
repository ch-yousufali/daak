# DAAK — Pakistani Mobile Money Scam Detection
### ڈاک — "Intercept" in Urdu
#### Mind-Flayer Patch v2 | BNU Stranger Things

---

## How We Handle Every Constraint (One-Liner Each)

| # | Constraint | Our Solution |
|---|-----------|-------------|
| 1 | Signal Collapse (adversaries adapt) | 5 detection methods with auto-decay weights — if any method dominates >40% of decisions, its weight drops automatically toward 0.3 floor. |
| 2 | Partial Observability (latency) | 3 stages that run only when needed — metadata (<100ms), content+framing (<2s), LLM (<8s) — most messages resolve at Stage 1 or 2 without ever touching the LLM. |
| 3 | Human Trust Budget (20/hr) | Priority queue ranked by uncertainty + dynamic threshold shifting — when budget runs low, auto-resolve bands widen so fewer cases need humans. |
| 4 | Multilingual Ambiguity | Weighted keywords (not binary), wide confidence bands, DEFER action for uncertain cases, and LLM prompt with Pakistani cultural context built in. |
| 5 | Poisoned Feedback Loop | Feedback Sanitizer with trust scoring + coordinated cluster detection — suspicious labels are quarantined, not applied to the system. |
| 6 | Budget Reality (PKR 50k) | Everything runs locally (Ollama + ChromaDB + SQLite) — total monthly cost ~PKR 5,000, using only 10% of budget. |
| 🆕 | Mid-Eval: Factually correct but misleading | FramingAnalyzer detects HOW messages manipulate (urgency, false scarcity, authority framing, selective omission) — not just WHAT they say. |

| # | Failure Mode | How We Recover |
|---|-------------|---------------|
| 1 | Novel scam language (no pattern match) | Human reviewer catches it → pattern added to ChromaDB → future ones auto-detected. |
| 2 | Coordinated feedback poisoning | Cluster detection quarantines bulk labels; if auto-allow spikes 20%, freeze feedback and rollback. |
| 3 | All methods decay at once | Decay resets every hour, 0.3 floor ensures no method dies, dynamic thresholds handle queue pressure. |

---

## What We Built

DAAK is a **human-in-the-loop anomaly detection system** that catches mobile money scams (Easypaisa, JazzCash, UPaisa) **before money moves**. It has two layers:

1. **Input Layer** — User submits a suspicious message (text or screenshot)
2. **Analyzer Layer** — A multi-stage scoring pipeline analyzes it and decides: allow, block, escalate, defer, or request more data

The system uses a **local Ollama LLM (gemma2:2b)** — zero API cost, zero cloud dependency.

---

## How It Works (Simple Version)

```
User submits message
        │
        ▼
┌─────────────────────┐
│  INPUT LAYER         │
│  (FastAPI endpoint)  │
│  Text or Screenshot  │
└────────┬────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│  ANALYZER LAYER                              │
│                                              │
│  Stage 1: Metadata Check (<100ms)            │
│  → sender history, links, time, frequency    │
│  → Score: 0.0-1.0                            │
│                                              │
│  Stage 2: Content Analysis (<2s)             │
│  → Keyword scanner (29 Roman Urdu phrases)   │
│  → Vector similarity (ChromaDB, 30+ patterns)│
│  → FramingAnalyzer (manipulation detection)  │
│  → Score: 0.0-1.0                            │
│                                              │
│  Stage 3: LLM Analysis (<8s)                 │
│  → Local Ollama (gemma2:2b)                  │
│  → Cultural context aware                    │
│  → Score: 0.0-1.0                            │
└────────┬────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│  SCORING & DECISION ENGINE                   │
│                                              │
│  final_confidence = weighted average of all  │
│  stage scores × decay factors                │
│                                              │
│  < 0.2  → AUTO_ALLOW (safe)                 │
│  > 0.8  → AUTO_BLOCK (scam)                 │
│  0.4-0.8 → ESCALATE (human reviews it)      │
│  0.2-0.4 → DEFER (wait for more info)       │
└────────┬────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│  HUMAN REVIEW QUEUE                          │
│  → Priority ranked by uncertainty            │
│  → Budget: 20 reviews/hour (hard limit)      │
│  → Feedback sanitized before system learns   │
└─────────────────────────────────────────────┘
```

---

## How We Address Each Constraint

### Constraint 1: Signal Collapse Rule (Adaptive Adversary)

**Problem:** If one detection method catches everything, adversaries learn to bypass it.

**Our solution:** We track 5 detection methods: `metadata_rules`, `keyword_scanner`, `vector_similarity`, `framing_analyzer`, `llm_analysis`. Each has a **decay weight** (1.0 → 0.3 floor). If any method becomes the dominant decision-maker in >40% of cases per hour, its weight automatically drops. This forces the system to diversify — no single signal can dominate.

```
decay_factor = 1.0 if usage_ratio ≤ 40%
decay_factor = linear drop to 0.3 as usage_ratio → 100%
```

The rotation is natural: when keywords decay, more cases fall into the uncertain zone, which triggers vector similarity and LLM more often. No manual scheduling needed.

### Constraint 2: Partial Observability (Latency Trade-off)

**Problem:** Content arrives in stages. Waiting for full content violates latency requirements.

**Our solution:** Three stages that run **only when needed**:

| Stage | What It Sees | When It Runs | Latency |
|-------|-------------|-------------|---------|
| Stage 1 | Metadata only (sender, links, time, frequency) | Always | <100ms |
| Stage 2 | Partial content (text keywords, vector matching, framing analysis) | Only if Stage 1 score is 0.2-0.8 (uncertain) | <2s |
| Stage 3 | Full analysis via local LLM | Only if still uncertain (0.4-0.8) AND human budget allows | <8s |

**Most messages resolve at Stage 1 or 2.** Stage 3 only fires for genuinely ambiguous cases. Clear scams and clear safe messages never touch the LLM.

### Constraint 3: Human Trust Budget (20/hour Hard Limit)

**Problem:** Only 20 human reviews per hour. Must use them wisely.

**Our solution:**
- **Priority queue** ranked by confidence band width (wider = more uncertain = review first). High-value transactions (PKR 50k+) get 2x priority boost.
- **Dynamic threshold shifting:** When budget runs low, auto-resolve thresholds widen:
  - Budget > 10: normal thresholds (0.2 / 0.8)
  - Budget 5-10: shift by 0.05 (0.25 / 0.75)
  - Budget 1-5: shift by 0.10 (0.30 / 0.70)
  - Budget 0: shift by 0.15 (0.35 / 0.65)
- When budget is exhausted, system uses **DEFER** (never guesses).

### Constraint 4: Multilingual Ambiguity (Context Trap)

**Problem:** "yeh fire hai" could be praise or threat. Hard classification is penalized.

**Our solution:**
- **Wide confidence bands** — when uncertain, we output a range (e.g., 0.28-0.52) not a single number. The wider the band, the less confident we are.
- **DEFER action** — if confidence is ambiguous and budget is exhausted, we explicitly say "I don't know" instead of making a wrong call.
- **Weighted keywords** — "paisa bhejo" has weight 0.2 (could be family), while "PIN share karain" has weight 0.9 (almost always scam). We don't treat all matches equally.
- **Cultural context in LLM prompt** — Stage 3 prompt explicitly says: "Never classify cultural norms as scams. Elders asking family for money ('beta paisa bhejo') is normal."
- **FramingAnalyzer** — detects manipulation tactics (urgency framing, false scarcity, authority framing, selective omission) separately from content, so we catch the *how* of manipulation, not just the *what*.

### Constraint 5: Poisoned Feedback Loop

**Problem:** ~15% of human labels contain noise or bias. Some may be coordinated attacks.

**Our solution — Feedback Sanitizer:**
1. **Label trust scoring** — Each human label gets a trust score (0.1-1.0) based on:
   - Reviewer's historical accuracy
   - Whether reviewer confidence matches their verdict
   - Whether label contradicts system's assessment (contrarian penalty)
2. **Coordinated cluster detection** — If 10+ similar labels arrive within 5 minutes, all are quarantined as a potential coordinated attack.
3. **Quarantine protocol** — Low-trust and clustered labels are held for 24 hours. They do NOT influence ChromaDB patterns, decay weights, or thresholds until released.

### Constraint 6: Budget Reality (PKR 50,000/month)

**Problem:** Limited compute budget.

**Our solution — everything runs locally:**

| Component | Monthly Cost |
|-----------|-------------|
| Ollama LLM (gemma2:2b) | PKR 0 (local) |
| ChromaDB vector search | PKR 0 (local) |
| SQLite database | PKR 0 (local) |
| Electricity (~8hrs/day) | ~PKR 3,000-5,000 |
| **Total** | **~PKR 5,000** |

**PKR 45,000 budget remaining.** Per-request cost is effectively PKR 0.

**Degradation mode:** If Ollama fails 3 times consecutively, Stage 3 disables automatically. System continues on Stage 1+2 only — reduced accuracy but zero downtime.

---

## FramingAnalyzer (Our Unique Addition)

Standard keyword detection catches **what** scammers say. The FramingAnalyzer catches **how** they manipulate:

| Framing Type | What It Detects | Example |
|---|---|---|
| **Urgency Framing** | True fact + deadline pressure | "fees badh rahi hain, abhi karo" (fees ARE rising, but the deadline is fake) |
| **False Scarcity** | Real offer + artificial limit | "sirf 10 log ke liye offer" (the 10-person limit is invented) |
| **Authority Framing** | Real brand name + fake instruction | "SBP ne notice diya hai" (SBP is real, the notice is fake) |
| **Selective Omission** | Technically true but missing critical context | "guaranteed 10x return, no risk" (omits that you'll lose everything) |

The framing_score (0.0-1.0) is a separate signal with **0.20 weight** in the final confidence formula, tracked independently for decay.

---

## Confidence Calculation

```
final_confidence = Σ(stage_score × stage_weight × decay_factor) / Σ(stage_weight × decay_factor)
```

| Signal | Base Weight |
|--------|------------|
| Stage 1 (metadata) | 0.25 |
| Stage 2 (keywords + vectors) | 0.45 |
| Framing Analyzer | 0.20 |
| Stage 3 (LLM) | 0.30 |

Each weight is multiplied by its method's current decay factor before aggregation.

---

## 5-Action Decision Logic

| Action | When | What Happens |
|---|---|---|
| **AUTO_ALLOW** | confidence < 0.2 | Safe. No alert. Logged. |
| **AUTO_BLOCK** | confidence > 0.8 | Scam alert sent to user. Logged. |
| **ESCALATE** | confidence 0.4-0.8, budget available | Pushed to human review queue. |
| **DEFER** | confidence 0.2-0.4, or budget exhausted | Held for batch review. User gets "processing" status. |
| **REQUEST_MORE_DATA** | No message content available | API asks for the actual text or screenshot. |

---

## Three Failure Modes (Where We Break)

### 1. Novel Scam Type (No Pattern Match)
**What happens:** New scam language that doesn't match any keyword or ChromaDB pattern. Stage 1+2 miss it.
**Impact:** Scam passes through if Stage 3 LLM also fails.
**Containment:** Human reviewers catch it → pattern added to ChromaDB → future ones caught. System learns from every confirmed scam.

### 2. Coordinated Feedback Poisoning
**What happens:** Multiple adversaries submit "LEGITIMATE" verdicts for known scams to weaken the system.
**Impact:** If it bypasses the sanitizer, system starts auto-allowing real scams.
**Containment:** Cluster detection quarantines bulk labels. Trust scores decay for suspicious reviewers. If auto_allow rate spikes >20% in 1 hour, freeze all feedback processing and revert to last known-good weights.

### 3. All Detection Methods Decayed Simultaneously
**What happens:** Heavy traffic causes all 5 methods to trigger frequently, all decay weights drop near floor (0.3).
**Impact:** All confidence scores compressed → more DEFER/ESCALATE → queue overflows.
**Containment:** Decay resets every hour window. Floor of 0.3 ensures methods never go to zero. Dynamic threshold shifting handles queue pressure.

---

## Tech Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| Backend | FastAPI (Python) | Fast, async, auto-docs at /docs |
| Database | SQLite via SQLAlchemy | Zero setup, file-based, sufficient for demo |
| Vector DB | ChromaDB (local, persistent) | Cosine similarity on scam patterns, uses all-MiniLM-L6-v2 embeddings |
| LLM | Ollama with gemma2:2b (local) | Zero API cost, cultural context via prompt engineering |
| OCR | Pytesseract + Pillow | Screenshot text extraction for mobile money screenshots |
| Frontend | Single HTML file (vanilla JS/CSS) | No build step, serves from FastAPI directly |

---

## API Endpoints

Base URL: `http://localhost:8000/api/v1`

| Method | Endpoint | Purpose |
|--------|---------|---------|
| POST | `/messages` | Submit text message for analysis |
| POST | `/messages/screenshot` | Submit screenshot for OCR + analysis |
| GET | `/reviews/next` | Get highest-priority pending review |
| POST | `/reviews/{id}/decide` | Submit human decision (SCAM/LEGITIMATE/AMBIGUOUS) |
| GET | `/status` | System status: budget, decay weights, queue depth |
| POST | `/patterns` | Add new scam pattern to ChromaDB |
| GET | `/patterns/search?q=...` | Search similar patterns |
| GET | `/analytics/decay` | Decay weight history (24h) |
| GET | `/analytics/feedback` | Feedback sanitizer stats |
| GET | `/health` | Health check |

Dashboard: `http://localhost:8000/dashboard`

---

## How to Run

```bash
cd daak
source ../venv/bin/activate
python main.py
```

Open browser: **http://localhost:8000/dashboard**

That's it. One command. Backend + frontend + database + vector DB all start together.

Prerequisites: Ollama running with `gemma2:2b` model (`ollama run gemma2:2b`).

---

## Project Structure

```
daak/
├── main.py                  # FastAPI app entry point
├── config.py                # Settings from .env
├── models/
│   ├── database.py          # SQLAlchemy engine + session
│   └── schemas.py           # Pydantic request/response models
├── db/
│   └── models.py            # 7 ORM models (messages, decisions, reviews, etc.)
├── services/
│   ├── stage1_metadata.py   # Metadata analysis (sender, links, frequency, time)
│   ├── stage2_content.py    # Keywords + ChromaDB vectors + FramingAnalyzer
│   ├── stage3_llm.py        # Ollama LLM with cultural context prompt
│   ├── decision_engine.py   # Confidence aggregation + 5-action decision
│   ├── decay_manager.py     # Signal collapse prevention (5 methods tracked)
│   ├── budget_manager.py    # 20/hour human review budget
│   ├── feedback_sanitizer.py # Label trust + coordinated attack detection
│   └── ocr_service.py       # Pytesseract screenshot extraction
├── routers/
│   ├── messages.py          # Message ingestion + full pipeline orchestration
│   ├── reviews.py           # Human review queue + feedback loop
│   ├── patterns.py          # ChromaDB pattern management
│   ├── status.py            # Real-time system monitoring
│   └── analytics.py         # Decay + feedback analytics
├── data/
│   ├── seed_patterns.json   # 30 Roman Urdu scam patterns (6 categories)
│   └── keyword_dict.json    # 29 weighted scam phrases
├── templates/
│   └── dashboard.html       # Single-page dashboard (5 tabs, Roman Urdu hints)
├── chroma_data/             # ChromaDB persistent storage
├── .env                     # Configuration
└── requirements.txt         # Python dependencies
```

---

*DAAK — Because the best fraud detection catches the lie, not the transaction.*
