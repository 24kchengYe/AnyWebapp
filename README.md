<div align="center">

# AnyWebapp

### Ship a full-stack web app from zero to production — with AI as your tech lead.

*An Agent Skill for Claude Code / Cursor / ChatGPT / Gemini CLI*

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Standard](https://img.shields.io/badge/Standard-Agent_Skills-blueviolet)](https://github.com/anthropics/skills)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Compatible-8A2BE2)](#installation)
[![Cursor](https://img.shields.io/badge/Cursor-Compatible-00A67E)](#installation)
[![ChatGPT](https://img.shields.io/badge/ChatGPT-Codex-74AA9C)](#installation)

</div>

---

## What is this?

AnyWebapp is an **AI Agent Skill** that turns your LLM into a pragmatic full-stack tech lead. Instead of generic coding advice, it guides you through a battle-tested 9-phase development process — from `npx create-next-app` to deployed, monetized, internationally accessible product.

**Built from real experience:** This skill was extracted from shipping [Human Skill Tree](https://github.com/24kchengYe/human-skill-tree) — a Next.js 16 AI SaaS that went from zero to production (with auth, payments, i18n, and China CDN) in 2 weeks by a solo developer.

## What it covers

| Phase | What you build |
|-------|---------------|
| 0. Init | Next.js 16 + TypeScript + Tailwind + first Vercel deploy |
| 1. MVP | Core feature with AI streaming (OpenRouter + Vercel AI SDK) |
| 2. UI/UX | Dark theme, glass-morphism, responsive, micro-interactions |
| 3. Data | localStorage → Supabase cloud sync migration |
| 4. Auth | Google + GitHub + Email via Supabase Auth |
| 5. Payment | LemonSqueezy (international) + 爱发电 (China) subscriptions |
| 6. i18n | next-intl multi-language (path-prefix routing) |
| 7. Deploy | Vercel CLI + custom domain |
| 8. China | Cloudflare CNAME proxy (free, no ICP filing) |
| 9. Launch | README, social media, Product Hunt |

## Key features

- **Phase-based**: Each phase produces a deployable product. Never skip ahead.
- **Production-tested code**: Every code snippet comes from a real shipped product, not a tutorial.
- **Pitfall warnings**: 10+ real bugs and their fixes (OpenRouter API mismatch, Cloudflare redirect loops, Supabase session loss, Turbopack Unicode parsing, etc.)
- **Decision rationale**: Not just "use Supabase" but WHY — tradeoffs documented for every choice.
- **Dual market**: Covers both international (Stripe/LemonSqueezy, English) and China (爱发电, WeChat Pay, Cloudflare CDN) deployment.

## Installation

### Claude Code

```bash
npx skills install 24kchengYe/AnyWebapp
```

### Cursor / Windsurf

Copy `SKILL.md` to your project's `.cursor/skills/` directory.

### ChatGPT / Gemini / DeepSeek

Open `SKILL.md`, copy the content, and paste it as Custom Instructions or System Prompt.

### Manual

```bash
git clone https://github.com/24kchengYe/AnyWebapp.git
# Use SKILL.md as your AI's system prompt
```

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | AI Agent Skill — give this to your LLM |
| `PLAYBOOK.md` | Human-readable reference manual — for you to read |

## Tech stack covered

```
Framework:    Next.js 16 (App Router) + TypeScript
Styling:      Tailwind CSS v4 + shadcn/ui
AI:           Vercel AI SDK v6 + OpenRouter (18+ models)
Auth:         Supabase Auth (Google, GitHub, Email)
Database:     Supabase PostgreSQL + RLS
Payment:      LemonSqueezy + 爱发电
i18n:         next-intl
Deployment:   Vercel (CLI) + Cloudflare CDN
```

## Usage example

After installing the skill, just tell your AI what you want to build:

> "I want to build an AI writing assistant as a SaaS product."

The AI will:
1. Break it into phases
2. Give you the exact commands to run
3. Provide production-ready code snippets
4. Warn you about pitfalls before you hit them
5. Guide you through deployment and payment setup

## Part of Human Skill Tree

This skill is part of the [Human Skill Tree](https://github.com/24kchengYe/human-skill-tree) project — 33+ AI Agent Skills covering K-12 education through career development and life skills.

## License

MIT — use it however you want.

---

![Visitors](https://visitor-badge.laobi.icu/badge?page_id=24kchengYe.AnyWebapp)

[![Star History Chart](https://api.star-history.com/svg?repos=24kchengYe/AnyWebapp&type=Date)](https://star-history.com/#24kchengYe/AnyWebapp&Date)
