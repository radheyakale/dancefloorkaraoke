# Technical Highlights

Notes on the interesting problems behind DanceFloorKaraoke.

## 1. Aligning two dance videos that don't share a clock

**Problem.** Comparing a user's attempt to a reference clip frame-by-frame only works
if the two start at the same instant and run at the same tempo. In reality users start
late, pause, and drift.

**Approach.** Before touching pixels, analyze audio. Using `librosa`, the pipeline
detects onsets and estimates tempo for both tracks, then derives a temporal mapping
that warps the user's timeline onto the reference. Motion comparison happens on the
aligned frames.

**Why it matters.** Audio is a far more reliable sync signal than visual motion —
it's high-contrast, cheap, and robust to camera angle and body proportions.

## 2. Making pose comparison fair across bodies and cameras

Raw landmark coordinates are meaningless across different body sizes, distances, and
framings. The pipeline normalizes landmarks (translate to a body-relative origin,
scale by a stable body dimension) so that a tall user in a wide shot is scored on
*motion*, not on their build or how close they stood to the camera.

## 3. Keeping the API responsive with an async video pipeline

Video analysis takes seconds to minutes. Doing it inline would block request threads
and collapse under concurrency. Instead:

- The API does the cheap part (validate, store, enqueue) and returns immediately.
- **RabbitMQ** buffers work; **workers** consume and process.
- Workers are stateless and horizontally scalable; the queue provides natural
  backpressure and lets slow jobs drain without dropping users.

This is the difference between "the upload spins forever" and "upload returns, then
the result appears."

## 4. GPU acceleration with honest fallback

The heaviest audio math is FFT-heavy and benefits enormously from a GPU. The system:

- Detects GPU availability and framework support at startup (`gpu_config`).
- Auto-tunes **batch size** to available VRAM.
- Uses **CuPy** for GPU FFT operations.
- **Falls back to CPU cleanly** when no GPU is present — critical because the same
  codebase is deployed on both GPU servers and a Raspberry Pi.

Configuration is automatic, so the same image runs on a workstation and an edge device.

## 5. Sessions that survive restarts and scale

In-memory sessions break the moment you run more than one Gunicorn worker or restart
the process. Sessions are instead persisted in the database via a session store, so
auth state is consistent across workers and deploys.

## 6. Portability as a design constraint

The deployment target ranges from Docker Compose on a server to a Raspberry Pi.
That constraint shaped choices throughout: avoiding hard GPU dependencies, bounding
frame counts (`MAX_FRAMES`), streaming uploads in chunks to GridFS rather than
buffering in memory, and keeping the stack buildable on ARM.

**Measured on edge hardware.** A complete comparison — audio synchronization plus
MediaPipe pose estimation over **920 frames** — finishes in **~5 minutes on a
Raspberry Pi 5**, with no GPU and no cloud offload. That is the honest CPU-only
baseline: good enough for on-site demos and small deployments, while the
same codebase uses GPU acceleration on capable servers to cut that time down
substantially.
