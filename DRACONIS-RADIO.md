# Draconis AI Radio — design

**Status: SCOPED / PARKED (2026-09-11). Not built.** Next concrete step before building:
figure out how to get the music library to where the analyzer/broadcaster can read it
(see [§8 Open items](#8-open-items--next-step)).

A self-hosted, AI-programmed **multi-station radio network** built on this fork of
[AudioMuse-AI](https://github.com/NeptuneHub/AudioMuse-AI). AudioMuse is the **DJ brain**
(sonic analysis + playlist/cluster generation); a separate **broadcaster** turns its
playlists into 24/7 streams you tune into. The AI clusters *your own* library into
sonically-coherent stations — classical, country, chill, high-energy, "sonic journey",
etc. — with no hand-curation.

---

## 1. Goal / vision

- Point it at the existing (large) music library.
- A **weekly job** keeps analysis fresh: ingest/analyze newly-added tracks, re-cluster.
- Run **multiple genre/mood stations** simultaneously, each a 24/7 stream at its own
  `<name>.draconis.io`.
- Let AudioMuse's **clustering define the stations automatically** — it finds the natural
  sonic groupings in the library; each cluster becomes a station.

## 2. Architecture

```
kirk (music files)
      │
      ├──► Navidrome (library API)  ──────────────┐  (AudioMuse reads the library here)
      │                                           │
      │                              WEEKLY GPU CRUNCH (on a HAL RTX 3090)
      │                              AudioMuse worker: analyze new tracks +
      │                              re-cluster (cuML) ──► writes results ──► Postgres
      │                                                                         │
      └──► AzuraCast (broadcaster, always-on, CPU) ◄── station playlists ◄──────┘
                 │        (one playlist per cluster/mood/genre)
                 └──► Classical | Country | Chill | High-Energy | Sonic-Journey | …
                            each a 24/7 stream → <name>.draconis.io
```

**Why this shape:**
- AudioMuse's results (embeddings, features, clusters, disk-paged IVF similarity index)
  all persist in **PostgreSQL**. Workers are stateless RQ jobs. So "where the GPU runs"
  and "where serving runs" are decoupled — both just point at a shared Redis + Postgres.
  There is no data handoff/migration; it's all in the DB.
- Serving (the web UI + playlist/similarity/song-path queries) and broadcasting (audio
  encoding) are **CPU/IO-bound** — no GPU needed. Only the periodic analysis wants a GPU.

## 3. Components & placement

| Component | Role | Runs on | GPU? |
|---|---|---|---|
| Music files | The library | **kirk** (see §8 — currently scattered) | no |
| **Navidrome** | Library API AudioMuse reads (OpenSubsonic) | kirk (or any box over the files) | no |
| **Postgres + Redis** | Shared state (analysis results) + job queue | the serving box (or kirk) | no |
| **AudioMuse flask** | Web UI, playlists, similarity, song-paths | serving box | no |
| **AudioMuse worker (weekly)** | Analyze new tracks + re-cluster | **HAL, RTX 3090** | **yes** |
| **AzuraCast** | Broadcaster: auto-DJ, rotation, Icecast+Liquidsoap, listener stats | serving box | no |

The **serving box can be HAL itself** (streaming is light) or any quieter box/mini. The
Tesla **P4/P40/P100** are **optional** — nothing in steady state needs their GPU; the
weekly crunch uses a HAL 3090 (see §4).

## 4. Hardware decisions & rationale

- **GPU acceleration is CUDA-only.** AudioMuse uses ONNX-CUDA (models: musicnn, CLAP,
  Whisper-small) and RAPIDS **cuML** (clustering). No AMD/ROCm, no Apple Metal. So Macs
  and AMD eGPUs (e.g. drtheopolis's RX 6800s) can't accelerate it.
- **cuML needs compute capability ≥ 7.0 (Volta/Turing/Ampere+).** The available Tesla
  cards — **P4 (8GB), P40 (24GB), P100 (16GB)** — are all **Pascal (6.0–6.1)**: ONNX
  *analysis* accelerates on them, but **cuML GPU-clustering does not run** (auto-falls
  back to CPU; it's off by default anyway).
- **HAL's RTX 3090 is Ampere (8.6 ≥ 7.0)** → it runs **both** fast ONNX analysis **and**
  cuML GPU-clustering (10–30× vs CPU). So the crunch belongs on a **3090**, not a Tesla.
- **8GB VRAM** is AudioMuse's recommended minimum; models are small. Set
  `PER_SONG_MODEL_RELOAD=true` only when VRAM is tight (a 3090's 24GB doesn't need it →
  faster).
- The heavy work is a **bounded weekly window**; the 3090 is free the rest of the time.

## 5. The weekly crunch job

A scheduled worker that wakes weekly, points at the shared Redis/Postgres, analyzes any
new tracks and re-runs clustering with `USE_GPU_CLUSTERING=true`, then exits.

- If **HAL is in the k8s cluster** → a **k8s CronJob** pinned to the 3090 node.
- Else → a weekly **cron/launchd** on HAL that `docker run`s the worker.
- **Guardrails (learned the hard way):** pin a *free* 3090 and set CPU/mem/pids limits —
  an unbounded GPU container once pegged HAL's cores. (fleet hazard:
  gpu-container-no-cpu-limit.)

## 6. Multi-station model

- **AzuraCast is multi-station by design** — one install hosts many independent stations,
  each with its own mount/stream URL, playlist rotation, auto-DJ, and listener stats.
- **AudioMuse fills them.** Its clustering + mood/genre/text search produce one playlist
  per station. The slick version: let AudioMuse's **sonic clusters auto-define** the
  station lineup, refreshed on the weekly crunch as the library grows.

## 7. The one custom piece — the glue

Everything above is off-the-shelf containers **except** the bridge from AudioMuse's
playlist output into AzuraCast's rotation. AudioMuse exports playlists (to the music
server / as playlists); AzuraCast programs stations from playlists/media. Wiring
"AudioMuse cluster N → AzuraCast station N (auto-refreshed weekly)" is the only bespoke
integration to build.

## 8. Open items / next step

- **★ Get the music to where it can be read (the agreed next step).** The library is
  currently **scattered across kirk's volumes** (`/Volumes/Data0001…0008/…/Music`,
  `Elements`, `Dropbox/Music`, the iTunes/Music.app library, …). Navidrome (for analysis)
  and AzuraCast (for broadcast) both need a coherent view of it — a consolidated path or a
  union mount. Deciding how to present the library is the prerequisite to building.
- **No broadcaster forked yet.** AzuraCast (or Liquidsoap+Icecast) is a new add — not yet
  in the GitHub repos.
- **Exposure:** per the Draconis procedure — Switchboard cloudflared tunnel + per-hostname
  GitHub-gated Access + CNAMEs for each `<station>.draconis.io` (and the AudioMuse UI).
- **AI-DJ text/lyrics features** point at the fleet LLM proxy (already instrumented).

## 9. Build order (when greenlit)

1. Resolve library access (§8) → stand up **Navidrome** on kirk over it.
2. Deploy AudioMuse serving stack (flask + redis + postgres) on the serving box; connect
   Navidrome; run a **first full analysis** on a HAL 3090.
3. Stand up **AzuraCast** on the serving box.
4. Build the **glue** (§7): AudioMuse clusters → AzuraCast stations.
5. Schedule the **weekly crunch** (§5).
6. Expose the UI + each station via tunnel/Access/CNAME.

---

*Fork of NeptuneHub/AudioMuse-AI. This doc records the Draconis-specific radio design;
upstream docs live in `docs/`.*
