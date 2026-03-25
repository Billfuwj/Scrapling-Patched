# Quick Start

## For AI Agents (OpenClaw, etc.)

Read **[AGENT_GUIDE.md](./AGENT_GUIDE.md)** — step-by-step commands with exact expected outputs.

## For Human Developers

Read **[TUTORIAL.md](./TUTORIAL.md)** — full tutorial in Chinese with explanations and architecture overview.

## What's Patched?

Read **[CHANGELOG-PATCH.md](./CHANGELOG-PATCH.md)** — P0 infinite-loop fixes applied to the original Scrapling.

## Install (TL;DR)

```bash
git clone https://github.com/Billfuwj/Scrapling-Patched.git
cd Scrapling-Patched
python3 -m venv venv && source venv/bin/activate
pip install -e ".[all]"
playwright install chromium
```

## Verify

```bash
python3 -c "from scrapling import Fetcher, StealthyFetcher, Selector; print('OK')"
```
