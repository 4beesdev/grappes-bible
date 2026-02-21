# Project Overview

> Last updated: 2026-02-21

## Vision

**grappes.ai** is a multi-agent AI roundtable platform. Users ask one question; multiple AI agents (Claude, GPT-4o, Gemini, Grok) hold a structured discussion — building on each other's ideas, challenging assumptions, and producing richer answers than any single model could.

## Core Concept

```
User asks a question
       ↓
  ┌─ Round 1 ──────────────────────────────┐
  │  Claude → GPT-4o → Gemini → Grok       │
  │  (each sees previous speakers' output)  │
  └─────────────────────────────────────────┘
       ↓
  ┌─ Round 2 ──────────────────────────────┐
  │  Claude → GPT-4o → Gemini → Grok       │
  │  (builds on Round 1)                    │
  └─────────────────────────────────────────┘
       ↓
  Final output: collaborative discussion
```

## Modes

| Mode | Purpose | Agent Behavior |
|------|---------|----------------|
| **Chat** | Casual conversation, ideas, recommendations | Open discussion, all perspectives welcome |
| **Analyse** | Data, strategy, comparisons | Critical thinking, evidence-based, challenge assumptions |
| **Build** | Code, websites, apps | Round 1: scope & plan. Round 2+: write code, fix each other's code |
| **Create** | Images & video | Creative generation with DALL-E 3, Imagen, Runway, Kling |

## Intensity Levels

| Level | Rounds | Token Budget | Description |
|-------|--------|-------------|-------------|
| **Light** | 1 | ~8,000 | Quick single-round answer |
| **Medium** | 3 | ~20,000 | Multi-round discussion |
| **Kraken** | Up to 100 | ~100,000 | Deep, unlimited exploration (dark UI, neon green) |

## Agent Personas

Each AI model has 4 specialized personas:

| Persona | Affinity | Role |
|---------|----------|------|
| **Analyst** | Analyse | Data, research, critical evaluation |
| **Coder** | Build | Software engineering, architecture, code review |
| **Writer** | Chat/Create | Communication, content, storytelling |
| **Strategist** | Build | Business strategy, planning, decisions |

**Autopilot mode:** system detects intent from the user's prompt and assigns the best persona automatically.

## Architecture

```
┌──────────────┐         ┌──────────────────┐
│  poly-web    │  SSE    │   poly-core      │
│  Next.js 14  │ ◄─────► │  Spring Boot 3   │
│  TypeScript  │  REST   │  Java 17         │
│  Tailwind    │         │  WebFlux (R2DBC)  │
│  Zustand     │         │                  │
└──────────────┘         └───────┬──────────┘
                                 │
                  ┌──────────────┼──────────────┐
                  │              │              │
             PostgreSQL       Redis        AI APIs
             (R2DBC)        (caching)     Claude, GPT-4o,
                                          Gemini, Grok,
                                          DALL-E 3, Imagen,
                                          Runway, Kling
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14 (App Router), TypeScript, Tailwind CSS, shadcn/ui, Zustand |
| Backend | Spring Boot 3, WebFlux, Java 17, Gradle |
| Database | PostgreSQL 16 (R2DBC), Flyway migrations |
| Cache | Redis 7 |
| Auth | JWT (access + refresh), Google OAuth, bcrypt |
| Payments | Paddle.net (webhooks) |
| AI Models | Claude (Anthropic), GPT-4o (OpenAI), Gemini 2.5 Flash (Google), Grok 3 (xAI) |
| Image Gen | DALL-E 3 (OpenAI), Imagen 3 (Google) |
| Video Gen | Runway Gen-4 Turbo, Kling AI Pro |
| Storage | S3-compatible (DigitalOcean Spaces) |
| Deployment | Docker, Docker Compose, nginx, Let's Encrypt |
| Server | DigitalOcean Droplet (AMS3) |

## Features

- **AI Roundtable** — sequential multi-agent discussion with shared context
- **Real-time Streaming** — SSE (Server-Sent Events), chunks arrive as they're generated
- **Agent Personas** — specialized roles per model (Analyst, Coder, Writer, Strategist)
- **Intent Detection** — automatic Chat / Analyse / Build / Create classification
- **Build Mode** — Round 1 scoping, Round 2+ coding with cross-agent code review
- **Kraken Mode** — dark UI, unlimited rounds, neon green theme
- **File Attachments** — PDF, Excel, CSV, TXT, ODF, images (vision-enabled)
- **Image Generation** — DALL-E 3 and Google Imagen side-by-side
- **Video Generation** — Runway and Kling with async polling
- **Projects** — group sessions with shared context and instructions
- **Markdown Rendering** — syntax-highlighted code, tables, math (KaTeX)
- **Subscriptions** — Paddle.net integration with webhooks and plan enforcement
- **Google OAuth** — one-click sign-in alongside email/password
- **Admin Dashboard** — platform stats, user management, usage charts
- **Data Retention** — S3 archiving, scheduled cleanup, context windowing
- **Rate Limiting** — per-user Redis-based rate and spending guards
- **Anomaly Detection** — automatic user blocking on suspicious usage patterns

## Repositories

| Repo | Description |
|------|-------------|
| [grappes-ai](https://github.com/4beesdev/grappes-ai) | Main codebase (poly-web + poly-core) |
| [grappes-bible](https://github.com/4beesdev/grappes-bible) | This repo — documentation, security, architecture |
