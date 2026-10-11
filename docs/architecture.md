# Architecture Deep-Dive

## System context

DanceFloorKaraoke is a monolith-backed, service-assisted platform. A single Flask
application serves the REST API and the built React SPA, while CPU/GPU-heavy video
processing is offloaded to asynchronous workers.

```
                    ┌─────────────────────────────────────────────┐
   Browser  ─────►  │ Nginx  (TLS termination, reverse proxy,      │
                    │         static SPA + media serving)          │
                    └───────────────┬─────────────────────────────┘
                                    │
                        ┌───────────▼───────────┐
                        │   Flask API (Gunicorn) │
                        │  auth · courses · social│
                        │  admin · payments       │
                        └───┬───────┬─────────┬───┘
                            │       │         │
             ┌──────────────────┐ ┌──────────┐ ┌────────────┐
             │ SQL (SQLAlchemy) │ │  Redis   │ │  RabbitMQ  │
             │ SQLite on Pi 5   │ │  cache   │ │ job queue  │
             │ MySQL at scale   │ │          │ │            │
             └──────────────────┘ └──────────┘ └─────┬──────┘
                                               │ consume
                                    ┌──────────▼──────────┐
                                    │  Video Worker       │
                                    │  OpenCV · MediaPipe │
                                    │  librosa · CuPy     │
                                    └──────────┬──────────┘
                                               │
                                    ┌──────────▼──────────┐
                                    │ MongoDB (GridFS)    │
                                    │ video / audio blobs │
                                    └─────────────────────┘
```

## Request lifecycle: a video comparison

1. **Upload** — the client posts the user video. The API streams it into MongoDB
   GridFS (chunked, so multi-hundred-MB clips never sit fully in memory).
2. **Enqueue** — a job descriptor (reference id, user video id, user id) is pushed
   onto the `video_comparison` RabbitMQ queue. The API returns a job reference
   immediately.
3. **Consume** — a worker process pulls the job. Workers are stateless and can be
   scaled horizontally; the queue provides backpressure.
4. **Align** — audio from both clips is analyzed with `librosa` (onset detection,
   tempo estimation). Frames are aligned on the audio timeline so a user who starts
   a beat late is not penalized for the offset.
5. **Landmark extraction** — `MediaPipe Pose` extracts 33 landmarks per frame for
   reference and user video. A `MAX_FRAMES` cap bounds worst-case work.
6. **Score** — landmarks are normalized (translation/scale invariant), compared, and
   aggregated into an overall score with per-region sub-scores.
7. **Persist & publish** — results and annotated media are written back to
   SQL/GridFS; the client polls or is notified and renders the synchronized
   side-by-side comparison.

## Data model (selected)

- **users** — credentials, roles (`user` / `admin`), OAuth identity, credit balance
- **reference_videos** — curated source choreography
- **video_comparisons** — scores, sub-scores, linked media ids
- **follows / likes / comments / notifications / achievements** — social graph
- **courses / lessons / enrollments / wishlists** — creator content + commerce
- **transactions / credit_bundles / purchases** — payments and credits
- **sessions / password_reset_tokens / email_subscriptions** — auth + lifecycle

Migrations are managed with **Alembic** and a consolidated `consolidated_schema.sql`.
The Raspberry Pi 5 deployment uses **SQLite**. A deployment at scale would use **MySQL**
with the same schema.

## Deployment topology

- **Local / demo:** `docker compose up` brings up Nginx, frontend build, backend,
  and Redis on a shared network.
- **Production:** Gunicorn behind Nginx with TLS; MongoDB and RabbitMQ as
  backing services.
- **Edge:** a trimmed build runs on a Raspberry Pi 5 for on-site demos, with
  **SQLite** as the database. The same build avoids hard GPU dependencies and
  degrades to CPU automatically. A full comparison (audio sync + MediaPipe over
  920 frames) runs in ~5 minutes on a Raspberry Pi 5.
- **At scale:** the same schema on **MySQL** instead of SQLite.

## Cross-cutting concerns

- **Auth:** JWT sessions in a database-backed session store (not in-memory), so
  sessions survive restarts and scale across workers. Google OAuth as an alternate
  identity provider.
- **Logging:** component-scoped loggers with rotating file handlers; the frontend
  mirrors logs to `localStorage` and ships them to the backend.
- **Config:** all secrets and endpoints come from environment variables; nothing
  credential-bearing is committed.
