# Parsers, Topology & Telemetry Specification

This document defines how **Code My Resume (CMR)** extracts verified technical signals deterministically offline inside `src/cmr/parsers/`. All string outputs emitted by `src/cmr/parsers/` pass through `sanitizer.py` before persisting to `.cmr/scan_report.json`.

---

## 1. Manifest Parser (`manifest_parser.py`)

### 1.1 Supported Manifests, Workspaces & Python 3.10 Compatibility

| Ecosystem | Manifest File | Primary Keys Extracted | Parsing & Grouping Notes |
| :--- | :--- | :--- | :--- |
| **Node.js / TypeScript** | `package.json` | `name`, `dependencies`, curated `devDependencies`, `workspaces`, `scripts` | Built-in `json`. Recursively traverses npm/yarn `workspaces` globs (e.g., `apps/*`) and child-tier sub-manifests (`backend/`, `mobile/`) while ignoring build caches like `.next/`. |
| **Python** | `pyproject.toml`, `requirements.txt` | `[project.dependencies]`, `[project.optional-dependencies]`, `[tool.poetry.dependencies]`, pinned reqs | Uses `tomllib` (3.11+) with `tomli` fallback on Python 3.10. |
| **Android / JVM** | `build.gradle`, `build.gradle.kts`, `pom.xml` | `dependencies`, `ndk`, `compileSdk` / `targetSdk`, ProGuard/R8 flags | Regex/block parser (extracts SDK targets, R8 shrinking, native C++ deps). |
| **Flutter / Dart** | `pubspec.yaml` | `dependencies`, `dev_dependencies` | Uses `pyyaml`. |
| **Go / Rust / PHP / .NET** | `go.mod`, `Cargo.toml`, `composer.json`, `*.csproj` | `require`, `[dependencies]`, `[workspace.members]`, `PackageReference` | TOML/JSON/XML/line parsers. |
| **Monorepo Orchestrators** | `nx.json`, `turbo.json`, `pnpm-workspace.yaml` | `plugins`, `pipeline`/`tasks`, workspace globs, child `apps/*/project.json` | **Note:** Modern Nx (`nx v18+`) uses `plugins` + child `project.json` files and often omits a root `"projects"` key in `nx.json`. |
| **Container Runtimes** | `Dockerfile*` | `FROM <base-image>`, exposed ports, multi-stage builders | Extracts verified runtimes (`node:22-alpine`, `python:3.11-slim`). |

### 1.2 `devDependencies` Inclusion & `TAXONOMY_RULES`
Build/language tooling (`typescript`, `vite`), CSS engines (`tailwindcss`), and testing frameworks (`vitest`, `jest`, `@playwright/test`, `@testing-library/react`, `pytest`, `ruff`) routinely reside in `devDependencies` or `[project.optional-dependencies].dev`. `manifest_parser.py` therefore merges `dependencies` **and** `devDependencies` cleanly, matching lowercase normalized package names plus user `config.yaml` `taxonomy_overrides`:

```python
TAXONOMY_RULES = {
    # Languages & Tooling (merged with source file-extension LOC weights)
    "typescript": ("TypeScript", "Languages"),
    "vite": ("Vite", "Full-Stack & Mobile"),

    # Backend, RPC, Auth & Observability
    "@nestjs/core": ("NestJS", "Backend & Distributed"),
    "@nestjs/jwt": ("NestJS JWT Auth", "Security & Compliance"),
    "passport": ("Passport.js", "Security & Compliance"),
    "bcrypt": ("bcrypt", "Security & Compliance"),
    "@nestjs/throttler": ("NestJS Rate Limiting", "Security & Compliance"),
    "@grpc/grpc-js": ("gRPC", "Backend & Distributed"),
    "@nestjs/observe": ("NestJS Observe (APM)", "Backend & Distributed"),
    "nestjs-pino": ("Pino Structured Logging", "Backend & Distributed"),
    "@socket.io/redis-adapter": ("Socket.IO Redis Adapter", "Backend & Distributed"),
    "fastify": ("Fastify", "Backend & Distributed"),
    "express": ("Express.js", "Backend & Distributed"),
    "@apollo/server": ("Apollo GraphQL", "Backend & Distributed"),
    "bullmq": ("BullMQ Distributed Queues", "Backend & Distributed"),
    "socket.io": ("Socket.io / WebSockets", "Backend & Distributed"),

    # Frontend, State & Creative Graphics
    "next": ("Next.js", "Full-Stack & Mobile"),
    "react": ("React", "Full-Stack & Mobile"),
    "three": ("Three.js (WebGL)", "Full-Stack & Mobile"),
    "@react-three/fiber": ("React Three Fiber", "Full-Stack & Mobile"),
    "tailwindcss": ("Tailwind CSS", "Full-Stack & Mobile"),
    "@tailwindcss/postcss": ("Tailwind CSS v4", "Full-Stack & Mobile"),
    "gsap": ("GSAP", "Full-Stack & Mobile"),
    "lenis": ("Lenis Smooth Scroll", "Full-Stack & Mobile"),
    "framer-motion": ("Framer Motion", "Full-Stack & Mobile"),
    "zustand": ("Zustand", "Full-Stack & Mobile"),
    "@reduxjs/toolkit": ("Redux Toolkit", "Full-Stack & Mobile"),
    "@apollo/client": ("Apollo Client", "Full-Stack & Mobile"),
    "pdf-lib": ("pdf-lib", "Document Intelligence"),

    # Mobile, Native & Push Messaging
    "react-native": ("React Native", "Full-Stack & Mobile"),
    "expo": ("Expo SDK", "Full-Stack & Mobile"),
    "expo-notifications": ("Expo Notifications", "Full-Stack & Mobile"),
    "@shopify/react-native-skia": ("Shopify React Native Skia", "Full-Stack & Mobile"),
    "react-native-vision-camera": ("React Native Vision Camera", "Full-Stack & Mobile"),
    "@notifee/react-native": ("Notifee", "Full-Stack & Mobile"),
    "@react-native-firebase/app": ("Firebase Mobile SDK", "Full-Stack & Mobile"),
    "graphql-request": ("GraphQL Request", "Full-Stack & Mobile"),

    # Cloud SDKs, Databases & Queues
    "@aws-sdk/client-s3": ("AWS S3 SDK", "DevOps, Cloud & Deploy"),
    "firebase-admin": ("Firebase Admin / FCM", "DevOps, Cloud & Deploy"),
    "pg": ("PostgreSQL", "Databases & Data"),
    "typeorm": ("TypeORM", "Databases & Data"),
    "prisma": ("Prisma ORM", "Databases & Data"),
    "drizzle-orm": ("Drizzle ORM", "Databases & Data"),
    "ioredis": ("Redis", "Databases & Data"),
    "mongoose": ("MongoDB", "Databases & Data"),

    # Testing & Quality Assurance
    "@playwright/test": ("Playwright E2E", "Testing & QA"),
    "vitest": ("Vitest", "Testing & QA"),
    "jest": ("Jest", "Testing & QA"),
    "@testing-library/react": ("React Testing Library", "Testing & QA"),
    "pytest": ("Pytest", "Testing & QA"),
    "ruff": ("Ruff", "Testing & QA"),

    # Python, Document Intelligence & Computer Vision
    "reportlab": ("ReportLab", "Document Intelligence"),
    "pdfplumber": ("pdfplumber", "Document Intelligence"),
    "openpyxl": ("OpenPyXL", "Document Intelligence"),
    "mediapipe": ("MediaPipe", "Computer Vision"),
    "@mediapipe/tasks-vision": ("MediaPipe Tasks Vision", "Computer Vision"),
    "torch": ("PyTorch", "Machine Learning & AI"),
    "fastapi": ("FastAPI", "Backend & Distributed"),
    "pydantic": ("Pydantic v2", "Backend & Distributed"),
    "typer": ("Typer CLI", "Backend & Distributed"),
    "rich": ("Rich TUI", "Backend & Distributed"),
}
```

---

## 2. Structural Topology Detector (`topology_detector.py`)

CMR verifies disk structure before emitting architectural claims:

| Classification | Structural Evidence Checked on Disk | Grounded Terminology Output |
| :--- | :--- | :--- |
| **`distributed_microservices`** | Presence of `nx.json` / `apps/*` / `services/*` containing $\ge 2$ non-test (`!*-e2e`, `!*-test`) subdirectories with independent entrypoints (`src/main.ts`, `main.py`, `cmd/*/main.go`) or independent `Dockerfile`/`project.json`. | *"Architected N distributed microservices inside an Nx/workspace monorepo..."* |
| **`modular_monolith`** | Single application entrypoint (`src/main.ts` / `main.py`) with a root wiring file (e.g., `src/app.module.ts` importing `N` non-framework feature `*Module` classes, or modular Python packages). | *"Engineered a domain-driven modular architecture featuring N business domain modules..."* |
| **`polyglot_fullstack_workspace`** | Single `ProjectProfile` uniting `2+` runtime tiers: sibling `.git` repos (`*-backend`, `*-portal`, `*-web`, `*-admin`, `*-mobile`, `*-react-native`, `FluxRush*`), or unprefixed sub-apps (`backend/`, `mobile/`, `web/`), or `.code-workspace` suites. | *"Led end-to-end full-stack architecture spanning backend APIs, web portal, admin console, and mobile client..."* |
| **`standalone_app`** | Single web, mobile, or backend application (including single-app npm workspaces such as `apps/web/`). | *"Built and deployed [Web / Mobile / API] application..."* |
| **`cli_or_library`** | CLI entrypoint (`pyproject.toml` `[project.scripts]`, Go/Rust binary) or reusable library package. | *"Engineered modular CLI / library..."* |

---

## 3. Infrastructure & Deployment Detector (`infra_detector.py`)

Inspects deployment artifacts (never reading `.env` secrets) without assuming unverified orchestrators like Kubernetes:
* **Containers & Serverless Run:** `Dockerfile*`, `docker-compose*.yml`, `cloudbuild.yaml`, `service.yaml` (Cloud Run), `k8s/`, `helm/`, `Chart.yaml`.
* **Edge & Mobile CI/CD:** `vercel.json`, `netlify.toml`, `eas.json`, `app.json`, `app.config.*`, `.github/workflows/*.yml`, `fastlane/`, `android/app/build.gradle` (ProGuard/R8, NDK ABI/page alignment).
* **Server, Reverse Proxy & Process Management:** `ecosystem.config.*` (PM2 process counts, EC2 `/home/ec2-user` hints), `*.nginx.conf`, `*SSL*.md` runbooks, `Procfile`.
* **Cloud Services & FinOps:** Verified references to AWS (`EC2`, `RDS`, `ElastiCache`, `S3`, `CloudFront`), GCP (`Cloud Run`, `Pub/Sub`, `Secret Manager`, `FCM v1`), Redis TLS clusters, and PostgreSQL/PostGIS extensions.

---

## 4. Offline Documentation Parser & Full-File Discovery (`doc_parser.py`)

`doc_parser.py` inside `cmr.parsers` is **100% deterministic and offline** (zero LLM dependency during Phase 1 parsing). Optional LLM Map-Reduce digestion runs later via `cmr.engine.summarizer`.

### 4.1 Exhaustive Discovery & Case-Insensitive Noise Exclusion
* Discovers **ALL `*.md` files** across every workspace tree outside `walker.py`'s pruned directories (`node_modules`, `.next`, `temp/`, `vault/`, `baseline/`, `appstore-screenshots/`, `.agents/`).
* Applies **case-insensitive** exclusions (`SESSION_JOURNAL.md`, `TODO.md`, `to-do.md`, `apple-review-reply*.md`) and skips oversized raw dumps exceeding `config.scanner.max_file_size_bytes` (`250 KB` default) with a warning logged.

### 4.2 Dynamic Tiering & Anti-Recency Weighting
1. **Tier `1A` — Root & System Specs:** `README.md` (including sibling `*-portal`, `*-admin`, `*-mobile` docs), `ARCHITECTURE.md`, `SYSTEM_DESIGN.md`, `PRD.md`, `database-schema.md`.
2. **Tier `1B` — Domain, Runbook & Deployment Docs:** All remaining product/operational docs (`*RUNBOOK*.md`, `*SETUP*.md`, `*SSL*.md`, `*OBSERVABILITY*.md`, `*publishing*.md`, `*flow*.md`, `*rules*.md`).
3. **Tier `2` — Point-in-Time Logs & Audits (Capped Budget):** Case-insensitive match on `changelog`, `audit_summary`, `audit-summary`, `review`, `walkthrough`, `release_notes`.

### 4.3 Full-File Scanning & Heading-Boundary Chunking
* **100% Top-to-Bottom File Inspection:** Reads the complete content of every eligible `.md` file—never truncating after opening paragraphs.
* **Noise Stripping & Structural Split:** Removes base64 URIs, long JSON dumps, and fenced code blocks $> 30$ lines, then extracts deterministic regex metrics (`\d+%`, `ms`, `FPS`, `$`, `₹`, migration/module/screen/test counts), `candidate_org_hints`, and Markdown heading chunks (`H1`–`H3`, ~`10,000` chars) stored in `ProjectProfile.doc_chunks`.
* **Optional Map-Reduce (`engine/summarizer.py`):** When `cmr scan --summarize` or `cmr generate` runs, every chunk $\le 8,000$ chars passes in full; larger files run Map summarization per heading chunk and store digests in `.cmr/cache/` keyed by `(file_path, sha256(sanitized_content), model_id)`.

---

## 5. Multi-Repo Git Telemetry & Fork Exclusion (`git_parser.py`)

Because multi-repo suites (`GJI/`, `RME/`, `GestureGame/`) hold `.git` inside child directories rather than at the suite root:
1. **Child `.git` Aggregation:** Discovers all `.git` directories inside `ProjectProfile.root_path` and aggregates commit counts, author shares, earliest `first_commit_date`, and latest `last_commit_date` across every sub-repo.
2. **Author Attribution:** Matches `git shortlog -s -e --all` case-insensitively against `CareerContext.git_author_emails` (including GitHub `*@users.noreply.github.com` aliases) and git config `user.email`.
3. **Edge-Case Resilience:** Gracefully skips newly initialized 0-commit repos, handles detached HEAD states, and tolerates missing `git` binaries on minimal systems.
4. **Fork / Low Authorship Filter:** Repos with `< 5%` user commits set `is_fork_or_low_authorship = True`; their exclusive languages never enter core competencies unless explicitly whitelisted.

---

## 6. Deterministic Pre-Persistence & Pre-LLM Sanitizer (`sanitizer.py`)

`sanitizer.py` scrubs **all parser strings** (`doc_parser`, `infra_detector`, `topology_detector`) *before* `.cmr/scan_report.json` or `.cmr/cache/` is saved to disk, preventing hardcoded config secrets or commented CORS IPs (e.g., `http://103.x.x.x:3000` inside server bootstrap files) from leaking:

1. **Zero `.env` & Secret File Ingestion:** `.env`, `.env.*` (except `.env.example`/`.env.template`), `*.pem`, `*.key`, `*.p12`, `*.keystore`, `*.jks`, `google-services.json`, `GoogleService-Info.plist`, `*service-account*.json`, `id_rsa*`, `id_ed25519*`, `.npmrc`, `.pypirc` are never opened.
2. **Version-Safe Public IPv4 Redaction:** Validates `0–255` octets and excludes RFC 1918 (`10.x`, `172.16–31.x`, `192.168.x`), `127.0.0.1`, `0.0.0.0`, and dot-quad version/SDK strings (`v1.2.3.4`, `16.0.10`, `10.0.22621.1`) before rewriting server IPs to `[REDACTED_IP]`.
3. **Credential & Connection URI Redaction:** Redacts API keys (`AIza...`, `sk-...`, `ghp_...`, `github_pat_...`, `AKIA...`, `xoxb-...`), JWTs (`eyJ...`), `Authorization: Bearer ...`, DB URIs (`postgres://user:pass@host`), and SSH commands (`ssh user@host`) to `[REDACTED_TOKEN]`, `[REDACTED_JWT]`, `[REDACTED_URI]`, `[REDACTED_SSH]`.
