# Parsers, Topology & Telemetry Specification

This document defines how **Code My Resume (CMR)** extracts verified technical signals across manifests, directory topology, deployment infrastructure, documentation, and git history inside `src/cmr/parsers/`.

---

## 1. Manifest Parser (`manifest_parser.py`)

### 1.1 Supported Manifests & Python 3.10 Compatibility

| Ecosystem | Manifest File | Primary Keys Extracted | Parsing Notes |
| :--- | :--- | :--- | :--- |
| **Node.js / TypeScript** | `package.json` | `name`, `dependencies`, `devDependencies`, `workspaces`, `scripts` | Built-in `json` |
| **Python** | `pyproject.toml`, `requirements.txt` | `[project.dependencies]`, `[tool.poetry.dependencies]`, pinned reqs | Uses `tomllib` (3.11+) with `tomli` fallback on Python 3.10 |
| **Go** | `go.mod` | `module`, `require (...)` directives | Line/block parser |
| **Rust** | `Cargo.toml` | `[dependencies]`, `[workspace.members]` | Uses `tomllib`/`tomli` |
| **PHP** | `composer.json` | `require`, `require-dev` | Built-in `json` |
| **Monorepo Orchestrators** | `nx.json`, `turbo.json`, `pnpm-workspace.yaml` | `projects`, workspace package globs | YAML / JSON |

### 1.2 Dependency Taxonomy Mapping (`TAXONOMY_RULES`)

Raw package identifiers are normalized into structured skill domains:

```python
TAXONOMY_RULES = {
    # Backend, RPC & Observability
    "@nestjs/core": ("NestJS", "Backend & Distributed"),
    "@grpc/grpc-js": ("gRPC", "Backend & Distributed"),
    "@nestjs/observe": ("NestJS Observe (APM)", "Backend & Distributed"),
    "@socket.io/redis-adapter": ("Socket.IO Redis Adapter", "Backend & Distributed"),
    "fastify": ("Fastify", "Backend & Distributed"),
    "express": ("Express.js", "Backend & Distributed"),
    "@apollo/server": ("Apollo GraphQL", "Backend & Distributed"),
    "bullmq": ("BullMQ Distributed Queues", "Backend & Distributed"),
    "socket.io": ("Socket.io / WebSockets", "Backend & Distributed"),

    # Frontend & 3D Web
    "next": ("Next.js", "Full-Stack & Mobile"),
    "react": ("React", "Full-Stack & Mobile"),
    "three": ("Three.js (WebGL)", "Full-Stack & Mobile"),
    "@react-three/fiber": ("React Three Fiber", "Full-Stack & Mobile"),
    "tailwindcss": ("Tailwind CSS", "Full-Stack & Mobile"),
    "gsap": ("GSAP", "Full-Stack & Mobile"),
    "lenis": ("Lenis Smooth Scroll", "Full-Stack & Mobile"),

    # Mobile & Graphics
    "react-native": ("React Native", "Full-Stack & Mobile"),
    "expo": ("Expo SDK", "Full-Stack & Mobile"),
    "@shopify/react-native-skia": ("Shopify React Native Skia", "Full-Stack & Mobile"),
    "react-native-vision-camera": ("React Native Vision Camera", "Full-Stack & Mobile"),

    # Databases & Caching
    "pg": ("PostgreSQL", "Databases & Data"),
    "typeorm": ("TypeORM", "Databases & Data"),
    "ioredis": ("Redis", "Databases & Data"),
    "mongoose": ("MongoDB", "Databases & Data"),

    # Python & Document / CV Systems
    "reportlab": ("ReportLab", "Document Intelligence"),
    "pdfplumber": ("pdfplumber", "Document Intelligence"),
    "mediapipe": ("MediaPipe", "Computer Vision"),
    "torch": ("PyTorch", "Machine Learning & AI"),
    "fastapi": ("FastAPI", "Backend Architecture"),
    "pydantic": ("Pydantic", "Backend Architecture"),
}
```

---

## 2. Structural Topology Detector (`topology_detector.py`)

**Why this exists:** Documentation prose can be ambiguous (e.g., referring to *"21 internal business services"* inside a single modular NestJS server vs. *"9 standalone microservices"*). CMR **always verifies structural code topology before generating architectural claims**:

| Classification | Structural Evidence Checked on Disk | Grounded Terminology Output |
| :--- | :--- | :--- |
| **`distributed_microservices`** | Presence of `nx.json` / `apps/*` (or `services/*`) where $\ge 2$ subdirectories each contain their own independent entrypoint (`src/main.ts`, `main.py`, `cmd/*/main.go`) or independent `Dockerfile` / `package.json`. | *"Architected N distributed microservices inside an Nx/workspace monorepo..."* |
| **`modular_monolith`** | Single application entrypoint (`src/main.ts`) importing multiple feature modules (`src/app.module.ts` + `src/<domain>/*.module.ts` or Django/FastAPI feature apps). | *"Engineered a domain-driven modular architecture featuring N business domain modules..."* |
| **`polyglot_fullstack_workspace`** | Parent project directory containing sibling repositories/subprojects matching multi-tier roles (`*-backend`, `*-portal`/`*-web`, `*-admin`, `*-mobile`). | *"Led end-to-end full-stack architecture spanning backend APIs, web portal, admin console, and mobile client..."* |
| **`standalone_app`** | Single web, mobile, or backend application with a unified root manifest. | *"Built and deployed [Web / Mobile / API] application..."* |

---

## 3. Infrastructure & Deployment Detector (`infra_detector.py`)

Engineers frequently own full production deployments (`Docker`, `AWS EC2/RDS/ElastiCache`, `Google Cloud Run`, `PM2`, `Nginx`, `Netlify`, `Vercel`, `EAS Build`), which standard dependency parsers miss. `infra_detector.py` inspects configuration artifacts and runbooks **without assuming unused orchestrators (like Kubernetes) unless active deployment manifests verify them**:

* **Containers & Serverless Run:** `Dockerfile*`, `docker-compose*.yml`, `cloudbuild.yaml`, `service.yaml` (Google Cloud Run), `k8s/`, `helm/`, `Chart.yaml`.
* **Edge & Mobile CI/CD:** `vercel.json`, `netlify.toml`, `eas.json`, `.github/workflows/*.yml`, `fastlane/`, `android/app/build.gradle` (e.g., 16 KB ELF page alignment, ProGuard/R8).
* **Server, Reverse Proxy & Process Management:** `ecosystem.config.*` (PM2), `*.nginx.conf`, `Nginx*.md` runbooks, `Procfile`.
* **Cloud Services & FinOps:** Extracts verified references to AWS (`EC2`, `RDS`, `ElastiCache`, `S3`, `CloudFront`, FinOps cost analyses), GCP (`Cloud Run`, `Pub/Sub`, `Secret Manager`, `FCM v1`), Redis TLS clusters, and PostgreSQL/PostGIS extensions from deployment configs, runbooks, or `.env.example` (never reading actual `.env` secrets!).

---

## 4. Documentation Parser & Exhaustive `.md` Discovery (`doc_parser.py`)

### 4.1 Exhaustive Discovery & Scratch Noise Exclusion
Hardcoding an allowlist like `README.md` or `ARCHITECTURE.md` misses critical architectural, deployment, and domain docs (e.g., `Nginx_SSL.md`, `AWS_SETUP.md`, `RABBITMQ_ANALYSIS.md`, `OBSERVABILITY.md`, `kyc-verification-flow.md`, `playstore-publishing.md`, `appstore-publishing.md`, `consolidation_rules.md`, `BACKEND_DEPLOYMENT_FCM_RUNBOOK.md`).

Instead, `doc_parser.py` discovers **ALL `*.md` files** across every project tree (outside `walker.py`'s pruned ignore list) **before** `career_context.yaml` is bootstrapped, while skipping noisy scratch/journal files:
* **Excluded Scratch Files:** `SESSION_JOURNAL.md`, `TODO.md`, `to-do.md`, `apple-review-reply*.md`, and anything under `temp/`, `vault/`, `baseline/`, `appstore-screenshots/`, or `.agents/`.

### 4.2 Dynamic Tiering & Anti-Recency Weighting
To prevent recent changelogs or audits from drowning out years of core full-stack engineering, every discovered `.md` file is dynamically classified into weighted tiers:

1. **Tier 1A — Root Overview & System Specs (Highest Weight):**
   * Root `README.md` (and sibling repo READMEs like `*-portal/README.md`, `*-admin/README.md`, `*-mobile/PROJECT_DOCUMENTATION.md`), `ARCHITECTURE.md`, `SYSTEM_DESIGN.md`, `PRD.md`, `database-schema.md`, `openapi.json`/`schema.gql`.
2. **Tier 1B — Domain, Runbook & Deployment Docs (High Weight):**
   * All remaining product/operational `.md` files (e.g., `*RUNBOOK*.md`, `*SETUP*.md`, `*SSL*.md`, `*OBSERVABILITY*.md`, `*ANALYSIS*.md`, `*publishing*.md`, `*flow*.md`, `*rules*.md`).
3. **Tier 2 — Point-in-Time Logs & Audits (Capped Secondary Weight):**
   * Files matching `changelog`, `audit_summary`, `review`, `walkthrough`, or `release_notes`.

### 4.3 Full-File Signal Extraction & Map-Reduce Chunking (Zero Deep-File Drops)
Reading only opening paragraphs or cutting off large docs after the first few lines misses crucial architecture, security, and FinOps wins buried deep inside 20 KB–60 KB manuals (e.g., `GJI/docs/ARCHITECTURE.md` or `HC/docs/strategy/IMPLEMENTATION_PLAN.md`). `doc_parser.py` therefore processes **100% of every discovered `.md` file** in two stages:

1. **Stage A — Full-File Deterministic Extraction (Python, Zero Line Cutoff):**
   * Scans the **entire file** top-to-bottom to extract regex-verified quantitative metrics (`\d+%`, `ms`, `FPS`, `$`, `₹`, migration/module/screen/test counts), URLs/domains, cloud services, and **`candidate_org_hints`** (company names, store counts, client names) to pre-fill `career_context.yaml`.
   * Strips base64 images, long JSON payloads, and code blocks $> 30$ lines to eliminate token bloat.
2. **Stage B — Adaptive Map-Reduce Summarization & SHA-256 Digest Cache:**
   * Files under **8,000 chars** are passed **in full**.
   * Larger Tier 1A/1B files are split along Markdown heading boundaries (`H1`–`H3`) into ~10,000-char chunks and summarized via a fast **Map pass** across *every* chunk (from top to bottom) before merging into `ProjectProfile`, ensuring zero silent truncation while respecting configurable per-project synthesis budgets (default: **16,000 chars** across Tier 1A/1B digests + **3,000 chars** for Tier 2).
   * **Incremental Digest Cache (`.cmr/cache/`):** Caches chunk summaries keyed by `(file_path, sha256(content), model_id)` so re-running `cmr scan` after editing a single project reuses unchanged doc digests at `$0` incremental API cost.

---

## 5. Git Telemetry & Fork Exclusion (`git_parser.py`)

Filesystem modification timestamps (`mtime`) frequently reflect file copies or sync events rather than real development history. `git_parser.py` relies on git commit logs instead:

1. **Active Date Range:** Queries `git log --reverse --format=%ad --date=short` (first commit) and `git log -1 --format=%ad --date=short` (latest commit), bounded by `career_context.yaml` employment anchors.
2. **Author Commit Share & Fork Detection:**
   * Inspects `git shortlog -sn --all` matching against `career_context.git_author_emails` / git config `user.email`.
   * If a repository is a forked tool or has **$< 5\%$ user-authored commits**, sets `is_fork_or_low_authorship = True`.
   * **Downstream Rule:** Languages/frameworks exclusive to `is_fork_or_low_authorship` repos (for example, a forked Go OSINT utility) are **never** promoted into the resume's Core Technical Competencies unless explicitly whitelisted by the user.

---

## 6. Pre-LLM Secret, IP & PII Sanitizer (`sanitizer.py`)

Production runbooks and security audit docs frequently reference live server IPv4 addresses, SSH commands, storefront tokens, JWTs, or sandbox credentials. Before any extracted digest leaves disk for a cloud LLM (`google-genai`, OpenAI, Anthropic), `sanitizer.py` enforces:

1. **Zero `.env` & Secret File Ingestion:** `.env`, `*.pem`, `*.key`, `google-services.json`, and `GoogleService-Info.plist` are never read.
2. **Deterministic Regex Redaction:**
   * Public IPv4 addresses (e.g., `103.x.x.x`, `18.x.x.x`) $\rightarrow$ `[REDACTED_IP]`
   * API keys & tokens (`AIza...`, `sk-...`, `ghp_...`, `eyJ...` JWTs, hex storefront tokens) $\rightarrow$ `[REDACTED_TOKEN]`
   * Database URIs / passwords (`postgres://user:pass@host`) $\rightarrow$ `[REDACTED_URI]`
3. **Post-Synthesis Leak Verification:** Cross-checked by `validator.py` so zero redacted placeholders or credential patterns ever land inside `ResumeAST` outputs.
