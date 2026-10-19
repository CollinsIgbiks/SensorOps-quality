# SensorOps — Project Brief

> **How to use this file:** paste or attach it at the start of a new AI chat, then say:
> *"Read this brief. Check my environment first, then guide me through the next milestone step by step."*
> Also commit it to each repo as `docs/PROJECT_BRIEF.md`.

---

## 0. Instructions for the AI assistant helping me

- **About me:** I'm Collins, an Electrical & Electronics Engineer (MSc) moving into software. I code in C/C++ and Python, and I've finished textbooks on Java and Go. I do freelance IoT and laser-comms work. I use VS Code, Git and GitHub.
- **Why I'm building this:** a public portfolio project for job applications in three areas: **Java Software Engineer**, **Software Development Engineer in Test (SDET)** and **AI & Automation Solution Consultant**. My CV already lists it as "In Progress", so it has to be real, working and explainable.
- **How to help me:**
  1. Before anything else, check my tools (Section 7). I'm on **Windows 11** with PowerShell.
  2. Work **one milestone at a time** (Section 6). Don't jump ahead.
  3. Explain **why** each design choice was made. I need to defend every part of this in interviews.
  4. Keep commits small and use Conventional Commit messages (`feat:`, `fix:`, `test:`, `ci:`, `docs:`).
  5. Never put secrets (API keys, passwords, tokens) in code or commits. Use `.env` files (git-ignored) and GitHub Secrets.
  6. Use current stable, LTS versions, and check them rather than guessing.
  7. At the end of each milestone, give me (a) a short README update and (b) the numbers I should record for my CV (Section 8).

---

## 1. What SensorOps is

**One-line pitch:** *SensorOps is an end-to-end IoT monitoring platform. It collects sensor data, uses AI to sort faults by urgency, and runs an automated test pipeline on every change. It shows the full journey from raw device data to a decision a person can act on.*

**The business problem:** facilities and engineering teams get flooded with raw sensor alerts and can't tell which ones matter. SensorOps stores the data reliably, flags readings that break their limits, uses AI to explain and rank each fault, and alerts a person only when something is critical.

**Three parts in three GitHub repos:**

| Part | Repo | Tech | Shows I can do |
|---|---|---|---|
| 1. Telemetry API | `sensorops-api` | Java 25, Spring Boot 4, PostgreSQL, Docker, GitHub Actions | Java backend, object-oriented design, databases, CI/CD, cloud |
| 2. AI Alert Triage | `sensorops-ai-triage` | n8n (low-code), generative AI, Python | AI use cases, low-code automation, pilots with measured results |
| 3. Quality Pipeline | `sensorops-quality` | Java 25, REST-Assured, Playwright, JUnit 5, k6, OWASP ZAP, Allure | Test automation, smoke/regression suites, CI/CD, performance and security testing |

GitHub user: **github.com/CollinsIgbiks**

### Version decisions (October 2026)
- **Java 25 (Eclipse Temurin)** instead of Java 21: Java 25 is the current LTS, and Oracle's free updates for JDK 21 ended in September 2026. Temurin is free with no licence questions.
- **Spring Boot 4.x** instead of 3.x: it's the current stable generation (built on Spring Framework 7).
- **Python 3.15**: the current stable release.
- **Maven Wrapper** (`mvnw`) instead of a global Maven install: everyone, including CI, builds with the same Maven version.

---

## 2. Architecture and data flow

```
 ┌───────────────────┐   POST /api/readings   ┌──────────────────────────┐
 │ Python Sensor     │ ─────────────────────▶ │ PART 1: Telemetry API    │
 │ Simulator         │                        │ Spring Boot + PostgreSQL │
 │ (in Part 2 repo)  │                        │                          │
 └───────────────────┘                        │ out-of-range reading?    │
                                              │  → create Alert (OPEN)   │
                                              │  → POST to n8n webhook   │
                                              └────────────┬─────────────┘
                                                           │ alert JSON
                                                           ▼
                                              ┌──────────────────────────┐
                                              │ PART 2: n8n AI Triage    │
                                              │ AI → category, urgency,  │
                                              │       plain-English note │
                                              │ → log to Google Sheets   │
                                              │ → Telegram/email if      │
                                              │   CRITICAL               │
                                              │ → PATCH verdict back     │
                                              │   to API /api/alerts/{id}│
                                              └──────────────────────────┘

 ┌────────────────────────────────────────────────────────────────────────┐
 │ PART 3: Quality Pipeline (GitHub Actions)                              │
 │ Starts the API (Docker) → REST-Assured API tests → Playwright UI tests │
 │ → k6 load test → OWASP ZAP scan → Allure report on GitHub Pages        │
 └────────────────────────────────────────────────────────────────────────┘
```

The loop closes because n8n writes the AI verdict back to the API, so the API holds the full history: reading → alert → AI triage.

---

## 3. Part 1: Telemetry API (`sensorops-api`)

### 3.1 Tech requirements
- Java 25 (LTS), Spring Boot 4.x, Maven (via the Maven Wrapper)
- Spring Web, Spring Data JPA, Bean Validation, Spring Boot Actuator
- PostgreSQL, with **Flyway** for database migrations
- springdoc-openapi for **Swagger UI** at `/swagger-ui.html`
- JUnit 5 + Mockito for unit tests, **Testcontainers** (real PostgreSQL in Docker) for integration tests, JaCoCo for coverage
- Dockerfile + `docker-compose.yml` (API + Postgres)
- GitHub Actions: build and test on every push and PR, then publish the Docker image to **GitHub Container Registry (GHCR)** on `main`

### 3.2 Layered design (object-oriented design evidence)
`controller` → `service` (business logic, behind interfaces) → `repository` (JPA) → `domain` (entities).
Use DTOs for requests and responses (never expose entities directly), plus one global exception handler that returns consistent error JSON.

### 3.3 Data model
- **Device**: `id`, `name`, `location`, `createdAt`
- **Threshold**: `id`, `deviceId`, `metric` (TEMPERATURE | HUMIDITY | VIBRATION), `minValue`, `maxValue`
- **Reading**: `id`, `deviceId`, `metric`, `value`, `recordedAt`
- **Alert**: `id`, `readingId`, `deviceId`, `status` (OPEN | ACKNOWLEDGED | RESOLVED), `aiCategory`, `aiUrgency` (LOW | MEDIUM | CRITICAL), `aiExplanation`, `createdAt`, `updatedAt`

### 3.4 Endpoints
| Method | Path | Purpose |
|---|---|---|
| POST | `/api/devices` | Register a device |
| GET | `/api/devices`, `/api/devices/{id}` | List or get devices |
| PUT | `/api/devices/{id}/thresholds` | Set min/max limits per metric |
| POST | `/api/readings` | Ingest a reading. If it's outside its threshold, create an Alert and notify n8n |
| GET | `/api/readings?deviceId=&metric=&from=&to=` | Query readings, paginated |
| GET | `/api/alerts?status=` | List alerts |
| PATCH | `/api/alerts/{id}` | Update status and AI verdict (used by n8n) |
| GET | `/actuator/health` | Health check (used by CI and Docker) |

### 3.5 Rules and acceptance criteria
- Invalid input returns **400** with field errors, an unknown ID returns **404**, a successful create returns **201** with a `Location` header.
- A reading outside its threshold **always** creates exactly one OPEN alert.
- The n8n webhook URL is configurable (`N8N_WEBHOOK_URL` env var). If n8n is down, the reading is still saved and the failure is logged.
- At least **80% line coverage** on the service layer (JaCoCo report in CI).
- `docker compose up` starts everything with one command.
- **Stretch goal:** deploy to a free cloud tier (e.g. Render or Railway) with a live Swagger URL in the README.

### 3.6 Agile evidence
Use a GitHub Projects board with columns Backlog / To Do / In Progress / Done. Write each feature as a user story ("As a facilities engineer, I want… so that…") with acceptance criteria. Work in one-week sprints and link each commit and PR to its issue.

### 3.7 Open design points (decide in the milestone named)
- **Milestones 4/6:** a stuck sensor reading a normal value never breaks a threshold, so it never raises an alert. Either add a "flatline" rule to the API or report this as a known limitation in `PILOT_RESULTS.md`.
- **Milestones 3/8:** protect `PATCH /api/alerts/{id}` (e.g. a shared API key header sent by n8n) so verdicts can't be rewritten by anyone.
- **Milestone 8:** the UI tests target saucedemo.com because SensorOps has no UI. Be ready to explain that, or add a small alerts dashboard later.

---

## 4. Part 2: AI Alert Triage (`sensorops-ai-triage`)

### 4.1 Tech requirements
- **n8n** self-hosted through Docker (free)
- AI model: **Ollama** running locally (free, no key needed), or a free-tier LLM API key kept in n8n credentials (never in the repo)
- Google Sheets (log), Telegram bot or Gmail (alerts)
- Python 3.15 for the simulator, in a virtual environment (`py -m venv .venv`)

### 4.2 Sensor simulator (`simulator/sensor_sim.py`)
- Registers 3–5 demo devices with thresholds through the API.
- Sends readings every few seconds, mostly normal, with an injected fault rate you can set (overheating, humidity spike, vibration from mechanical wear, and stuck or noisy sensors).
- Command-line options: number of devices, interval, fault rate, duration.
- Writes a **labelled CSV** of every injected fault (the "ground truth" used to measure AI accuracy).

### 4.3 n8n workflow
1. **Webhook** receives the alert from the API.
2. **AI node** gets the reading, the device, the threshold and the last few readings. It must return JSON: `{category, urgency, explanation}`, where the explanation is one sentence a non-engineer can understand.
3. **Validate** the AI output: if the JSON is invalid, fall back to `urgency: MEDIUM, category: UNCLASSIFIED`.
4. **PATCH** the verdict back to `/api/alerts/{id}`.
5. **Log** the alert and verdict to Google Sheets.
6. **IF CRITICAL**: send an instant Telegram/email alert.
7. **Schedule trigger** (daily): an AI-written summary of the day's alerts.

### 4.4 Pilot evaluation (this is the consultant-role evidence)
- Run the simulator for at least 500 readings.
- Compare the AI's categories and urgencies with the simulator's labelled CSV.
- Report: total readings, alerts raised, critical escalations, **category accuracy %**, **urgency accuracy %**, and examples of failures and what changed in the prompt.
- Write this up as `docs/PILOT_RESULTS.md`: problem → approach → results → limitations → recommendation.

### 4.5 Repo contents
`workflow/sensorops-triage.json` (exported, **with credentials removed**), `simulator/`, `docker-compose.yml`, `.env.example`, `docs/workflow.png`, `docs/demo.gif`, `docs/PILOT_RESULTS.md`, `README.md`.

---

## 5. Part 3: Quality Pipeline (`sensorops-quality`)

### 5.1 Tech requirements
- Java 25 + Maven (via the Maven Wrapper), JUnit 5
- **REST-Assured** for API tests against the SensorOps API
- **Playwright for Java** for UI tests against the public practice site **saucedemo.com**, using the **Page Object Model**
- **k6** for performance testing, **OWASP ZAP baseline** for the security scan
- **Allure** reports published to **GitHub Pages**
- **Jira** (free plan) for test cases and bug tracking

### 5.2 Test suites
- Tag tests with `@Tag("smoke")` or `@Tag("regression")`.
- **Smoke** (fast, under 2 min): health check, create device, post a reading, an alert is created for an out-of-range reading, UI login works.
- **Regression** (full): every endpoint, validation errors (400), not found (404), pagination, threshold edge cases (values exactly at min/max), alert status transitions, and UI flows (login failures, add to cart, checkout).
- Test data is created and cleaned up by the tests themselves, with no dependency on test order.

### 5.3 CI (GitHub Actions)
- The pipeline pulls the API image from GHCR and starts API + Postgres with Docker Compose, then waits for `/actuator/health`.
- **On every push and PR:** smoke suite.
- **Nightly (cron):** full regression, plus k6 load test and ZAP baseline scan.
- Publish the Allure report to GitHub Pages and link it from the README.

### 5.4 Performance and security
- **k6:** ramp to, for example, 50 virtual users posting readings. Record p95 response time and error rate, with thresholds that fail the build if they're breached.
- **ZAP baseline:** run against the API and record findings, then fix or document them.

### 5.5 Testing types covered (maps to the SDET job description)
White-box (unit tests in Part 1) · black-box (API and UI here) · integration and end-to-end (Testcontainers + Docker Compose) · performance (k6) · UI (Playwright) · security (ZAP) · smoke, sanity and regression (tags + CI schedule).

---

## 6. Milestones (build in this order)

| # | Milestone | Done when |
|---|---|---|
| 1 | **Setup:** create 3 repos, profile README, GitHub Projects board, copy this brief into `docs/` | Repos public, each with a README stub and first commit |
| 2 | **API core:** devices, thresholds, readings, Postgres + Flyway, Docker Compose | `docker compose up` works and endpoints work in Swagger |
| 3 | **API alerts + tests:** alert logic, webhook call, unit and Testcontainers tests, CI, GHCR image | CI green, coverage ≥ 80% on services |
| 4 | **Simulator:** Python simulator + labelled CSV | Streams readings into the API and alerts appear |
| 5 | **n8n triage:** full workflow + write-back + Sheets + alerts | A critical fault reaches my phone with an AI explanation |
| 6 | **Pilot evaluation:** 500+ readings, accuracy measured, `PILOT_RESULTS.md` | Numbers recorded |
| 7 | **Quality: API tests:** REST-Assured smoke + regression running in CI | Smoke on push, regression nightly |
| 8 | **Quality: UI, performance, security:** Playwright POM, k6, ZAP, Allure on Pages | Report link live in README |
| 9 | **Polish:** READMEs, diagrams, demo GIFs, optional cloud deploy | A recruiter understands each repo in 60 seconds |

Rough timeline: about 6 weekends.

---

## 7. My environment

Checked and working (October 2026, Windows 11):

| Tool | Version / notes |
|---|---|
| JDK | Eclipse Temurin 25, `JAVA_HOME` set |
| Maven | Not installed globally; each project uses the Maven Wrapper (`mvnw.cmd`) |
| Python | 3.15; use `py -m pip` (bare `pip` isn't on PATH) |
| Docker | Docker Desktop on WSL 2; `hello-world` runs |
| Git | Installed; commits use my GitHub noreply email |
| GitHub | Authenticated from my PC |
| VS Code | 1.141.0, with: Extension Pack for Java, Spring Boot Extension Pack, Python, Docker, GitHub Actions |

Still needed later: **k6**, an Ollama model or an AI API key, a Telegram bot token, a Google account for Sheets, and the Jira free plan.

---

## 8. What to record for my CV

Record these as I go, as **real numbers only**:
- API: number of endpoints, test count, coverage %, CI build time, live URL (if deployed)
- AI triage: readings processed, alerts raised, critical escalations, category and urgency accuracy %, prompt iterations
- Quality: number of automated tests (API / UI), smoke run time, k6 p95 latency and error rate, ZAP findings fixed
- Agile: number of sprints, user stories completed

These replace the "In Progress" bullets on my three CVs once each part is done.

---

## 9. README template (each repo)

1. **SensorOps: [Part name]** + one-line description + where it fits (link to the architecture diagram)
2. The problem it solves
3. Architecture / workflow screenshot
4. How to run it (copy-paste commands)
5. Tests and CI (badges, Allure link)
6. Results (real numbers)
7. What I'd do next
