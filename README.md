# Personalised News Dashboard

A no-doom daily news digest — fetches 10+ RSS sources every 30 minutes, semantically deduplicates stories, ranks them by your personal topic preferences, and serves a clean React UI. Multi-user with JWT auth.

---

## What It Does

- Ingests RSS feeds from BBC, Reuters, NDTV, TechCrunch, NASA, WSJ and more every 30 minutes
- Deduplicates stories semantically using sentence-transformer embeddings + cosine similarity + union-find clustering (not just URL matching)
- Ranks articles per user based on recency + personal topic weights updated by feedback
- Rewrites clickbait headlines and generates plain-English summaries via LLM (optional — app works without it)
- Each user gets an independent personalised digest that shifts based on thumbs up/down

---

## Run Locally

### Prerequisites
- Docker Desktop installed and running

### Start everything
```bash
docker compose up --build
```

Then open:
- **App:** http://localhost:5174
- **Backend health:** http://localhost:8001/health

### First run note
The first digest may take a minute — it fetches feeds, downloads the sentence-transformer model, embeds articles, deduplicates, and ranks on first boot.

---

## Using the App

1. Open http://localhost:5174
2. Click **Sign up** — enter any email + password
3. You'll be redirected to your personal digest
4. Thumbs up/down articles to shift tomorrow's feed toward your interests
5. Each account has completely independent preferences

---

## Optional: AI Headline Rewriting

The app works without an API key — it uses original RSS headlines and summaries. To enable LLM rewriting:

```bash
# Add to your .env file or export before running
export ANTHROPIC_API_KEY=your_key_here
docker compose up --build
```

---

## Manual Pipeline Trigger

Force a feed refresh without waiting 30 minutes:

```bash
curl -X POST http://localhost:8001/api/pipeline/run
```

Celery Beat also runs this automatically every 30 minutes in the background.

---

## Auth API

```bash
# Sign up
curl -X POST http://localhost:8001/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"yourpassword"}'

# Login
curl -X POST http://localhost:8001/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"yourpassword"}'

# Get digest (use token from signup/login)
curl http://localhost:8001/api/digest/today \
  -H "Authorization: Bearer <your_token>"

# Submit feedback
curl -X POST http://localhost:8001/api/feedback \
  -H "Authorization: Bearer <your_token>" \
  -H "Content-Type: application/json" \
  -d '{"article_id":"<uuid>","feedback":"up"}'
```

---

## Stack

| Layer | Technology |
|---|---|
| Backend | FastAPI (Python) |
| Database | PostgreSQL |
| Job queue | Redis + Celery |
| Embeddings | sentence-transformers (all-MiniLM-L6-v2, runs locally) |
| LLM rewriting | Anthropic API (optional) |
| Frontend | React + Vite |
| Infrastructure | Docker Compose |

---

## Useful Debug Commands

```bash
# View live logs
docker compose logs -f backend celery_worker celery_beat

# Check article count in DB
docker compose exec postgres psql -U user -d newsdash -c "SELECT COUNT(*) FROM articles;"

# Check dedup clusters
docker compose exec postgres psql -U user -d newsdash -c \
  "SELECT cluster_id, COUNT(*) FROM articles GROUP BY cluster_id HAVING COUNT(*) > 1 LIMIT 5;"

# Check a user's topic weights
docker compose exec postgres psql -U user -d newsdash -c \
  "SELECT topic, weight FROM topic_weights WHERE user_id='<uuid>' ORDER BY weight DESC;"
```

---

## Environment Variables

Copy `.env.example` to `.env` and fill in:

```
DATABASE_URL=postgresql://user:password@postgres:5432/newsdash
REDIS_URL=redis://redis:6379/0
SECRET_KEY=your_random_32_char_string
ANTHROPIC_API_KEY=optional
```

Generate a SECRET_KEY:
```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```
