---
title: 'Building a SaaS with FastAPI: What I Learned Shipping Two Products'
description: 'Practical lessons from building and deploying two SaaS products with Python FastAPI - architecture decisions, deployment, and mistakes to avoid.'
pubDate: 'Apr 02 2026'
heroImage: ''
---

I've shipped two SaaS products using FastAPI as the backend: an AI-powered grant writing assistant and a freight offer analyzer. Both are in production, both serve real users, and both taught me things I wish I'd known earlier.

This isn't a FastAPI tutorial. It's the stuff between the tutorials — architecture decisions, deployment patterns, and mistakes that cost me hours.

## Why FastAPI (And When Not To)

**FastAPI wins when:**
- You need async I/O (calling external APIs, LLM providers, databases concurrently)
- Your team knows Python
- You're building API-first (separate frontend)
- You need to process documents, PDFs, or run ML models

**FastAPI loses when:**
- You need server-side rendering (use Next.js)
- Your product is mostly CRUD with a web UI (use Django or Rails)
- You want a batteries-included framework (FastAPI is minimal by design)

I chose it because both my products call external APIs heavily (Claude for AI analysis, Google Maps for routing) and need to process large documents. Python's ecosystem for document parsing is unmatched.

## Architecture That Worked

```
React Frontend (Lovable/Vercel)
    ↓ HTTPS
FastAPI Backend (Docker on VPS)
    ↓
PostgreSQL (Docker, same VPS)
    ↓
External APIs (Claude, Maps, etc.)
```

Simple. One VPS runs everything behind Traefik (reverse proxy + auto SSL). Total hosting cost: $8/month.

### Key decisions:

**1. Docker Compose for everything**

Backend, database, reverse proxy — all in Docker Compose. One `docker compose up -d` and you're live. No Kubernetes, no ECS, no complexity.

```yaml
services:
  api:
    build: ./api
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://...
    depends_on:
      - postgres
  
  postgres:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
```

**2. Traefik for SSL**

Don't manage SSL certificates manually. Traefik auto-provisions Let's Encrypt certificates and handles routing. Set it up once, forget it exists.

**3. asyncpg over SQLAlchemy**

For simple queries, raw asyncpg with a connection pool is faster and simpler than SQLAlchemy's ORM. I use connection pools initialized on startup:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.pool = await asyncpg.create_pool(DATABASE_URL)
    yield
    await app.state.pool.close()
```

**4. Keep the API surface small**

Both products have fewer than 10 endpoints. Resist the urge to build REST resources for everything. If your frontend only needs 5 endpoints, build 5 endpoints.

## Mistakes I Made

### 1. Not Handling Token Limits Early

When calling LLMs, your input size matters. I hit Anthropic's 200k token limit on the first real document because I was sending raw HTML. Fix: aggressively clean and truncate input before sending.

**Lesson:** Always calculate token usage before API calls. Set hard limits. Clean your input data ruthlessly.

### 2. Synchronous Mindset

My first version processed documents sequentially — fetch page, extract text, then call AI. Switching to concurrent fetching with `asyncio.gather()` cut response times by 60%.

```python
# Bad
for doc in documents:
    text = await fetch_document(doc.url)

# Good  
texts = await asyncio.gather(*[
    fetch_document(doc.url) for doc in documents
])
```

### 3. No Caching Strategy

Every identical request hit the AI API again. Adding a simple PostgreSQL cache (URL as key, response as JSONB) cut costs by ~40% and response times by 90% for repeated queries.

### 4. Environment Variables as Strings

Pydantic settings parse environment variables, but `list[str]` doesn't parse from a single env var string. Use `str` and split manually, or use JSON format. This one cost me an hour of debugging in production.

## Deployment Flow

My deploy process is 3 commands:

```bash
git push origin main
ssh server "cd /docker/myapp && git pull && docker compose up -d --build"
```

No CI/CD pipeline. No GitHub Actions. For a solo founder running 2 products, this is fine. Add automation when it hurts, not before.

### Monitoring

- **Health endpoint** (`/health`) — checks DB connection, returns uptime
- **Request logging** — FastAPI middleware that logs method, path, duration, status
- **Error alerts** — Sentry free tier catches unhandled exceptions

That's it. No Grafana dashboards, no Prometheus, no ELK stack. When something breaks, Sentry tells me. When I want to check status, I hit `/health`.

## Cost Breakdown

| Item | Monthly Cost |
|------|-------------|
| VPS (Hostinger) | ~$8 |
| Domain (x2) | ~$2 |
| Claude API | $5-30 (usage based) |
| Sentry | Free |
| Total | ~$15-40/month |

Both products run on the same VPS. The most expensive line item is AI API calls, which scale with usage (which means revenue).

## What I'd Do Differently

1. **Start with PostgreSQL from day one** — I initially used Supabase for convenience, then migrated. Should've gone direct from the start.
2. **Write database migrations** — I've been running raw SQL for schema changes. It works until it doesn't.
3. **Add rate limiting earlier** — bots found my endpoints within a week of launch.
4. **Structured logging** — print statements work in development. In production, you want JSON logs you can actually search.

## The Bottom Line

FastAPI + PostgreSQL + Docker + cheap VPS is an incredibly productive stack for solo founders. You can go from idea to production in a weekend, and the monthly cost is less than a Netflix subscription.

Don't over-engineer. Ship first, optimize when it hurts.

---

*I build SaaS products and websites. Currently working on [Cargoflow](https://cargoflow.lt) (AI freight analysis) and [Grantis](https://grantis.lt) (AI grant writing). Follow the journey on [LinkedIn](https://linkedin.com/in/aidasmarcink).*
