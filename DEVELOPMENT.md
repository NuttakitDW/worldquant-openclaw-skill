# WorldQuant Automated Alpha Search - Development Plan

## Vision
Build an **OpenClaw Skill** that automates WorldQuant Brain alpha generation, simulation, and optimization using your existing `automated-alphas-search` repo + local LLM + OpenClaw orchestration.

---

## Phase 1: Foundation (Week 1-2)

### 1.1 Audit Current Repo
- [ ] Read `main.py` completely
- [ ] Understand database schema (`database/`)
- [ ] Review operator definitions (`operators_latest.json`)
- [ ] Test existing workflow: `make start`
- [ ] Document API endpoints in use

**Deliverable:** `AUDIT.md` - Current state analysis

### 1.2 Create Skill Structure
```
/app/skills/worldquant-alpha-search/
├── SKILL.md                          # Main skill doc
├── scripts/
│   ├── alpha_generator.py            # Local LLM alpha gen
│   ├── simulator.py                  # Submit to WQ Brain
│   ├── tracker.py                    # Results tracking
│   └── optimizer.py                  # Parameter optimization
├── references/
│   ├── operators.md                  # Operator guide
│   ├── validation_rules.md           # Alpha validation
│   └── workflow.md                   # Full workflow
└── assets/
    └── sample_alphas.json            # Examples
```

**Deliverable:** Skill skeleton with proper structure

---

## Phase 2: Core Skill Development (Week 2-3)

### 2.1 Alpha Generator Script
**Purpose:** Generate alpha ideas using local LLM (Ollama)

```python
# Pseudo-code flow
def generate_alpha():
    1. Load cached WorldQuant session
    2. Fetch available fields/operators
    3. Call Ollama to generate alpha code
    4. Validate syntax against operators_latest.json
    5. Return validated alpha expression
```

**Features:**
- Batch generation (e.g., 10 alphas per run)
- Constraint-based generation (no invalid operators)
- Caching of good alphas

**Deliverable:** `scripts/alpha_generator.py` (standalone, testable)

### 2.2 Simulator Script
**Purpose:** Submit alpha to WorldQuant Brain for simulation

```python
# Pseudo-code flow
def simulate_alpha(alpha_expr):
    1. Use cached session from login skill
    2. POST to /alpha/{alpha_id}/simulate
    3. Poll for results (sharpe, turnover, fitness)
    4. Save results to database
    5. Return metrics
```

**Features:**
- Async polling (don't block OpenClaw)
- Error handling (invalid alphas, rate limits)
- Result persistence

**Deliverable:** `scripts/simulator.py` (testable, stateless)

### 2.3 Tracker Script
**Purpose:** Persist and analyze alpha performance

```python
# Database schema (SQLite or JSON):
{
    "alpha_id": "unique_id",
    "expression": "alpha code",
    "status": "generated|simulating|completed|failed",
    "metrics": {
        "sharpe": 1.5,
        "turnover": 0.45,
        "fitness": 0.8
    },
    "created_at": timestamp,
    "simulated_at": timestamp
}
```

**Features:**
- Track generation → simulation → ranking
- Export best alphas
- Prevent duplicate submissions

**Deliverable:** `scripts/tracker.py` + `database.json`

---

## Phase 3: OpenClaw Integration (Week 3-4)

### 3.1 Main Skill Orchestrator
**SKILL.md actions:**
- `generate` - Create N alphas via LLM
- `simulate` - Submit alphas to WQ Brain
- `status` - Show simulation progress
- `rank` - Sort by sharpe/turnover/fitness
- `export` - Get top alphas

**Example triggers:**
```
"generate 10 alphas"
"simulate last batch"
"show top 5 alphas"
"export results to csv"
```

**Deliverable:** Core SKILL.md with full workflow

### 3.2 Session Caching Integration
**Reuse existing login skill:**
```python
# In alpha_generator.py
from utils.session import load_cached_session

session = load_cached_session("brain_session_cache.json")
# If expired: auto re-authenticate via worldquant_login.sh
```

**Deliverable:** Seamless session reuse

### 3.3 Async Job Tracking
**For long-running simulations:**
```
User: "generate 10 and simulate"
Bot: "Started batch job #123. Check back in 5 min"
[5 min later: auto update with results]
```

**Deliverable:** Background job system

---

## Phase 4: Optimization & Tuning (Week 4+)

### 4.1 Alpha Optimizer
**Auto-tune parameters:**
- Constraint: `sharpe > 1.25`
- Minimize: `turnover < 70`
- Maximize: `fitness >= 1`

**Deliverable:** `scripts/optimizer.py`

### 4.2 Feedback Loop
- Track which operator patterns work best
- Bias LLM toward successful patterns
- Learn from failed alphas

**Deliverable:** Learning system

### 4.3 Performance Dashboard
- Visualize sharpe/turnover over time
- Show progress toward constraints
- Export reports

**Deliverable:** Markdown/JSON reports

---

## GitHub Strategy

### Option A: Public Repo (Recommended for this)
```
https://github.com/NuttakitDW/openclaw-worldquant-skill
```

**Structure:**
```
openclaw-worldquant-skill/
├── SKILL.md
├── scripts/
├── references/
├── DEVELOPMENT.md         # This plan
├── PROGRESS.md           # Weekly updates
└── README.md             # Quick start
```

**Benefits:**
- Track progress via commits
- Easy rollback if needed
- Reusable for others

### Option B: Keep in Workspace Only
- Simpler for private work
- Still use git locally for history

### Recommendation
**Use GitHub with these practices:**
1. **Main branch** - Stable, tested code
2. **Dev branch** - Active development
3. **Tags** - v0.1 (foundation), v0.2 (simulation), v0.3 (optimization)
4. **Issues** - Track bugs, TODOs
5. **Releases** - Package stable versions as .skill files

---

## Weekly Milestones

### Week 1
- [ ] Skill skeleton created
- [ ] `alpha_generator.py` working
- [ ] GitHub repo initialized
- **Commit:** v0.1-alpha

### Week 2
- [ ] `simulator.py` working
- [ ] Database schema finalized
- [ ] Session caching integrated
- **Commit:** v0.2-simulator

### Week 3
- [ ] SKILL.md complete
- [ ] Full end-to-end workflow tested
- [ ] OpenClaw integration done
- **Commit:** v0.3-integrated

### Week 4+
- [ ] Optimization layer
- [ ] Performance dashboard
- [ ] Package as distributable .skill
- **Release:** v1.0

---

## Tools & Dependencies

### Already Have
✅ WorldQuant Brain API (authenticated)
✅ OpenClaw + Skills framework
✅ Your existing `automated-alphas-search` repo
✅ Local Ollama/LLM

### Need to Add
- [ ] SQLite or persistent JSON database
- [ ] Async job queue (optional: use OpenClaw's native system)
- [ ] Better logging/monitoring

---

## Quick Start (Next Steps)

**Do this NOW:**
1. Create GitHub repo (or skip if private)
2. Create `/app/skills/worldquant-alpha-search/SKILL.md` skeleton
3. Run `main.py` from your repo once to verify it works
4. Document what you see in `AUDIT.md`
5. Share audit findings here

**Then start Phase 1.2**

---

## Success Criteria

✅ **Automated alpha generation** - 10+ alphas per day
✅ **Automated simulation** - Auto-submit to WQ Brain
✅ **Tracking** - Database of all results with metrics
✅ **OpenClaw integration** - Chat commands to trigger jobs
✅ **Reproducibility** - Anyone can use the skill

---

## Questions for You

1. **GitHub?** Public (sharable) or private (just you)?
2. **Database?** SQLite (lighter) or PostgreSQL (more robust)?
3. **LLM model?** Which Ollama model are you running?
4. **Priority?** Speed of generation, or quality of alphas?
5. **Frequency?** Run continuously, or on-demand via OpenClaw commands?

Let me know and we'll kick off Phase 1! 🚀
