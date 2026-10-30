# Syncing Hevy workouts to Garmin Connect: a developer's guide

**A fully automated Hevy-to-Garmin sync is achievable today using open-source Python tools**, despite Garmin officially rejecting Hevy's API application. The most practical path combines Hevy's official API (requires Pro subscription), a FIT file encoder, and the reverse-engineered Garmin Connect upload endpoint — and someone already built this exact pipeline. The project `hevy2garmin`, published in late February 2026, handles the entire flow: fetching workouts, mapping exercises, generating FIT files, and uploading to Garmin Connect. For developers comfortable with Python, this is a weekend project at most — or a `git clone` away from working.

## Hevy's official API gives you everything you need

Hevy ships a **public REST API (v1)** with full Swagger documentation at `https://api.hevyapp.com/docs/`. Authentication uses a simple API key passed via an `api-key` header. The catch: **Hevy Pro is required** to generate a key, which you obtain at `https://hevy.com/settings?developer`.

The API exposes all the data needed for sync. `GET /v1/workouts` returns paginated workout history with full detail — exercise names, set types (normal, warmup, dropset, failure), weight in kg, reps, distance, duration, and RPE. `GET /v1/workouts/count` returns total workout count, and `GET /v1/workout_events?since=<date>` enables incremental sync by surfacing updated or deleted workouts since a given timestamp. Exercise templates, routines, and routine folders are also accessible. The API even supports **webhooks** for real-time workout completion notifications, which could trigger an immediate sync.

A Python client library exists (`pip install hevy-api` from `github.com/remuzel/hevy-api`), though raw `requests` calls work fine given the simple auth model. Hevy also supports native **CSV export** from the app (Settings → Export & Import Data), which includes columns for exercise title, set index, set type, weight, reps, distance, duration, and RPE — useful as a fallback if you want to avoid the Pro subscription cost.

## Garmin Connect accepts uploads, but only through the back door

Garmin's official Connect Developer Program (`developer.garmin.com/gc-developer-program/`) is **restricted to approved business partners** — no individual developer tier exists. The program's Activity API reads data *out* of Garmin, and the Training API pushes *planned* workouts to devices, but **no official API endpoint exists for uploading completed activity files**. Garmin explicitly rejected Hevy's application for API access, stating Hevy was "too small."

The community solved this years ago. The **`garth` library** (`pip install garth`, 733 GitHub stars) handles Garmin SSO OAuth authentication, including MFA support, with tokens that persist for roughly a year. Built on top of it, **`python-garminconnect`** (`pip install garminconnect`) wraps 105+ Garmin Connect endpoints including the critical `upload_activity()` method. This hits the same internal upload endpoint that Garmin's own web interface uses:

```
https://connect.garmin.com/modern/proxy/upload-service/upload/.fit
```

Garmin Connect accepts **FIT, TCX, and GPX** file uploads. For strength training, **FIT is the only viable format** — it's the only one that encodes exercise categories, sets, reps, and weight via dedicated `SetMesg` message types. TCX and GPX lack strength training semantics entirely. Garmin Connect will display the workout with a muscle group graphic, exercise breakdown, and per-set data when the FIT file is properly encoded with `Sport.TRAINING` and `SubSport.STRENGTH_TRAINING`.

## The exercise mapping problem is the hardest part

The core technical challenge isn't API access — it's **translating Hevy's free-text exercise names into Garmin's enumerated exercise catalog**. Garmin's FIT protocol requires exercises to be identified by predefined category and name enums (e.g., `ExerciseCategory.BENCH_PRESS` with `BenchPressExerciseName.BARBELL_BENCH_PRESS`). These enums are defined in the FIT SDK's `Profile.xlsx` file, downloadable free at `developer.garmin.com/fit/download/`.

Every existing project tackles this differently. `hevy2garmin` uses keyword heuristics — if the Hevy exercise name contains "bench press" and "barbell," it maps to the appropriate Garmin enum. Unmapped exercises get logged to `unmapped_exercises.json` for manual review. `HevyConnect` uses a static lookup table with fuzzy matching as a fallback. The `underthebar` project maintains a `garmin_translations.json` file. No approach is perfect, but keyword heuristics handle **80–90%** of common exercises automatically.

A second challenge: **creating valid FIT files**. The official Garmin Python FIT SDK (`pip install garmin-fit-sdk`) is **read-only** — Garmin has confirmed no plans to add an encoder. For writing FIT files, use the community `fit-tool` library (`pip install fit-tool`) or write a custom binary encoder as `hevy2garmin` does. The FIT format uses a compact binary structure with file headers, definition messages, data messages, and CRC checksums — nontrivial but well-documented in Garmin's FIT protocol specification.

## Four open-source projects already exist on GitHub

The community has built several tools spanning the automation spectrum:

- **`amasolov/hevy2garmin`** — The most complete automated solution. A Python script that fetches workouts via Hevy's official API, converts to FIT using a custom binary encoder, and uploads to Garmin via `garth`. Supports scheduled runs (every 30 minutes), duplicate prevention via state files, multi-user configs, and Garmin MFA. Published February 28, 2026, designed for Home Assistant but runs standalone with cron. Dependencies are just `garth` and `requests`.

- **`lucasgirod/hevy_sync`** — A Python CLI tool (`pip install hevy-sync`) with Docker and Kubernetes support. Uses an older browser-based OAuth flow for Hevy authentication rather than the v1 API key. Supports output to FIT files, JSON, or direct Garmin upload. Recommended sync interval is every 3 hours due to Garmin rate limits.

- **`TonyTromp/HevyConnect`** — A Next.js web application for manual conversion. Users drag-and-drop a Hevy CSV export, the app converts it to a FIT file using `@garmin/fitsdk` (the JavaScript SDK does support encoding), and downloads the result for manual upload to Garmin Connect. No automation, but no Hevy Pro required since it uses CSV export.

- **`alwalms/hevy-to-garmin-converter`** — A lightweight static web tool deployed at `alwalms.github.io/hevy-to-garmin-converter/` for client-side Hevy CSV-to-Garmin conversion. Minimal but functional for quick manual transfers.

For reference architecture, **`philosowaffle/peloton-to-garmin`** is the gold standard "X-to-Garmin" sync project — a mature C# application handling Peloton workouts including strength training, with Docker, WebUI, and GitHub Actions deployment options.

## The recommended developer approach

**For fully automated sync**, clone `hevy2garmin` and set up a cron job on your Mac. The entire setup takes under 30 minutes:

```bash
git clone https://github.com/amasolov/hevy2garmin.git
cd hevy2garmin
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
# Configure users.json with Hevy API key + Garmin credentials
.venv/bin/python run_sync.py
# Then add to crontab: */30 * * * * cd /path/to/hevy2garmin && .venv/bin/python run_sync.py
```

**For a custom build**, the cleanest stack is a Python script using three libraries: `hevy-api` or raw `requests` for Hevy data, `fit-tool` for FIT file generation, and `garminconnect` (built on `garth`) for upload. The data pipeline is straightforward: pull workout JSON from Hevy → map exercise names to Garmin enums → encode FIT activity file with strength training messages → call `api.upload_activity("workout.fit")`. A webhook-triggered approach is also possible — Hevy's API supports webhook subscriptions for workout completion events, which could trigger a lightweight web server or serverless function to run the sync immediately.

**For manual export/import without Pro**, use `HevyConnect` or the static converter. Export CSV from Hevy (Settings → Export & Import Data), convert to FIT via one of these web tools, then upload the FIT file at `connect.garmin.com` via the Import button on the Activities page.

## No viable no-code bridge exists today

Neither **Zapier**, **Make (Integromat)**, nor **IFTTT** supports both Hevy and Garmin Connect natively. Zapier has no Garmin integration at all. **n8n** (self-hosted) is the closest option — it supports custom HTTP request nodes and inline Python/JavaScript — but at that point you're writing the same code you'd put in a standalone script with more overhead.

## Conclusion

The Hevy-to-Garmin sync problem is effectively solved. `hevy2garmin` delivers a working automated pipeline right now, requiring only a **Hevy Pro subscription** for the API key and a Garmin Connect account. The main limitation is exercise name mapping — expect to manually add a few custom exercises to the mapping file after the first sync. For developers who want more control, the building blocks are mature: Hevy's Swagger-documented API, the `garth`/`garminconnect` ecosystem for Garmin uploads, and the `fit-tool` library for FIT file creation.
