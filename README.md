# WorldQuant OpenClaw Skill

Automated alpha search and simulation for WorldQuant Brain using OpenClaw + Local LLM.

## Quick Start

```bash
# Login to WorldQuant
login to worldquant

# Generate alphas
generate 10 alphas

# Simulate batch
simulate batch 1

# Check results
show top alphas
```

## What It Does

- 🧠 **Generate** alpha ideas using local LLM (Ollama)
- 🔍 **Validate** syntax against WorldQuant operators
- 📊 **Simulate** alphas on WorldQuant Brain API
- 📈 **Track** results and rankings
- 🎯 **Optimize** parameters toward constraints

## Requirements

- OpenClaw (installed)
- WorldQuant Brain account (Expert tier)
- Local Ollama with LLM model
- Python 3.8+

## Installation

1. OpenClaw skill auto-discovery will find this
2. Or manually place in `/app/skills/worldquant-alpha-search/`

## Development

See [DEVELOPMENT.md](DEVELOPMENT.md) for roadmap and milestones.

## Status

🔧 **Phase 1: Foundation** (in progress)

- [ ] Skill structure
- [ ] Alpha generator
- [ ] Integration tests

## Author

Nuttakit (@NuttakitDW)

## License

MIT
