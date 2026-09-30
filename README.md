<!--
Pre-publication note: This practitioner report is for the submission. Full detail about the funded project and authors' affiliation will be revealed upon paper acceptance.
-->

# Eye-Rec: webcam eye tracking and a self-hosted LLM for formative quizzes

This repository accompanies the LAK27 practitioner report *What Webcam Eye
Tracking Can and Cannot Tell a Teacher About Formative Quizzes* (anonymised for
review). It holds the complete Eye-Rec code, the deployment
set-up for one rented server, and the aggregate results of the first classroom
collection ("wave 1", 74 students, August 2026). This README is the reference
document the paper points to: it describes the whole deployment and reports the
data-collection results in more detail than four pages allow.


## Contents

1. [Overview and scope](#1-overview-and-scope)
2. [Repository map](#2-repository-map)
3. [Quick start on a laptop](#3-quick-start-on-a-laptop)
4. [Deployment on a rented server](#4-deployment-on-a-rented-server)
5. [Cost](#5-cost)
6. [Capacity and latency](#6-capacity-and-latency)
7. [Teacher workflow](#7-teacher-workflow)
8. [Student workflow](#8-student-workflow)
9. [Tracking and logging](#9-tracking-and-logging)
10. [Analytics and visualisation](#10-analytics-and-visualisation)
11. [LLM analytics](#11-llm-analytics)
12. [Languages](#12-languages)
13. [Ethics, data protection and the EU AI Act](#13-ethics-data-protection-and-the-eu-ai-act)
14. [Data-collection results (wave 1)](#14-data-collection-results-wave-1)
15. [Regenerating numbers and figures](#15-regenerating-numbers-and-figures)
16. [Known issues and limitations](#16-known-issues-and-limitations)
17. [Showcase video](#17-showcase-video)
18. [Licence and third-party notices](#18-licence-and-third-party-notices)
19. [How to cite](#19-how-to-cite)

---

## 1. Overview and scope

Eye-Rec runs short formative quizzes in the student's own browser and records
*how* each question was read, not only which answer was chosen.

- A teacher writes a quiz as a plain-text outline (`.txt` or `.pdf`),
  checks the parsed draft in a review editor and approves it. Students can join
  only after approval.
- Students join with a group code. WebGazer.js 3.5.3 estimates gaze from the
  laptop webcam **inside the browser**. The server receives no video or images,
  only gaze coordinates, the screen rectangles of the question and answer
  regions, answers with times, focus and copy events, and technical set-up
  values (for example camera name and browser user agent).
- The teacher dashboard shows item statistics, Bloom profiles, gaze replays and
  (since release 1.3.0) scanpaths, an AOI timeline and class dwell per region.
- On request, a self-hosted open-weight language model (EuroLLM-9B-Instruct via
  Ollama, on the same server) drafts a class report from a pseudonymous feature
  table. A flagged rule-based report replaces a failed or timed-out run.

```text
Student laptop (browser)
  webcam -> WebGazer.js 3.5.3 (face mesh, ridge regression)
  sends no video or images, only: gaze x,y (<= 30 Hz), AOI rectangles,
  answers, focus/copy events, set-up values
        |
        | HTTPS
        v
Rented server (Docker Compose)
  caddy    ports 80/443       Let's Encrypt certificate, reverse proxy
        |
        v
  app      https://62-238-19-178.sslip.io     FastAPI + uvicorn, SQLite file data/eyerec.db
        |
        v
  ollama   hosted       EuroLLM-9B-Instruct (Q4_K_M), reports on request

Teacher browser -> caddy -> app: /teacher  /review  /dashboard  /system
```

**Intended use.** Formative quizzes in ordinary teaching, analysed at class
level. In wave 1 the quiz did not count towards grades.

**What Eye-Rec does not do.**

- It does not record, store or upload video, face images, face landmarks or
  biometric templates.
- It does not grade with gaze. Scores come from the answer key only; gaze,
  events and set-up values never enter a score.
- It infers no emotions and performs no cheating detection. Focus and copy
  events are plain browser-event counts shown to the teacher.
- Students never see a score (`/api/finish` returns only `{ok: true}`).
- It makes no decisions. LLM reports are advisory and never shown to students.
- It has no live class roster, no LMS/LTI integration and no QR codes.
- The wave-1 data do not support identifying which answer option a student
  read; they support comparisons of question and answer regions at class level
  (section 14).

**Status.** Software release 1.3.0 (14 September 2026). Wave 1 was collected on
17-18 August 2026 with release 1.2 and the original measurement pipeline
(3 x 3 click calibration, 3-second centre validation). Features added later are
marked with their release throughout.

| Release | Date | Main additions |
|---|---|---|
| 1.0 | 2026-07-27 | Quiz, calibration, gaze logging, dashboard, replay, LLM report, 7 languages |
| 1.1 | 2026-08-12 | Four question types, draft-review-approve, PDF snapshots, open-answer grading, custom LLM question |
| 1.2 | 2026-08-13 (dated additions to 2026-08-20) | Survey, Bloom levels, item discrimination, KaTeX, `/system` page, collection fixes, A/B flag and teacher-event log (2026-08-20) |
| **wave 1** | 2026-08-17/18 | 74 students, one section |
| 1.3.0 | 2026-09-14 | Research options (5-point validation, metric set-up, calibration arms), scanpath view, AOI timeline, class dwell, gaze profiles |

---

## 2. Repository map

Status: **ready** = publish as is; **scrub** = remove identifying strings or
fix before publishing; **create** = does not exist yet; **exclude** = never
publish.

| Path | Contents | Status |
|---|---|---|
| `eyerec/main.py` | FastAPI app, 57 routes (pages, student API, teacher API, exports, admin) | ready (fix issues in section 16) |
| `eyerec/db.py` | SQLite schema, WAL settings, migrations, deletion routines | ready (fix section deletion, section 16) |
| `eyerec/analysis.py` | smoothing, AOI re-tagging, metrics, I-DT fixations, Bloom, item statistics, gaze profiles, LLM feature table | ready |
| `eyerec/llm.py` | Ollama client, prompts, JSON schemas, guard rails, fallback, open-answer grading, fine-tune export | scrub (prompt wording, stale docstring) |
| `eyerec/quiz_loader.py`, `eyerec/pdf_rich.py` | quiz parser; PDF snapshots of question bodies | ready |
| `eyerec/survey.py` | post-quiz survey instrument (v2, 16 items) | scrub (comment and one option name the host country) |
| `eyerec/auth.py`, `eyerec/sysstats.py` | teacher accounts (scrypt), `/system` statistics | ready |
| `eyerec/static/` | pages (`student`, `teacher`, `review`, `dashboard`, `system`, `index`), `gaze-provider.js`, `gaze-metrics.js`, `gaze-calibration.js`, `gaze-experiments.js`, five `i18n*.js` files | scrub (consent wording, section 16) |
| `eyerec/static/vendor/` | `webgazer.js` (3.5.3, patched), MediaPipe face mesh, TF.js models, Chart.js 4.4.7, KaTeX 0.16.28 (about 21 MB) | scrub (add licence files) |
| `tests/` | 11 pytest files (398 cases at 1.3.0) and `tests/js/` (41 node tests) | ready |
| `docs/ARCHITECTURE.md`, `API.md`, `CODE_MAP.md`, `DATABASE.md`, `EYE_TRACKING.md`, `PDF_FORMAT.md`, `LLM.md`, `DEPLOY_HETZNER.md`, `HOSTING.md`, `INDEX.md`, `RELEASE_NOTES_1.*.md` | reference documentation | scrub (stale values, section 16) |
| `docs/STATE_*.md`, `RESEARCH_GUIDE.md`, `PLAN_V1_1.md`, `V1_1_IMPLEMENTATION_CONTRACT.md`, `GAZE_ACCURACY_AND_TOOLING.md`, `ML_SCENARIOS.md` | internal planning notes | exclude or scrub |
| `docker-compose.yml`, `Dockerfile`, `Caddyfile`, `.env.example`, `.dockerignore` | container deployment | ready (pin versions) |
| `deploy.sh` | one-command update (rsync, rebuild, health check) | scrub (remove default server and domain) |
| `run.sh`, `run.bat` | laptop / LAN launcher (Linux, Windows) | ready |
| `requirements.txt` | Python dependencies | scrub (split runtime/test, pin) |
| `sample_quiz/` | `bloom_ensemble_quiz` (v2 format, 12 items), `ensemble_learning_quiz`, `random_forest_quiz` (v1), each `.md` + `.pdf` | ready |
| `scripts/simulate_class.py`, `scripts/make_sample_pdf.py` | synthetic class; sample PDF builder | ready |
| `paper/report/wave1_numbers.py`, `ml_feasibility.py`, `make_drift_explainer.py`, `make_report_figures.py` | wave-1 analysis scripts | scrub (section code, legacy threshold, `--db` argument) |
| `paper/report/wave1_numbers.json`, `ml_feasibility.json` | aggregate results | scrub (drop per-student array) |
| `paper/report/profiles.py`, `profiles.json` | script that regenerates the gaze profiles | **create** |
| `paper/analyze.py`, `paper/stats.json` | cohort, Bloom, survey and drift statistics | scrub (section code) |
| `paper/pymovements_bridge.py`, `PYMOVEMENTS.md`, `pymovements_compare.json` | independent fixation cross-check | scrub (per-student blocks, section code) |
| `paper/release/build_release.py` | de-identified data release builder (not yet run for deposit) | scrub (salt, audit scope) |
| `paper/video/` | showcase-video pipeline (`script.py`, `tts.py`, `record.py`, `assemble.py`, `overlay.js`, `slides/`, `prepare_db.py`, `demo_section.py`, `capture_*.py`) | scrub (`overlay.js` mask list, section code) |
| `paper/lak27/` | paper builder (`build_paper.py`, `paper_text.json`), `make_figures.py`, `make_input_figure.py`, `figs/`, PDF, `LAK27_Showcase_Video.mp4` | ready after check |
| `images/` | README figures, `make_readme_figures.py`, `readme_aggregates.json` | ready |
| `LICENSE`, `THIRD_PARTY_NOTICES.md`, `CITATION.cff` | licence and notices (section 18) | **create** |
| `data/`, `paper/data/`, `paper/release/out/`, `.env`, `.claude/`, `.venv/`, `tools/` | live database, snapshots, credentials, certificates, local binaries | **exclude** |
| `paper/tracker-lab/`, `paper/aied2027/`, `paper/report/*.docx/*.pdf`, `paper/report/build_*.js`, `paper/PLAN.md`, `paper/prereg-osf.md` | other projects and internal reports | **exclude** |

---

## 3. Quick start on a laptop

Requirements: Python 3 (the container image uses 3.12), a current Chromium- or
Firefox-based browser with a webcam, and optionally
[Ollama](https://ollama.com) for the LLM report.

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
EYEREC_ADMIN_PASSWORD='choose-a-password' ./run.sh
```

`run.sh` starts three things and stops all of them on Ctrl+C:

- Ollama, if one is already listening on port 11434 or a bundled binary exists
  in `tools/`; otherwise the AI tab uses the rule-based fallback.
- An HTTPS server on port 8443 with a self-signed certificate for the laptop's
  LAN address (regenerated when the address changes or the certificate expires),
  so other machines in the room get webcam access.
- A plain-HTTP server on port 8000 for the teacher working on the laptop.

Then:

1. Open `http://localhost:8000/teacher`, log in as `admin` (password from
   `EYEREC_ADMIN_PASSWORD`, read at the first start only; without it a random
   password is written to `data/admin_credentials.txt`), and upload `sample_quiz/bloom_ensemble_quiz.md`.
2. Review and approve the draft, note the group code (for example `EM-7Q2K`).
3. Open `http://localhost:8000/student?mock=1`, enter the code and take the quiz
   with the mouse as simulated gaze. Mock mode is accepted only from localhost.
4. Open `http://localhost:8000/dashboard`.

Simulate a class and pull the report model:

```bash
.venv/bin/python scripts/simulate_class.py --code EM-7Q2K --students 12
ollama pull hf.co/bartowski/EuroLLM-9B-Instruct-GGUF:Q4_K_M   # about 5.6 GB
```

On Windows, `run.bat` creates the virtual environment on first run. The
database is one portable file, `data/eyerec.db`.

The same container stack also runs locally without HTTPS (browsers treat
`localhost` as a secure context, other devices on the LAN do not):

```bash
mkdir -p data && sudo chown -R 1000:1000 data   # the app runs as UID 1000
docker compose up -d --build                    # no profile: Caddy is skipped
docker compose exec ollama ollama pull hf.co/bartowski/EuroLLM-9B-Instruct-GGUF:Q4_K_M
```

Tests (398 pytest cases and 41 node tests at release 1.3.0; the suite mocks the
LLM, so no model is needed):

```bash
.venv/bin/python -m pytest -q
node --test tests/js/
```

---

## 4. Deployment on a rented server

Wave 1 ran on one Hetzner Cloud CX43 in an EU location. The steps below
reproduce that set-up with placeholders: `eyerec.example.org` for the domain
and `203.0.113.10` for the server address.

### 4.1 Requirements

| Resource | Requirement | Why |
|---|---|---|
| RAM | 16 GB | EuroLLM-9B Q4_K_M holds about 6.1 GB resident, plus about 0.3 GB app and 1.5 GB Docker/OS. |
| CPU | 8 shared vCPUs used; no GPU | Only the LLM is CPU-heavy; gaze estimation runs in the students' browsers. |
| Disk | about 12 GB for the stack | plus about 50-52 bytes per gaze sample (section 6) |
| Network | public IPv4 and IPv6, ports 22/80/443 | many campus and home networks are IPv4-only |
| HTTPS | mandatory | browsers expose the webcam (`getUserMedia`) only on secure origins |

The Arm64 CAX31 (8 vCPU, 16 GB) is documented as equivalent where offered.
Shared web-hosting packages cannot run Eye-Rec (no Docker, no long-running
processes).

### 4.2 Steps

1. **Create the server** (Ubuntu, EU location, SSH key, public IPv4 + IPv6,
   optionally backups). Note its address.
2. **DNS.** Create an `A` record `eyerec.example.org -> 203.0.113.10` (ask your
   IT department for a subdomain). For short pilots a wildcard-DNS service that
   maps a hostname containing the IP to that IP also works (wave 1 used one);
   such hostnames reveal the server address and depend on a third party.
3. **Install Docker:**
   ```bash
   ssh root@203.0.113.10
   curl -fsSL https://get.docker.com | sh
   docker --version && docker compose version
   ```
4. **Copy the repository** to `/root/eyerec` (`git clone` or `rsync`, excluding
   `.venv`, `tools`, `data`, `.env`).
5. **Configure:**
   ```bash
   cd /root/eyerec
   cp .env.example .env
   # edit .env: DOMAIN=eyerec.example.org  EYEREC_ADMIN_PASSWORD=<strong password>
   mkdir -p data && chown -R 1000:1000 data
   ```
6. **Start with the `https` profile** (adds Caddy, which obtains and renews a
   Let's Encrypt certificate automatically):
   ```bash
   docker compose --profile https up -d --build
   docker compose ps     # app, ollama, caddy running
   ```
7. **Pull the model once** into the `ollama_models` volume:
   ```bash
   docker compose exec ollama ollama pull hf.co/bartowski/EuroLLM-9B-Instruct-GGUF:Q4_K_M
   ```
8. **Verify** from your own machine:
   ```bash
   curl -s -o /dev/null -w "%{http_code} tls=%{ssl_verify_result}\n" https://eyerec.example.org/health  # 200 tls=0
   curl -s https://eyerec.example.org/health        # {"ok":true,"llm":{...,"model_pulled":true}}
   curl -s -o /dev/null -w "%{http_code}\n" http://203.0.113.10:8000/   # 000: app port not public
   ```
9. **Log in** at `https://eyerec.example.org/teacher` as `admin` and create one
   account per teacher. Students use `https://eyerec.example.org/student`.
10. **Smoke test** with synthetic students: upload and approve a sample quiz,
    then run on the server
    ```bash
    docker compose exec -T app python scripts/simulate_class.py \
        --base-url http://localhost:8000 --code <TEST-CODE> --students 3
    ```
    and delete the test section before opening it in the dashboard (see the
    section-deletion issue in section 16).

The documentation estimates about 30 minutes for these steps plus the model
download; this was not timed.

### 4.3 Compose services and profiles

| Service | Image | Ports | Notes |
|---|---|---|---|
| `app` | built from `Dockerfile` (`python:3.12-slim`, non-root UID 1000, one uvicorn process) | `127.0.0.1:8000` only | `./data` bind-mounted to `/app/data`; vendored tracker assets are baked into the image, so no tracker file comes from a CDN |
| `ollama` | `ollama/ollama` (unpinned) | none | reachable only from `app` at `http://ollama:11434`; volume `ollama_models` |
| `caddy` | `caddy:2` | 80, 443 | only with `--profile https`; `Caddyfile` reverse-proxies `{$DOMAIN}` to `app:8000`; volumes `caddy_data`, `caddy_config` |

Without a profile, `docker compose up` starts `app` and `ollama` for local tests.

### 4.4 Environment variables

| Variable | Safe placeholder / default | Effect |
|---|---|---|
| `DOMAIN` | `eyerec.example.org` | Caddy hostname (https profile) |
| `EYEREC_ADMIN_PASSWORD` | `change-me-please` | password of the `admin` account created at first start; if empty, a random one is written to `data/admin_credentials.txt` (read it, then delete the file) |
| `OLLAMA_MODEL` | `hf.co/bartowski/EuroLLM-9B-Instruct-GGUF:Q4_K_M` | report model; `cas/eurollm-1.7b-instruct-q8` is the documented faster option |
| `OLLAMA_TIMEOUT` | `1200` in compose (code default `300`) | seconds before an analysis falls back to the rule-based report |
| `OLLAMA_KEEP_ALIVE` | `24h` (ollama service) | keeps the model resident; Ollama's default unloads it after 5 idle minutes |
| `OLLAMA_URL` | `http://ollama:11434` in compose | Ollama endpoint |
| `EYEREC_MAX_STUDENTS` | `200` | joins per section; further joins get HTTP 429 |
| `EYEREC_ALLOW_MOCK` | `0` | `1` allows `?mock=1` from non-local clients (demos only) |
| `EYEREC_HTTPS_PORT` | `443` in compose (code default `8443`) | target of the http-to-https redirect |
| `EYEREC_HTTPS_REDIRECT` | `1` | `0` disables the redirect in LAN mode |
| `EYEREC_DB` | `data/eyerec.db` | database path (analysis scripts point it at a copy) |

Keep `.env` only on the server. It is excluded from `rsync`, the Docker image
and the repository.

### 4.5 Firewall

Docker publishes container ports through its own iptables rules, which `ufw`
does not filter. The protection that matters is the `127.0.0.1:8000` binding in
`docker-compose.yml`. Enable `ufw` for the host (`ufw allow 22,80,443/tcp`
before `ufw enable`) and, for container ports, a provider-side cloud firewall
allowing 22, 80 and 443.

### 4.6 Backups

The whole state is the `data/` folder (one SQLite file plus WAL side files).

```bash
# online snapshot while the server runs
docker compose exec app python -c "import sqlite3; s=sqlite3.connect('data/eyerec.db'); d=sqlite3.connect('data/backup.db'); s.backup(d)"
# copy home after each class
scp -r root@203.0.113.10:/root/eyerec/data ./eyerec-backup-$(date +%F)
```

Restoring means putting the folder back and running `docker compose restart app`.
Provider backups (daily, seven kept) cost 20% of the server price. Every copy
holds personal data: a student's deletion request has to reach backups too.

### 4.7 Updates

```bash
EYEREC_SERVER=root@203.0.113.10 EYEREC_DOMAIN=eyerec.example.org ./deploy.sh
```

`deploy.sh` copies the code with `rsync --delete` (excluding `.venv`, `tools`,
`data`, `.git`, `.env`, caches and `paper`), refuses to restart anything if the
server's `.env` is missing, runs `docker compose --profile https up -d --build`
(only changed services restart) and polls `https://DOMAIN/health` up to 40
times, 3 s apart, until it returns HTTP 200, then prints the TLS result and
whether the model is pulled. **Never deploy during a quiz:** the app restarts
for about 30 s; answers submitted in that window are lost (the client does not
retry `/api/answer`), whereas gaze batches are kept in the browser and resent.

### 4.8 Monitoring page

`https://eyerec.example.org/system` (admin only) refreshes every 10 s and shows
host load, RAM, disk, app memory and uptime, database and WAL size with row
counts, LLM state (warm or cold via Ollama `/api/ps`), timeout and recent
analysis durations, the student cap, a capacity outlook, an all-sections gaze
summary and in-process counters (requests, 5xx errors, gaze batches and rows,
answers, students active in the last 15 minutes). Four chips summarise it:

| Chip | Warning | Bad |
|---|---|---|
| Load | 1-minute load above CPU count | |
| RAM | below 4 GB available | below 2 GB available |
| Disk | below 10 GB free | |
| LLM | model cold | Ollama unreachable |

The counters live in process memory and reset on restart; the page is live
monitoring, not a history.

---

## 5. Cost

| Item | Amount | Notes |
|---|---|---|
| Hetzner Cloud CX43 (8 shared vCPU, 16 GB RAM, 160 GB disk) | **EUR 15.99 a month** | net list price for new orders from 15 June 2026 at the provider's EU locations |
| VAT | not included | depends on the customer's country |
| Primary IPv4 | not included | billed separately (EUR 0.50 a month net on the provider's IPv4 price page; the EUR 0.70 in `DEPLOY_HETZNER.md` is out of date) |
| Backups | not included | optional, 20% of the server price |
| Domain | none in wave 1 | wildcard DNS; an institutional subdomain is recommended |
| LLM | no per-call cost | runs on the same server; no external AI service |
| Staff time | not costed | set-up, updates, backups |

With backups and one IPv4 address, the list-price arithmetic gives about
EUR 19.69 a month before VAT; this is not an invoice figure. Prices depend on
location and order date, and the CX43 appeared as not currently available on
the provider's product page when checked on 29 September 2026 (possibly a
display issue; the CAX31 is the documented alternative). The older
"EUR 8-15 a month" in `docs/HOSTING.md` predates the model choice and the price
change.

---

## 6. Capacity and latency

Only LLM report durations were measured. No load test was run, and no
end-to-end latency was recorded for gaze upload, answer submission or dashboard
loading (Caddy has no access log, and `/system` keeps counters, not timings).

### 6.1 Capacity

| Quantity | Value | Kind | Source |
|---|---|---|---|
| Gaze rate per student | at most 30 samples/s | client constant (upper bound) | `gaze-provider.js` |
| Requests per active student | about 0.6/s | derived from 30 Hz and a 2 s flush | `docs/DEPLOY_HETZNER.md` |
| Realised tracking rate, wave 1 | median 22 Hz (n = 67) | measured in the start validation | `wave1_numbers.json` |
| Realised ingest, wave 1 | about 18 rows/s per student | estimate: 1,058,222 samples / (71 finishers x 13.8 median minutes on the questions) | derived |
| Storage per gaze sample | about 50-52 bytes | synthetic benchmark (360,000 rows) | `docs/DATABASE.md` |
| Wave-1 database | about 55.5 MB for 1,058,222 samples (about 52.5 bytes/sample including all tables) | file size of the frozen snapshot | snapshot metadata |
| Joins per section | 200 | configuration default, not a tested concurrency limit | `docker-compose.yml` |
| Write model | one uvicorn worker, SQLite WAL, `synchronous=NORMAL`, one writer at a time | design | `eyerec/db.py` |
| Served so far | one section, 74 joins over 17-18 August 2026 | observation | wave 1 |

An analysis holds one worker thread for the whole LLM call (up to
`OLLAMA_TIMEOUT`); concurrent analyses queue inside Ollama, one at a time.
The quiz never waits for the LLM.

### 6.2 LLM latency history (CX43, CPU only)

| Date | Input | Duration | Outcome | Source |
|---|---|---|---|---|
| 2026-08-14 | reference run, class size not documented | 348 s cold, 118 s warm | LLM report | `RELEASE_NOTES_1.2.md` |
| 2026-08-17 | first 57 wave-1 students, all per-student features (about 4,800 prompt tokens) | 1,243 s | exceeded the 1,200 s timeout; rule-based report shown | `RELEASE_NOTES_1.2.md` |
| 2026-08-17 | same class, compact features (above 16 students), identical input | 200 s, 373 s, 1,226 s | high variance in output length and time | `RELEASE_NOTES_1.2.md` |
| 2026-08-17 | same class, after a brevity instruction ("BE CONCISE") | 393 s (one warm run) | LLM report | `RELEASE_NOTES_1.2.md` |

Notes:

- Generation speed is reported as about 3.7 tokens/s on the shared vCPUs (the
  compose comment says about 4); the measurement method is not documented.
- The tuning took five runs on the live class. A `num_predict` of 1,200 cut
  reports off mid-JSON; 2,000 is now used as a guard, not a length target.
- The stored analyses in the wave-1 snapshot from 17 August are three EuroLLM
  reports and three rule-based fallbacks.
- 393 s is a single run; no cold run for 57 or more students was measured.
- Before 14 August the 300 s code default caused "Read timed out" fallbacks in
  production, which led to `OLLAMA_TIMEOUT=1200` and `OLLAMA_KEEP_ALIVE=24h`.
- The Ollama version was not recorded (the image is unpinned); record it with
  any new latency figure.
- Documented but unmeasured options: a dedicated-CPU plan (estimated about 2x
  faster) or EuroLLM-1.7B (estimated about 5x faster, lower quality).

---

## 7. Teacher workflow

### 7.1 Accounts

Two roles: `admin` (created at first start; sees all sections; adds, resets and
deactivates teacher accounts on `/teacher`) and `teacher` (sees only own
sections). Passwords are hashed with scrypt (n = 2^14, r = 8, p = 1); sessions
last 30 days. Students never log in.

### 7.2 Writing a quiz

One file per quiz: `.pdf` or `.txt` (at most 25 MB, a PDF at most 60
pages), all in the same plain-text outline (format v2, `docs/PDF_FORMAT.md`):

```text
QUIZ: Random Forests

Q1. [MC] What does the "random" in random forests refer to?
A) Random deletion of training rows
B) Random bootstrap samples and random feature subsets
C) Random choice of the loss function
D) Random shuffling of the class labels
ANSWER: B
BLOOM_LEVEL: remember

Q2. [YN] The out-of-bag error requires a separate validation set.
ANSWER: NO
BLOOM_LEVEL: understand

Q3. [FILL] Sampling with replacement is called ____.
ANSWER: bootstrap | bootstrapping | bootstrap sampling
BLOOM_LEVEL: apply

Q4. [OPEN] Explain in 2–3 sentences why averaging many trees
reduces variance.
MODEL: Trees are high-variance estimators; averaging many
decorrelated trees cancels their individual errors.
BLOOM_LEVEL: evaluate
```

| Type | Rules | Grading |
|---|---|---|
| `[MC]` | exactly four options `A)`-`D)`, `ANSWER: <letter>`; an untagged question with A-D options counts as `[MC]` (v1 files still parse) | automatic |
| `[YN]` | no options, `ANSWER: YES` or `ANSWER: NO` | automatic |
| `[FILL]` | no options, `ANSWER: v1 \| v2 \| ...` (each at most 120 characters); a missing `____` gives a warning | automatic; case-, accent-, space- and trailing-punctuation-insensitive |
| `[OPEN]` | no `ANSWER` line, optional `MODEL:` reference answer | pending until the LLM proposes or the teacher decides (section 11.7) |

`BLOOM_LEVEL:` (alias `BLOOM:`) is optional: `remember`, `understand`, `apply`,
`analyze`, `evaluate` or `create`. TeX math (`$...$`, `$$...$$`) renders through
a self-hosted KaTeX in the quiz, the review editor and the dashboard; malformed
math is shown as typed. In PDF uploads, question bodies with images, drawings,
fraction bars or non-ASCII math symbols are cropped from the page as PNG
snapshots (pdfplumber for positions, pypdfium2 at 144 dpi); answer options are
always re-rendered as buttons from the extracted text, so formulas belong in the
question body. `.md`/`.txt` files never get snapshots. Keywords stay in English
in every interface language.

### 7.3 Upload and parser errors

The parser is strict and rejects a file at its **first** violation with HTTP
422. The message names the line (unknown tag, duplicate `ANSWER`, bad Bloom
level, option order) or the question (wrong option count, missing answer, empty
text), and the teacher page shows it word for word, for example:

```text
Line 14: unknown question type [TF] - use [MC], [YN], [FILL] or [OPEN].
Question 3 ([MC]) has 3 options; exactly 4 (A-D) are required.
```

Server messages are English-only. Encoding: `.md`/`.txt` are read as UTF-8;
a Latin-1 file silently turns accented letters into replacement characters, and
a UTF-8 byte-order mark hides the `QUIZ:` title line.

### 7.4 Review and approval

Every upload creates a **draft** section. The teacher page shows the group code
with the notice that students can join only after review and approval, any
parse warnings and a preview. A student who tries to join a draft gets HTTP 403.
The review editor (`/review?code=...`) lets the teacher edit title, text, type,
options, correct answer, fill-in variants, the open reference answer and the
Bloom level, and add or delete questions; PDF snapshots appear "as students will
see it". *Save draft* re-validates with the parser's rules; *Approve & publish*
asks for confirmation. Published questions cannot be edited or unpublished
(grades would become ambiguous); research options and notes can still change.

![Teacher input: plain-text quiz file and the review editor](images/fig_input.png)

*Teacher input. (a) First question of `sample_quiz/bloom_ensemble_quiz.md`;
(b) the review editor after upload (sample quiz on a throw-away test server).*

### 7.5 Section options

All off by default:

| Option | Effect | Release |
|---|---|---|
| Require a personal code | students must enter a code (letters, digits, `-`, `_`, at most 24 characters) that links them across quizzes; a reused code in the same section is rejected with 409 | 1.2 (2026-08-17) |
| Research A/B | balanced server-side assignment of the adaptive or frozen calibration policy | 1.2 (2026-08-20) |
| 5-point validation | accuracy and precision at start and end (about +8 s per student) | 1.3.0 |
| Metric set-up | bank-card scaling and blind-spot distance test for measured degrees (about +45 s, skippable) | 1.3.0 |
| Calibration experiment | balanced assignment of `std`, `dense16`, `pursuit` or `reanchor` calibration; not yet run with real faces | 1.3.0 |

None of the research options was used in wave 1. Arm assignment gives each new
join the currently smallest arm (ties broken with `secrets.choice`), so arm
sizes never differ by more than one; mock joins are never assigned. Flags can be
changed per section later and affect only future joins.

### 7.6 Sharing the code

The group code is the initials of the first two title words, a hyphen and four
random characters without look-alikes (no I, L, O, 0, 1), for example `RF-ZCJH`.
The teacher page shows it large under "Project this code for your class",
together with the student URL. Students type it on `/student`; there is no QR
code or LMS integration.

### 7.7 During the class

There is **no live roster**. The dashboard updates on reload: the Students tab
shows *Finished* or, for anyone unfinished, a red answered/total fraction
(for example `11/25`). The only self-refreshing screen is the admin page
`/system` (every 10 s, "active students" across all sections). The teacher can
delete a single student row (also deletes the survey) and "Clean up empty rows"
(no answers, gaze or survey, joined more than 2 hours ago, so students still
calibrating are safe). Deleted anonymous ids are retired and never reissued.

### 7.8 Dashboard

| Tab | Contents |
|---|---|
| Overview | tiles (students, finished, mean and median score, mean total time), % correct and mean time per question, Bloom profile, per-question table (type, Bloom level, discrimination, answer distribution), "Gaze measurement" card (1.3.0) |
| Students | score distribution, time vs score, off-layout gaze vs score, re-reads per student, accuracy by Bloom level, question map (difficulty x discrimination, at least 6 students), gaze profiles (1.3.0, exploratory), sortable roster, student detail with set-up line |
| Gaze replay | Replay and Scanpath views, AOI timeline, reading metrics, class dwell per region (section 10) |
| Open answers | only when the quiz has `[OPEN]` items: LLM proposals, reasons, source, teacher verdicts |
| Survey | response rate, Likert distributions with means, select breakdowns, free-text answers, individual responses with delete |
| AI analysis | model status, optional teacher question, *Analyze now*, stored reports with delete (section 11) |
| Export | downloads (section 7.9) |

![Dashboard overview with wave-1 data](images/fig_overview.png)

*Overview tab with the wave-1 class (74 joined, 71 finished, mean score 65%).*

![Students tab group charts with wave-1 data](images/fig_students_masked.png)

*Students tab group charts with wave-1 data (anonymous ids on the re-read chart
masked). The "off-screen" label in this chart means gaze outside all question
and answer regions.*

### 7.9 Exports

| Export | Shape | Notes |
|---|---|---|
| Section JSON | full dump | quiz, students, answers, survey, events, raw gaze samples and layout rectangles per question |
| Research CSV | one row per student x question, **89 columns** at 1.3.0 | data dictionary in section 9.6; earlier exports have fewer columns (65 in the export made during wave 1) |
| Survey CSV | one row per respondent, 24 columns | 6 metadata + 18 item columns (16 current, 2 retired) |
| Longitudinal CSV | one row per student x section across the teacher's sections, 49 columns | linked by the optional personal code |
| Grading fine-tune JSONL | teacher-graded open answers in the runtime grading format | 404 if none |
| Analysis fine-tune JSONL | (features -> report) chat pairs | 404 if no report |

---

## 8. Student workflow

1. **Join** at `/student`: group code, optional (or required) personal code.
   The server creates the student row with a per-section id (`S01`, `S02`, ...)
   and stores the window size. The student sees their own id.
2. **Privacy notice.** A mandatory screen in the student's language (selector
   with globe icon at the top right on join, notice and instruction screens,
   hidden from calibration on). The camera starts only when the button is
   pressed. Two conditional bullets appear only when they apply (head-pose
   experiment; research measurements, added in 1.3.0).
3. **Calibration.** Nine dots on a 3 x 3 grid at 10/50/90% of the window,
   three clicks each (27 training samples). The camera preview is visible only
   here. A lighting hint appears below brightness 45/100 and a stronger
   covered-camera hint below 12; neither blocks.
4. **Validation.** The student looks at a centre dot for 3 s. The quiz always
   continues; students are not asked to recalibrate.
5. **Quiz** in fullscreen, one question at a time, forward only. Leaving
   fullscreen shows an overlay, pauses gaze sampling and the answer timer and is
   logged. Copy and right-click are blocked and logged. The first answer counts.
6. **End validation** (the same 3-s centre dot).
7. **Survey** (voluntary, skippable, every item optional; section 14.14).
8. **Done.** The camera tracks are stopped and WebGazer's local storage and
   IndexedDB data are deleted. No score is shown.

![Privacy notice in English (mock mode)](images/fig_consent.png)

*The privacy notice in English, rendered in mock mode on a test server. Two
phrases are known gaps: "no personal data" and "anonymous" describe
pseudonymous personal data (section 16).*

**Mock mode** (`?mock=1`) uses the mouse as simulated gaze and never starts the
camera. It is accepted only from localhost unless `EYEREC_ALLOW_MOCK=1`; mock
students are flagged in exports and never assigned to research arms.
`scripts/simulate_class.py` drives the real API with synthetic students from
four behaviour archetypes.

---

## 9. Tracking and logging

### 9.1 Tracker

| Setting | Value |
|---|---|
| Library | WebGazer.js 3.5.3, vendored and self-hosted with MediaPipe face mesh and TF.js assets; TF.js model URLs patched to `/static/vendor/models` |
| Regression | ridge regression, WebGazer's Kalman filter on, `saveDataAcrossSessions(false)` |
| Eye features | each eye resized to 10 x 6 pixels (`resizeEye(e,10,6)`), 120 values at any camera resolution |
| Camera request | 640 x 480 ideal (min 320 x 240, max 1920 x 1080), `facingMode: user` |
| Calibration policy | `freeze` (default since 2026-08-17): WebGazer's mouse and click listeners are removed after calibration, so the model keeps its 27 calibration rows. `adapt` (stock, wave-1 default before the fix): quiz clicks and mouse moves keep training a 50-row window. `gaze_variant` records the policy that ran. |
| Third-party requests | none in the default configuration (the bundle contains tfhub.dev URLs only on non-default branches) |

### 9.2 Gaze samples

- The browser keeps at most one estimate per 33.3 ms (30 Hz), divides x and y
  by the viewport size, clamps them to [0, 1] and rounds to 4 decimals.
- Samples are buffered only during questions and never while the fullscreen
  overlay is shown; the paused interval is removed from the clock.
- Each sample is `{q, t, x, y, aoi}`: question index, milliseconds since the
  question was rendered, coordinates and the live AOI tag.
- Buffers flush every 2 s (at most 5,000 samples per request); on page unload
  `sendBeacon` sends chunks of 700; a failed batch stays in the buffer.
- The server stores `gaze_samples(student_id, q_index, t_ms, x, y, aoi)`,
  indexed on `(student_id, q_index)`, accepting x and y in [-0.5, 1.5].

### 9.3 AOI rectangles

For every question the client posts the bounding boxes of the question card and
of the answer regions (options A-D for multiple choice, A-B for yes/no, the text
field for fill-in and open), normalised to the viewport. They are posted at
render and again 150 ms after a resize, scroll, font load or image load; the
server keeps the last layout per student and question. Hit-testing inflates
each rectangle by 0.02 and checks options first, then the answer field, then the
question; everything else is `off`.

### 9.4 Events, answers and set-up metadata

| Record | Content |
|---|---|
| `events` | `fullscreen_exit`, `tab_hidden`, `window_blur`, `copy_attempt` (copy or context menu, both blocked), `mock_mode` at join; type, detail, question, server UTC time at 1-s resolution; fire-and-forget |
| `answers` | chosen option or text (at most 4,000 characters), correctness, `grade_source`, `time_ms` from render to *Next* excluding fullscreen pauses; first answer wins (a re-answer returns 409) |
| `students.client_info` | camera label, granted width/height/fps, maximum capabilities, `lightness` (mean luma of one 64 x 48 frame, 0-100; the frame is discarded), `track_hz`, `val_error`, `val_error_end`, calibration clicks, gaze policy, requested camera mode, browser user agent (up to 200 characters), browser locale, device pixel ratio; research keys only when a research option applies. Scalars only (at most 64 keys). |
| `survey_responses` | survey answers under the same id |
| `teacher_events` | teacher user id, section, time, type, detail: one row per dashboard **tab switch**, added 2026-08-20 (after wave 1). These are the teachers' personal data; tell every teacher. |
| `llm_reports` | model, input features, report JSON, `duration_ms`, `llm_used` |

`focus_lost` in the dashboard, the CSV and the LLM features is the count of
`fullscreen_exit`, `tab_hidden` and `window_blur` **events**, not episodes.

### 9.5 What is not stored

No video, frames, face images, face landmarks or templates; no table for them
exists, and the only binary table holds question snapshots cropped from the
teacher's PDF. The database stores no IP addresses. Note, however, that the client code (not
the API types) is what keeps images out: most fields are bounded, but the AOI
label is an unbounded string.

### 9.6 Research CSV data dictionary (89 columns, release 1.3.0)

| Theme | n | Columns |
|---|---|---|
| Identity and outcome | 7 | `section`, `anon_id`, `pseudonym` (optional personal code), `finished`, `score_pct` (pending open answers excluded from the denominator), `n_pending`, `focus_lost` |
| Camera and legacy validation | 11 | `camera`, `cam_width`, `cam_height`, `cam_fps`, `cam_max_width`, `cam_max_height`, `track_hz` (estimates per second in the 3-s start validation), `lightness`, `val_error`, `val_error_end`, `gaze_quality` (legacy label from a fixed v1.0 rule that was never fitted to data; not used in this README; prefer `track_hz` and `val_error`) |
| Tracker configuration | 5 | `cam_requested`, `gaze_variant` (policy that ran; `default` = legacy adaptive), `gaze_assigned` (A/B assignment), `cal_clicks`, `mock` |
| 5-point validation and geometry (1.3.0) | 18 | `val_points`, `acc_x`, `acc_y`, `acc_px`, `prec_x`, `prec_y`, `prec_px`, `prec_sd_px`, `acc_deg`, `prec_deg`, `acc_deg_end`, `prec_deg_end`, `drift_deg`, `min_aoi_x_deg`, `min_aoi_y_deg`, `geom_source` (estimated / measured), `px_per_cm`, `view_dist_cm` |
| Calibration experiment (1.3.0) | 5 | `cal_arm`, `cal_points`, `cal_samples`, `arm_fallback`, `cal_assigned` |
| Student extras | 2 | `overconfidence` (share of confident answers that were wrong; empty because the confidence prompt is disabled), `browser_lang` (browser locale, not the chosen UI language) |
| Item and answer | 10 | `q`, `qtype`, `bloom`, `chosen`, `answer_text` (up to 500 characters), `correct`, `pending`, `grade_source` (`auto`, `pending`, `llm`, `teacher`), `confidence`, `time_ms` |
| Gaze per item | 13 | `n_samples`, `off_share`, `transitions`, `rereads`, `first_option_ms`, `n_fixations`, `mean_fix_ms`, `dwell_question`, `dwell_A`, `dwell_B`, `dwell_C`, `dwell_D`, `dwell_answer` |
| Survey | 18 | `svy_` + 16 current items + 2 retired (`sys_helpful`, `sys_discuss`) |
| **Total** | **89** | |

Student-level columns repeat on each of the student's rows.

---

## 10. Analytics and visualisation

### 10.1 Processing pipeline

1. WebGazer's Kalman filter (client).
2. Rolling median over 5 samples per question (server, `SMOOTH_WINDOW`).
3. Re-tagging of the smoothed samples against the stored layout (0.02 padding).
4. Per-sample duration = gap to the next sample, capped at 500 ms; the last
   sample gets the median gap.
5. Metrics and I-DT fixation detection.

| Metric | Definition |
|---|---|
| Dwell share per AOI | summed sample durations per AOI / total; includes `off` |
| Off-layout share (`off_share`) | dwell share outside every question and answer region |
| Transitions | changes between distinct non-`off` AOIs after collapsing repeats (not saccades) |
| Re-reads | returns to the question after an option or answer AOI |
| First option (`first_option_ms`) | time to the first sample in an answer AOI |
| Fixations (I-DT) | dispersion (max-min x) + (max-min y) <= 0.10 of the window, onset-to-offset span >= 150 ms, never across a gap > 500 ms; centroid = sample mean; AOI = majority AOI |

At about 22 Hz (median inter-sample gap 46 ms) saccades cannot be resolved.
Fixation counts depend on the pipeline: resampling the same wave-1 data to a
25 ms grid raised Eye-Rec's own count from 46,394 to 75,927 (about 1.6x).
Compare fixation measures only within one pipeline and report the tracking rate
with them.

**Independent check.** `paper/pymovements_bridge.py` ran pymovements 0.28.0 on
the same wave-1 gaze (65 students, 1,533 student x question cells, 25 ms linear
grid, no smoothing on either side). Per-cell fixation counts correlated at
r = 0.989; with dispersion unit, minimum-duration and offset conventions
matched, r = 1.000 with 94% / 98% temporal overlap.

### 10.2 Views and their releases

| View | Tab | Release | Notes |
|---|---|---|---|
| Item statistics, % correct and time per question | Overview | 1.0 | |
| Group charts (score distribution, time vs score, off-layout vs score, re-reads) | Students | 1.1 | |
| Bloom profile, item discrimination | Overview | 1.2 | discrimination from n >= 6 |
| Accuracy by Bloom level, question map | Students | 1.2 (added 2026-08-17) | |
| Replay | Gaze replay | 1.0 | 960 x 600 canvas, 2x speed, 800 ms fading trail, fixation circles, per-sample heat layer **on by default** |
| Scanpath view | Gaze replay | **1.3.0** | fixations only: numbered circles (radius 0.6 * sqrt(duration), 4-40 px) in the AOI colour, arrowed transitions, no heat layer |
| AOI timeline | Gaze replay | **1.3.0** | AOI lane and fixation lane, gaps > 500 ms white, 1-s ticks |
| Class dwell per region | Gaze replay | **1.3.0** | median AOI rectangles shaded by mean dwell share with n; region-level, not a heat map |
| Gaze measurement card | Overview | 1.3.0 | accuracy, precision and a minimum AOI size when 5-point validation ran |
| Gaze profiles (exploratory) | Students | 1.3.0 | see 10.3 |

The scanpath, AOI-timeline and class-dwell views did not exist during wave 1;
wave-1 figures that use them are retrospective renderings.

![Scanpath view of one wave-1 student](images/fig_scanpath.png)

*Scanpath view (release 1.3.0) of one of the three wave-1 students whose start
validation error (e = 0.112) was below the option spacing; question 8, rendered
after the quiz from a copy of the snapshot, id masked. Even here the option
dwell shares in the side panel are uncertain.*

### 10.3 Gaze profiles

Exploratory k-means on six per-student features standardised within the class:
off-layout share, re-reads, fixations per question, mean fixation duration,
time z-score and drift (end minus start validation error; missing values
imputed with the median). Eligible: students with at least one answer and a
working start validation (non-zero tracking rate and a recorded error), at
least 8 of them. k-means++ with a fixed seed (42), 10 restarts, at most 100
iterations, k in {2, 3, 4} with k <= n/3, chosen by the highest positive mean
silhouette; every profile needs at least max(3, ceil(0.1 n)) members, otherwise
no profiles are shown. Names come from 11 fixed neutral keys (traits with
|z| >= 0.4; drift never names a profile).

Wave-1 profile numbers are **not reported here**: no committed script
regenerates them, and two earlier runs on different copies disagree. They must
be regenerated by `paper/report/profiles.py` (to be created) before being
cited. Mean fixation duration and re-reads correlate with the tracking rate,
and drift is a property of the tracker (section 14.12), so profiles on
unadjusted features partly describe laptops.

---

## 11. LLM analytics

### 11.1 Model and runtime

| Item | Value |
|---|---|
| Model | EuroLLM-9B-Instruct (9.154B parameters, developed for 35 languages including all 24 official EU languages, Apache-2.0) |
| Quantisation | GGUF `Q4_K_M` (mixed 4-bit k-quant, about 4.9 bits per weight), 5.58 GB, `hf.co/bartowski/EuroLLM-9B-Instruct-GGUF:Q4_K_M` |
| Runtime | Ollama sidecar container, no host port, `OLLAMA_KEEP_ALIVE=24h` |
| Settings | temperature 0.2, `num_ctx` 8192, `num_predict` 2000, no streaming, output constrained to a JSON schema (Ollama `format`) |
| Timeout | 1,200 s deployed, 300 s code default |
| Report languages | the seven interface languages (section 12); the fallback is English |
| Latency | 57 students: 1,243 s with all features (timeout, fallback); compact input 200-1,226 s; one warm run 393 s after a brevity instruction (section 6.2) |
| Model resolution | if the configured tag is missing, the best available EuroLLM (9B preferred), else the first model; the model used is stored with each report |

The model card states that EuroLLM-9B has not been aligned to human preferences
and may produce hallucinations or false statements. Its pre-training sequence
length is 4,096 tokens, whereas Eye-Rec requests an 8,192-token context; the
57-student full-feature prompt (about 4,800 tokens) exceeded the training length.

### 11.2 Input

A pseudonymous feature table (`analysis.section_features`), never raw gaze,
names or personal codes:

- Section: code, title, number of students, mean score, ids of students who
  joined but answered nothing (excluded from grouping).
- Per question: number, type, Bloom level, first 120 characters of the stem,
  % correct, mean time, mean re-reads, mean off-layout share (options and keys
  are not sent).
- Per student: `anon_id`, `score_pct`, `n_answered`, `time_z`, `rereads`,
  `focus_lost`, `wrong_q`, `off_share`, `weak_bloom`.
- **Compact mode** above 16 answering students (all of wave 1): per-student rows
  keep only `anon_id`, `score_pct`, `off_share`, `weak_bloom`; a class Bloom
  profile is added; the model lists at most five example ids per group.

The table is serialised as indented JSON: compact separators made the 9B model
produce incoherent reports in a live test.

### 11.3 Prompt rules and output schema

The system prompt asks for a 3-5 sentence summary, one group per student,
difficult questions and recommendations, at most 25 words per free-text field,
at most 6 difficult questions and 4 recommendations, and students named only by
id. Groups start from score bands (>= 80 understood, 50-79 partially
understood, < 50 not understood); the model may move a student to the adjacent
band only within 10 points of a boundary and is asked to cite gaze evidence.

Output shape (illustrative values):

```json
{
  "summary": "string",
  "clusters": [{"label": "understood | partially understood | not understood",
                "students": ["S01", "..."], "evidence": "string"}],
  "difficult_questions": [{"q": 17, "reason": "string"}],
  "recommendations": [{"topic": "string", "action": "string", "reason": "string"}]
}
```

### 11.4 Guard rails

A server pass (`_normalize_report`) drops invalid labels and unknown or
duplicate ids, merges duplicate labels, fills in omitted students from their
score band (this restores full membership in compact mode) and moves any
placement outside the adjacent-band, 10-point window back. The requirement to
cite gaze evidence is a prompt instruction only; the server does not check it.
The groups therefore remain essentially score-based; the LLM contributes the
narrative, not a gaze-based clustering.

### 11.5 Fallback

If Ollama is unreachable, times out or returns invalid JSON or schema,
`/analyze` still returns HTTP 200 with `llm: false`, `model: "fallback"` and a
deterministic report of the same shape (score-band groups; questions below 60%
correct, or the two hardest, as difficult). The dashboard shows "Rule-based
fallback used" with the error. The fallback is always in English, its text says
the LLM was "not reachable" even after a timeout, and the error string is not
stored with the report. An unexpected error outside these cases returns HTTP 500
without a stored report.

### 11.6 Storage and teacher questions

Each analysis is stored in `llm_reports` with model, input features, report,
`duration_ms` and `llm_used`, and can be deleted. The teacher can add a
free-text question (up to 2,000 characters), answered in a second call over the
same features and report (answer capped at 4,000 characters); if that call
fails, the structured report is still returned.

### 11.7 Open-answer grading

*Propose grades with the LLM* makes one call per `[OPEN]` question with all
answers the teacher has not graded (question, optional `MODEL:` answer and
`{anon_id, answer}` pairs; free-text answers do reach the self-hosted model).
The model returns correct/incorrect with a one-sentence reason, judging content
rather than spelling or language. Proposals take effect in the score at once,
labelled "Graded by: LLM"; the teacher can override any verdict with a note (up
to 400 characters), and an LLM write never overwrites a teacher grade. There is
no separate approval step. Wave 1 had no open questions, so this path was not
used with students.

### 11.8 Fine-tuning export and evaluation status

Two JSONL exports provide teacher-corrected (features -> report) pairs and
teacher-graded open answers in the runtime format, for later QLoRA
fine-tuning. No model has been fine-tuned. `tests/test_llm.py` (16 tests)
covers the plumbing with a mocked model. Report quality and LLM-teacher grading
agreement have **not** been evaluated; a rating study is planned.

---

## 12. Languages

The student interface, privacy notice, survey, teacher pages and reports are
available in seven of the 24 official EU languages: **Dutch, English, Finnish,
French, German, Italian and Spanish**. Each language defines the same 693 keys
across five files (`i18n.js`, `i18n-student2.js`, `i18n-teacher2.js`,
`i18n-review.js`, `i18n-system.js`); all six pages have the same selector.

- Selection order: `?lang=` parameter, stored choice, browser locale, English.
  Teachers and students choose independently. The student's chosen language is
  not stored (only the browser locale), so wave-1 language use is unknown.
- Reports follow the dashboard language; other codes fall back to English. JSON
  keys and group labels stay in English and are translated for display.
- English-only: quiz keywords, parser and API messages, two dashboard error
  strings, the fallback report, guard-rail texts and the grading prompt.
- Live model reports are documented for two of the seven languages; no
  native-speaker review of any translation is recorded, and the survey
  translations have no equivalence check.
- Adding a language takes 693 strings (about 4,400 English words) in 37
  translation objects across the five files, an option in six selectors, two
  hard-coded arrays used by parity checks, a `REPORT_LANGUAGES` entry in
  `eyerec/llm.py`, and plural rules for the ten missing EU languages that need
  more than one/other.

---

## 13. Ethics, data protection and the EU AI Act

**Approval and consent.** The institution's ethics committee approved the study
before data collection (reference withheld for review). Students received a
participant information sheet and consented before the session. The quiz did
not count towards grades. The in-app notice is a second layer, not a
substitute for the information sheet.

**Data minimisation.** No video, face images, landmarks or biometric templates
leave the browser; the server stores the records listed in section 9. Linked by
pseudonymous identifiers, optional personal codes and device metadata, these
records are **personal data** under the GDPR, not anonymous data.

**Retention and deletion.** Retention is teacher-controlled; nothing expires on
its own except login sessions. Teachers can delete a student (cascades to gaze,
layouts, answers, events and survey), a survey, empty rows, a saved report or a
whole section (after re-typing its code). Two limits are listed in section 16:
saved LLM reports keep a deleted student's data, and section deletion currently
fails once dashboard usage has been logged. Copies (backups, snapshots) must be
handled separately.

**Human oversight.** Outputs are teacher-facing and never shown to students;
reports record model, inputs, duration and engine; LLM grade proposals are
labelled and overridable, and teacher grades are never overwritten; the model
runs on the same rented server as the application, with no third-party API.

**EU AI Act.** The Act has prohibited systems that infer emotions in education
institutions since 2 February 2025, except for medical or safety reasons
(Regulation (EU) 2024/1689, Art. 5(1)(f)). The Commission's guidelines on
prohibited practices (C(2025) 5052 final, 29 July 2025, paras. 251 and 255)
place gaze-point tracking without emotion inference outside this ban, although
they list eye tracking as a behavioural-biometric modality. Eye-Rec performs no
emotion recognition, and gaze never enters a score. Annex III (point 3) treats
systems that evaluate learning outcomes (b) or detect prohibited behaviour
during tests (d) as high-risk; the Commission's draft guidelines on high-risk
classification (consultation draft of 19 May 2026) exclude formative learning
analytics whose outputs do not inform grades. Our use was formative. If scores
counted towards grades, point 3(b) would apply, and the per-student grouping,
likely a form of profiling, would rule out the Article 6(3) exemption. Under the
Digital Omnibus on AI (Regulation (EU) 2026/1744), Chapter III Sections 1-3
apply to Annex III systems from 2 December 2027.

**We make no compliance claim.** Each deploying institution should classify its
own use, determine the GDPR legal basis, decide whether a data-protection impact
assessment is needed and conclude a processing agreement with its host.

**Data release.** No student-level data are published. A de-identified release
builder exists (`paper/release/build_release.py`: drops free text, user agents,
camera names, tokens, absolute times and personal codes; k = 5 suppression audit
on four background variables, so demographics would be released only as
aggregates). It has not been deposited and needs the fixes in section 16 and
approval for public release first.

---

## 14. Data-collection results (wave 1)

### 14.1 Setting

One section of a master's course in machine learning at a European university,
17-18 August 2026, students' own laptops, one formative quiz of 25
Bloom-tagged items (15 multiple-choice, 5 fill-in, 5 yes/no; remember 5,
understand 6, apply 6, analyze 5, evaluate 2, create 1). The measurement
pipeline was the original one: 3 x 3 calibration with three clicks per dot
(27 clicks; the click count is recorded for 72 of 74 students), a 3-s centre
validation before the first and after the last question, and VGA camera
requests for all 74. The collection spanned a software
change: after the first 57 students the default calibration policy became
`freeze` (section 14.10). Numbers refer to the frozen snapshot of 18 August.

### 14.2 Participation and losses

| Measure | Value |
|---|---|
| Joined | 74 |
| Finished all 25 items | 71 (3 did not finish) |
| Gaze samples | 1,058,222 (all 74 students) |
| Score of finishers | mean 65.0% (SD 21.7), median 68% |
| Mean total time (dashboard) | 14 min 11 s |
| **Recording losses** | **7 of 74 (9.5%)**: 5 recordings produced no gaze (0 Hz), 2 never reached validation |
| Start validation available | 67 |
| Start and end validation available | 62 (5 of the 67 lack an end validation) |
| Surveys submitted | 61 |

The teacher deleted about 31 empty rows from abandoned re-joins, and deleted ids
are never reused, so anonymous ids run beyond 74; do not infer head counts from
ids. On 17 August one student answered every item but the finish call was
lost; since then `/api/finish` and `/api/client-info` are retried up to three
times.

| Bloom level | Items | Mean % correct |
|---|---|---|
| remember | 5 | 67.3 |
| understand | 6 | 70.9 |
| apply | 6 | 57.5 |
| analyze | 5 | 66.5 |
| evaluate | 2 | 64.1 |
| create | 1 | 56.3 |

### Webcam report

Sections 14.3-14.6 describe what laptop webcams delivered: accuracy, its
relation to the screen layout, tracking rate and cameras.

### 14.3 Validation error

The validation error *e* is the mean Euclidean distance of **all** gaze
estimates in the 3-s centre validation from the target (0.5, 0.5), with x as a
fraction of window width and y as a fraction of window height.

| Statistic | Value |
|---|---|
| n | 67 |
| Median | 0.218 |
| IQR | 0.169-0.260 |
| Range | 0.096-0.485 |
| Brightness vs error | Pearson r = 0.12, p = 0.319 (median brightness 56/100) |

![Distribution of validation error and tracking rate](images/fig_readme_validation.png)

*(a) Start validation error with the median option spacing (0.129), half of it
(0.064), the question-to-option-A distance (0.159) and the median (0.218).
(b) Tracking rate during the start validation. Drawn by
`images/make_readme_figures.py` from the aggregate files.*

Caveats: a single centre point (usually the most favourable region), including
the first landing period; units are anisotropic in pixels; wave 1 stored no
error direction.

### 14.4 Degrees (estimate)

Wave 1 measured neither viewing distance nor pixel size. Converting the median
error on the median window (1,470 x 732 px): e_px = e * sqrt((W^2 + H^2)/2) =
253 px.

| Assumption | Size on screen | Visual angle at 60 cm |
|---|---|---|
| 96 dpi | 6.7 cm | 6.4° |
| 14-inch screen, 31 cm wide | 5.34 cm | 5.1° |

So roughly 5-6°, an estimate under stated assumptions, not a measurement.
Release 1.3.0 adds the measured alternative (metric set-up and 5-point
validation).

### 14.5 Option spacing

From 900 stored laptop layouts (window wider than 900 px) with a question and
four options: median centre-to-centre option spacing 0.129 of the window
height, question centre to option A 0.159, half spacing 0.064.

| Error below | Students (of 67) |
|---|---|
| half spacing (0.064) | 0 |
| 0.10 | 2 |
| option spacing (0.129) | 3 |
| question to option A (0.159) | 13 |

The median error exceeds the option spacing, so the data support comparing the
question region with the answer region at class level, not identifying which
option a student read. The comparison is approximate: *e* is a 2-D distance,
the spacing is vertical, and the server re-tags a question's samples against the
last stored layout (a mid-question scroll can misattribute earlier samples).

### 14.6 Tracking rate and cameras

| Measure | Value |
|---|---|
| Tracking rate (start validation) | median 22 Hz (IQR 18.5-23.0; range 11.7-26.3; 2 below 15 Hz), n = 67 |
| Inter-sample gap during questions | median 46 ms (p5 34, p95 66) |
| Camera frame rate reported | 66 of the 67 with a validation reported 30 fps, 1 reported 60 fps (of all 72 with metadata: 71 and 1) |
| Granted resolution | 640 x 480 for 60, portrait 480 x 640 for 12, none for 2 |
| Distinct camera labels | 22 |
| Failed recordings (0 Hz) | 5; brightness 0, 2, 8, 26 and 72, so 4 of 5 had dark frames |

Although almost every camera reported 30 frames per second, tracking ran at a
median of 22 Hz, which suggests that laptop processing, not the camera, limits
the rate. Because WebGazer reduces each eye image to 10 x 6 pixels, better
cameras would probably add little; this is an inference from the code, not a
tested result. Camera labels are manufacturer strings, not reliable hardware
identifiers; device claims rest on the instructor's observation that students
used laptops.

### Drift report

Sections 14.7-14.10 compare the validation before the first and after the last
question.

### 14.7 Drift overall

Drift is the change of the validation error from the start to the end
validation (Δ = e_end − e_start; positive = worse).

| Statistic | Value |
|---|---|
| n (both validations) | 62 |
| Mean error | 0.224 → 0.285 |
| Mean drift | +0.061 (median +0.057, SD 0.102) |
| Worse at the end | 46 of 62 (74.2%) |
| Relative growth of the mean | 27.3% |
| Wilcoxon signed-rank | W = 367.5, p = 2.0 x 10^-5 |
| Effect size | d_z = 0.6 |

![Drift by camera-name group and over time on task](images/fig_drift.png)

*(a) Start and end validation error per student (thin) and camera-name group
means (thick). (b) Drift against minutes on the 25 questions.*

### 14.8 Drift by camera-name group

| Group (camera label) | Students in group | With both validations | Mean start → end | Mean drift | Worse |
|---|---|---|---|---|---|
| Apple (MacBook Air / FaceTime HD) | 20 | 18 | 0.231 → 0.284 | +0.052 | 12 of 18 |
| HP | 12 | 11 | 0.220 → 0.281 | +0.062 | 7 of 11 |
| Integrated Camera / Webcam | 14 | 9 | 0.186 → 0.254 | +0.067 | 9 of 9 |
| USB2.0 / Chicony | 9 | 9 | 0.212 → 0.276 | +0.064 | 7 of 9 |
| Other names | 17 | 15 | 0.248 → 0.314 | +0.066 | 11 of 15 |
| No camera reported | 2 | 0 | | | |

Mean drift varied little between groups (+0.052 to +0.067); with 9-18 students
per group, differences were not tested.

### 14.9 Drift and time on task

Time between the validations is approximated by the sum of answer times
(it excludes instructions and fullscreen pauses): median 13.8 minutes on the
questions (range 2.9-28.5). Drift was not associated with this time (Spearman
ρ = 0.11, p = 0.381, n = 62); median rate 0.0037 per minute (mean 0.0057). With
only two measurement points this is a start-end difference, not a drift curve;
whether drift comes from posture changes is a hypothesis.

### 14.10 Freeze vs adaptive calibration policy

| Policy | Students (of 74) | With both validations | Mean start → end | Mean drift | Worse |
|---|---|---|---|---|---|
| Adaptive (legacy) | 55 | 47 | 0.208 → 0.264 | +0.056 | 72% |
| Freeze (after the 17 August fix) | 19 | 15 | 0.275 → 0.351 | +0.076 | 80% |

Mann-Whitney p = 0.645. The split was not randomised: freeze students joined
later and started with larger errors. There is **no evidence that freezing the
calibration reduced drift**. A randomised comparison is available since
2026-08-20 (Research A/B option).

### 14.11 Event-detection limits

374 focus events (`fullscreen_exit`, `tab_hidden`, `window_blur`) from 22 of 74
students; median 0 per student, maximum 116 for one student. These are event
counts: one window switch may fire more than one event type, events are sent
without retry and time-stamped by the server at 1-s resolution. They describe
data quality (gaze is not sampled while the overlay is shown) and are not
proctoring.

### 14.12 Gaze features: reliability, set-up and learner

Records: 1,531 answered items with at least 20 gaze samples each, from 65
students with a working start validation. Per-student statistics use the 61
students with at least ten such items.

| Feature | Split-half (Spearman-Brown, odd vs even items, n = 61) |
|---|---|
| Time per question (log s) | 0.93 |
| Off-layout share | 0.94 |
| Re-reads per question | 0.91 |
| Transitions per second | 0.96 |
| Fixations per second | 0.97 |
| Mean fixation duration | 0.96 |
| Dwell share on the question | 0.92 |

| Feature (per-student mean, n = 61) | vs validation error | vs tracking rate | vs score |
|---|---|---|---|
| Transitions per second | −0.26 (p = 0.044) | **+0.50** (p < 0.001) | −0.14 |
| Mean fixation duration | +0.25 (p = 0.055) | **−0.48** (p < 0.001) | −0.14 |
| Re-reads per question | −0.08 | +0.31 (p = 0.016) | −0.07 |
| Off-layout share | +0.04 | −0.12 | **+0.40** (p = 0.001) |
| Fixations per second | −0.03 | +0.13 | **−0.49** (p < 0.001) |

Spearman ρ, uncorrected for 15 tests. Features are stable within a student, but
transition rate and fixation duration largely follow the laptop's tracking rate;
off-layout share and fixation rate relate to the score. Adjust gaze features for
tracking rate before clustering or prediction.

| Model (L2 logistic regression, item one-hot, 5 folds grouped by student) | AUC |
|---|---|
| Item difficulty only | 0.633 |
| + response time | 0.630 |
| + gaze features (no time) | 0.665 |
| + response time + gaze features | 0.663 |

Base rate 0.626 correct. The gain from gaze (+0.032) comes from one grouped
split with no confidence interval: a small, class-level signal, not a basis for
decisions about individuals.

![Correlations and prediction AUC](images/fig_readme_features.png)

*(a) Per-student Spearman correlations of gaze features with set-up and score.
(b) Cross-validated AUC for answer correctness.*

### 14.13 Gaze profiles

Computed by the dashboard (section 10.3); numbers withheld until
`paper/report/profiles.py` regenerates them from a committed script.

### 14.14 Student survey

Voluntary and skippable, shown right after the quiz, survey instrument v2 (16
items). 61 students submitted it. Likert items, 1 = strongly disagree to
5 = strongly agree; only means are reported.

| Item | Wording | Mean | n |
|---|---|---|---|
| `at_commit` | I am determined to complete this program successfully. | 4.75 | 59 |
| `at_interest` | The topics of this course interest me. | 4.55 | 60 |
| `sys_selfinsight` | Answering these questions showed me what I do and do not understand yet. | 4.14 | 57 |
| `sys_privacy` | I feel comfortable using the webcam-based system. | 4.05 | 57 |
| `sys_motivation` | Short exercises like this increase my motivation to engage with the course. | 4.21 | 56 |
| `sys_more` | I would like short exercises like this in more lessons. | 4.18 | 57 |
| `sys_discuss_wish` | I would like to see and discuss my results with my teacher. | 3.89 | 56 |

Background items (prior region, academic-English comfort, work hours) were
collected but are not published here. A k-anonymity check shows why: with only the four
background questions (prior region, years since the last degree, weekly work
hours, academic-English comfort), 26 of the 60 respondents who answered all four
had a unique combination of answers and 40 fell into groups of fewer than five
(`paper/release/build_release.py` suppresses such combinations). The repository
therefore shares aggregates only. These are self-reports collected in the instructor's own system
right after the quiz; they are not evidence of informed acceptance and do not
generalise beyond this class.

---

## 15. Regenerating numbers and figures

The wave-1 database is **not** in this repository. The analysis scripts read a
frozen snapshot (`paper/data/eyerec-2026-08-18.db`) that stays with the
research team until a release is approved; they open it read-only or work on a
temporary copy. The aggregate outputs are committed, and every README figure
can be redrawn from them.

```bash
.venv/bin/pip install numpy scipy matplotlib pillow

# with access to the snapshot (research team only)
.venv/bin/python paper/analyze.py                     # -> paper/stats.json
.venv/bin/python paper/report/wave1_numbers.py        # -> paper/report/wave1_numbers.json
.venv/bin/python paper/report/ml_feasibility.py       # -> paper/report/ml_feasibility.json
.venv/bin/pip install pymovements                     # dev-only
.venv/bin/python paper/pymovements_bridge.py --grid-ms 25   # -> paper/pymovements_compare.json
.venv/bin/python paper/report/profiles.py             # to be created -> profiles.json

# from the committed aggregates (anyone)
python images/make_readme_figures.py                  # fig_readme_validation.png, fig_readme_features.png
python paper/lak27/make_figures.py                    # fig_drift.png; currently also expects the scanpath capture (to be split)
```

| Output | Script | Input |
|---|---|---|
| cohort, Bloom, survey means (`stats.json`) | `paper/analyze.py` | snapshot |
| validation, rate, drift, degrees, layout (`wave1_numbers.json`) | `paper/report/wave1_numbers.py` | snapshot |
| reliability, correlations, AUC (`ml_feasibility.json`) | `paper/report/ml_feasibility.py` | copy of snapshot |
| fixation cross-check (`pymovements_compare.json`) | `paper/pymovements_bridge.py` | snapshot |
| `images/fig_readme_validation.png`, `fig_readme_features.png`, `readme_aggregates.json` | `images/make_readme_figures.py` | aggregate JSON |
| `images/fig_drift.png` | `paper/lak27/make_figures.py` | `wave1_numbers.json` |
| `images/fig_scanpath.png` | `paper/video/capture_fig.py` then `paper/lak27/make_figures.py` | local copy of the snapshot served on a throw-away server |
| `images/fig_input.png` | `paper/video/capture_input.py` then `paper/lak27/make_input_figure.py` | sample quiz on a throw-away server |
| `images/fig_overview.png`, `fig_students_masked.png`, `fig_consent.png` | showcase-video captures (`paper/video/record.py`) | copy of the snapshot / mock mode |
| paper PDF | `paper/lak27/build_paper.py` + LibreOffice (`render.sh`) | `paper_text.json`, `figs/` |

The scripts require numpy, scipy and matplotlib; the paper and video builders
use a separate environment with python-docx, Playwright, Piper and ffmpeg.

---

## 16. Known issues and limitations

Building and running Eye-Rec in a real class surfaced the gaps below. We list
them, with the intended fix where it is simple, so that anyone reusing the code
can plan around them. Items marked (P) are to be fixed before the repository is
published.

**Governance and consent**

- The privacy notice describes pseudonymous personal data as "anonymous" and
  says "no personal data are collected"; it mentions fullscreen and window
  events but not copy events, and says "browser version" where the full user
  agent is stored. The wave-1 notice did not mention public data release.
  Fix: say "pseudonymous", list every stored item and name any public release. (P)
- The in-app notice has no decline route and the server does not record the
  consent click; the student row and window size are created before the notice.
  Fix: add a camera-free exit and log the click.
- Saved LLM reports keep a deleted student's id and features. Fix: remove the
  student from stored reports on deletion; deletion must also reach backups and
  snapshots.
- Section deletion fails with a foreign-key error once `teacher_events` rows
  exist. Fix: delete those rows first and add a test. (P)
- The release builder derives ids from a fixed, published salt, and its k-audit
  covers only four background fields. Fix: secret salt and a wider audit before
  any deposit.

**LLM**

- The report prompt interprets a high off-layout share as "distraction or
  tracking loss" and many re-reads as "difficulty understanding the question";
  both are untested assumptions, and the first reads like attention inference.
  Replace with neutral wording (for example "gaze outside the item regions:
  tracking loss or looking away"). (P)
- LLM grade proposals count at once (override, not approval); the grading prompt
  hard-codes the students' semester.
- `num_ctx` 8192 exceeds the model's 4,096-token training length.
- The fallback is English-only, says "not reachable" after a timeout, and does
  not store the error.
- Non-ASCII quiz text reaches the report prompt as `\uXXXX` escapes.
- The dashboard's texts ("may take 1-3 minutes on CPU", "about 2 minutes once
  the model is warm") understate real class-size latency (section 6.2).

**Measurement and analytics**

- Fill-in matching is accent-insensitive, which can accept wrong answers where
  diacritics distinguish words (for example Finnish *säde*/*sade*).
- Fill-in keys are matched exactly after normalisation. In wave 1 the key of
  item 17 rejected 13 answers of "ridge regression"; accepting them would raise
  the item from 21% to 39% correct, still the lowest item. Fix: list every
  accepted variant in the key (or accept answers that contain a key term).
- Items without any gaze samples count as entirely off-region (100%), which
  inflates class off-region means and the item means sent to the model. Fix:
  store missing gaze as missing and leave it out of averages.
- A legacy quality label and its fixed threshold remain in the client constants,
  the dashboard column and the `gaze_quality` export column; they were never
  fitted to data and should be removed. (P for published figures)
- The server keeps only the last AOI layout per question.
- Gaze profiles cluster on unadjusted, rate-sensitive features.
- Greek PDF quizzes are shown almost entirely as snapshots (Greek letters are in
  the math-symbol set).

**Operations**

- Answers submitted during a deploy restart are lost; events are fire-and-forget.
- Personal codes are not checked against a roster.
- Dependencies and images are unpinned (`>=` in `requirements.txt`,
  `ollama/ollama`, `caddy:2`); the model is pulled by tag.
- uvicorn's default access log writes request paths (including media tokens) to
  the container log.
- No load test and no request-latency measurement exist.

**Stale documentation (P)**

- `eyerec/llm.py` docstring and `docs/LLM.md` name `llama3.2:3b` and a 300 s
  timeout; `docs/LLM.md` claims "native-quality" output in all languages.
- `docs/ARCHITECTURE.md` and `docs/API.md` give 300 s; `ARCHITECTURE.md` still
  lists "no images in questions" and claims a frame cannot pass the API.
- `docs/HOSTING.md` mentions an unused `TEACHER_KEY`, a smaller plan and
  EUR 8-15 a month; `docs/DEPLOY_HETZNER.md` gives EUR 0.70 for IPv4 and "about
  2 minutes" per warm analysis.
- `docs/CODE_MAP.md` describes one i18n file and four selectors.
- `RELEASE_NOTES_1.2.md` says "calibration drift fixed by default", which the
  wave-1 data do not support (section 14.10).
- The old top-level README claims about 100-200 px accuracy (measured median
  about 253 px) and links files that are not in the repository.

**Scope of the evidence**

One formative quiz in one section, run by the system's developers; a software
change during the collection; a non-randomised policy split; laptops only by
the instructor's account; no evaluation of effects on teaching or learning, of
LLM report quality or of the translations; the 1.3.0 research options and
calibration arms not yet used with real faces.

---

## 17. Showcase video

To be updated.

---

## 18. Licence and third-party notices

**Proposed licence: GPL-3.0-or-later.** The repository ships a modified copy of
WebGazer.js, which is licensed under GPL version 3 "or (at your option) any
later version" (the WebGazer README also offers LGPLv3 to companies valued
under USD 1M). A GPL-3.0-or-later licence for Eye-Rec keeps the combined work
distributable. The final decision belongs to the institution's legal office.

`THIRD_PARTY_NOTICES.md` (to be created) should list:

| Component | Location | Licence | Notes |
|---|---|---|---|
| WebGazer.js 3.5.3 | `eyerec/static/vendor/webgazer.js` | GPL-3.0-or-later | modified: TF.js model URLs patched from tfhub.dev to `/static/vendor/models`; the bundle has no version string; link the 3.5.3 source |
| MediaPipe face mesh assets | `eyerec/static/vendor/mediapipe/` | Apache-2.0 | vendored |
| TF.js face models | `eyerec/static/vendor/models/` | Apache-2.0 | vendored |
| Chart.js 4.4.7 | `eyerec/static/vendor/chart.umd.js` | MIT | header present |
| KaTeX 0.16.28 | `eyerec/static/vendor/katex/` | MIT | add licence file (no header in the minified file) |
| EuroLLM-9B-Instruct (GGUF) | downloaded at run time | Apache-2.0 | not redistributed |
| Python dependencies | `requirements.txt` | own licences | installed from PyPI; list after checking each |

---

## 19. How to cite

```bibtex
@inproceedings{anonymous2027eyerec,
  title     = {What Webcam Eye Tracking Can and Cannot Tell a Teacher About
               Formative Quizzes},
  author    = {Anonymous},
  booktitle = {Companion Proceedings of the 17th International Conference on
               Learning Analytics and Knowledge (LAK27)},
  year      = {2027},
  note      = {Practitioner report, under review},
  url       = {[https://anonymous.4open.science/r/[ID]](https://anonymous.4open.science/r/Web-Eye-Track-3589/README.md)}
}
```

Please also cite WebGazer (Papoutsaki et al., 2016, *WebGazer: Scalable webcam
eye tracking using user interactions*, IJCAI) and the EuroLLM-9B technical
report (arXiv:2506.04079) when you use those components.
