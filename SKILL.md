---
name: worldquant-alpha-search
description: Automated alpha generation, validation, and simulation for WorldQuant Brain. Use when you need to generate alpha ideas, simulate on WorldQuant Brain, track results, and optimize toward constraints (sharpe > 1.25, turnover < 70). Generates alphas using local LLM, validates against operator definitions, submits to API for backtesting, and ranks by performance metrics. Triggered by "generate alphas", "simulate batch", "show top alphas", "export results", or "optimize parameters".
---

# WorldQuant Alpha Search - OpenClaw Skill

Automated alpha generation and optimization for WorldQuant Brain API. Generate alpha ideas using local LLM, validate against real operators, simulate on WorldQuant Brain, and track performance metrics.

## Quick Start

### Prerequisites
- WorldQuant Brain account (Expert tier recommended)
- OpenClaw with this skill installed
- Local Ollama with LLM model running
- Valid cached WorldQuant session (from `worldquant-brain-login` skill)

### Basic Commands

#### Generate Alphas
```
generate 10 alphas
```
Creates 10 new alpha expressions using LLM, validates syntax, stores to database.

#### Simulate Batch
```
simulate batch 1
```
Submits alphas from batch 1 to WorldQuant Brain for backtesting.

#### View Results
```
show top alphas
show batch 1 results
```
Display ranked alphas by sharpe, turnover, fitness scores.

#### Export
```
export top 5
export batch 1 to csv
```
Export alphas and results for external analysis.

## How It Works

### Architecture

```
┌─────────────────────────────────────────────────────┐
│  OpenClaw Chat Interface                             │
│  (User commands: "generate", "simulate", "rank")    │
└──────────────────┬──────────────────────────────────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
    ┌───▼──┐  ┌───▼──┐  ┌───▼─────┐
    │ Gen  │  │ Sim  │  │ Tracker │
    │      │  │      │  │         │
    └──┬───┘  └──┬───┘  └────┬────┘
       │         │           │
       └─────────┼───────────┘
               │
        ┌──────▼────────┐
        │ Database      │
        │ (alphas,      │
        │  results)     │
        └───────────────┘
               │
        ┌──────▼────────────┐
        │ WorldQuant Brain  │
        │ API               │
        └───────────────────┘
```

### 1. Alpha Generation

**Input:** Constraints, field definitions, operator rules
**Process:**
1. Load available fields from WorldQuant API (or cache)
2. Call Ollama LLM with prompt: "Generate alpha using these fields and operators"
3. Parse response → alpha expression
4. Validate syntax against `operators_latest.json`
5. Store to database with status "generated"

**Output:** Validated alpha expression

### 2. Simulation

**Input:** Alpha expression
**Process:**
1. Retrieve cached WorldQuant session (from login skill)
2. POST to `/alpha` endpoint with expression
3. Poll `/alpha/{id}/simulate` until complete
4. Extract metrics: sharpe, turnover, fitness, error
5. Update database with results

**Output:** Performance metrics

### 3. Tracking & Ranking

**Input:** Simulation results
**Process:**
1. Store results in database with timestamp
2. Calculate percentile ranks vs all alphas
3. Filter by constraint satisfaction (sharpe > 1.25, turnover < 70)
4. Sort by fitness score

**Output:** Ranked alpha list

## Configuration

### Database
**Location:** `brain_session_cache.json` (same dir as script)

**Schema:**
```json
{
  "alphas": [
    {
      "id": "alpha_001",
      "expression": "rank(close) * volume",
      "status": "completed",
      "metrics": {
        "sharpe": 1.45,
        "turnover": 0.52,
        "fitness": 0.88,
        "max_drawdown": -0.15
      },
      "created_at": "2026-04-26T09:30:00Z",
      "simulated_at": "2026-04-26T09:35:00Z"
    }
  ]
}
```

### LLM Model
Set via environment or config:
```bash
export OLLAMA_MODEL="qwen2.5-coder:7b"
```

## API Reference

### Internal Functions

See `references/` for detailed documentation:
- `alpha_generator.py` - Generate alphas via LLM
- `simulator.py` - Submit to WorldQuant Brain
- `tracker.py` - Persist and analyze results
- `validator.py` - Check alpha syntax

### WorldQuant Brain Endpoints Used

- `GET /authentication` - Verify session
- `POST /alpha` - Create alpha
- `POST /alpha/{id}/simulate` - Run backtest
- `GET /alpha/{id}` - Fetch results

## Troubleshooting

### "Session invalid"
- Your WorldQuant session expired
- Run: `login to worldquant` to refresh

### "Operator not found"
- Alpha uses invalid operator
- Check: `operators_latest.json` for valid operators
- See: `references/operators.md` for syntax rules

### "Rate limited (429)"
- Too many submissions to WorldQuant
- Wait 60 seconds, then retry
- Reduce batch size

### "Ollama not responding"
- Local LLM server not running
- Start: `ollama serve`
- Check model: `ollama list`

## Development

See [DEVELOPMENT.md](DEVELOPMENT.md) for:
- 4-phase roadmap
- Weekly milestones
- Contributing guidelines
- Architecture decisions

Current phase: **Phase 1 - Foundation**

## Status

🔧 **Early Development**
- [x] Skill structure
- [ ] Alpha generator (in progress)
- [ ] WorldQuant simulator
- [ ] OpenClaw integration
- [ ] Optimization layer

## License

MIT
