# Product Requirements & CLI Specification (`PRD.md`)

**Product:** Code My Resume (CMR)  
**Version:** `0.1.0`  
**Status:** Specification Baseline  

---

## 1. Product Vision & Mission

**Code My Resume (CMR)** eliminates the gap between what software engineers actually build and how they present their work. Instead of writing resume bullet points from memory—or letting generic AI tools hallucinate buzzwords—CMR connects directly to ground-truth codebases, manifests, deployment configs, git authorship, and documentation, merges them with human career timelines, and compiles a verified Single Source of Truth (`ResumeAST`) into synchronized career artifacts.

---

## 2. Target User Personas

| Persona | Core Pain Point | How CMR Solves It |
| :--- | :--- | :--- |
| **Lead Full-Stack / Founding Engineer** | Owns frontend, mobile, backend, infra, and business ops, but changelogs skew bullets toward recent hotfixes or one layer. | Polyglot workspace grouping + Tier 1A/1B/2 anti-recency weighting + `career_context.yaml` non-code leadership inputs. |
| **Senior Backend / Systems Architect** | Misclassification between distributed microservices vs. modular monoliths; outdated legacy skills creeping in from old resumes. | `topology_detector.py` structural verification + `validator.py` legacy-claim leak guards. |
| **Fractional CTO / Independent Consultant** | Needs to group multiple client repos cleanly (e.g., mobile app + logistics CLI under one brand) and hide sensitive/unverifiable employers. | `project_groups`, `anonymize_employer`, `denied_claims`, and `excluded_directories` inside `career_context.yaml`. |
| **Open-Source / Privacy-Conscious Developer** | Refuses to upload proprietary code or production runbooks to third-party SaaS resume websites. | 100% offline-first CLI scanning, `sanitizer.py` IP/token redaction, `.cmr/cache/` incremental hashing, and optional local **Ollama** inference. |

---

## 3. CLI Command Contract (`cmr`)

All commands store intermediate state inside `.cmr/` in the working directory by default so users don't have to pass repeated file paths.

### 3.1 `cmr scan <PATH>` *(Phase 1 — `v0.1.0`)*
Recursively scans a local projects directory deterministically offline and generates `.cmr/scan_report.json`.

| Flag | Default | Description |
| :--- | :--- | :--- |
| `PATH` *(arg)* | Required | Root directory containing one or more repositories (e.g., `~/Projects`). |
| `--output`, `-o` | `.cmr/scan_report.json` | Destination path for discovered `ProjectProfile` JSON payload. |
| `--config`, `-c` | `config.yaml` | Optional YAML override for ignore dirs, budgets, and git settings. |
| `--no-cache` | `False` | Bypass `.cmr/cache/` SHA-256 cache and re-digest all files. |
| `--summarize` | `False` | *(Phase 2)* Run LLM Map-Reduce doc summarization via `engine/summarizer.py` after deterministic extraction (never enabled without explicit opt-in). |

### 3.2 `cmr init-context` *(Phase 2 — `v0.2.0`)*
Bootstraps a pre-populated `.cmr/career_context.yaml` from scan results and emits `.cmr/career_context.schema.json` for editor validation.

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--from-scan` | `.cmr/scan_report.json` | Input scan report used to pre-populate project mappings, git dates, and `candidate_org_hints`. |
| `--output`, `-o` | `.cmr/career_context.yaml` | Destination YAML file to review and customize. |
| `--config`, `-c` | `config.yaml` | Global defaults YAML file. |
| `--interactive` | `False` | Prompt interactively for employment anchor bounds and excluded folders. |

### 3.3 `cmr generate` *(Phase 2 — `v0.2.0`)*
Merges `scan_report.json` + `career_context.yaml`, synthesizes & validates `ResumeAST`, and renders requested export targets.

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--scan` | `.cmr/scan_report.json` | Verified scan report JSON. |
| `--context` | `.cmr/career_context.yaml` | User-reviewed career context YAML. |
| `--config`, `-c` | `config.yaml` | Global LLM provider, taxonomy overrides, and PDF density settings. |
| `--output`, `-o` | `./output` | Output directory for generated resume, GitHub, and LinkedIn files. |
| `--format`, `-f` | `pdf,md,html,github,linkedin` | Comma-separated export targets (`ast` writes `.cmr/resume_ast.json` only). |
| `--pages` | `2` | Target PDF page budget (`1` or `2`). |
| `--page-size` | `A4` | Paper geometry (`A4` or `LETTER`), overriding `config.yaml`. |
| `--validate-only` | `False` | Re-run `validator.py` + exporters against an existing `.cmr/resume_ast.json` without invoking an LLM. |
| `--strict` | `True` | Fail non-zero if `validator.py` flags any topology, secret, or LinkedIn char-limit violation. |

### 3.4 `cmr diff` *(Phase 2 — `v0.2.0`)*
Compares newly generated `README.md` against an existing file on disk or a public GitHub profile repo before overwriting.

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--target`, `-t` | Required | Path to existing `README.md` or `<owner>/<repo>` GitHub target. |
| `--source`, `-s` | `./output/README.md` | Newly exported GitHub profile Markdown. |

### 3.5 Standard Exit Codes

| Code | Meaning |
| :--- | :--- |
| `0` | Success (all requested outputs generated and validated). |
| `1` | Runtime or I/O error (missing path, invalid YAML/JSON syntax). |
| `2` | Validation failure (`validator.py` caught topology error, ungrounded metric, secret leak, or LinkedIn char overflow in `--strict` mode). |
| `3` | PDF pagination budget failure (unable to fit `target_pages` or density bounds within font scale range). |

---

## 4. Configuration & Context File Contracts

### 4.1 `config.yaml` (Global Scanner & Engine Defaults)
See root [`config.yaml`](../config.yaml) for the live starter configuration matching this contract:
```yaml
scanner:
  ignore_dirs:
    - node_modules
    - .git
    - .venv
    - venv
    - env
    - .yarn
    - build
    - dist
    - .gradle
    - .next
    - .nuxt
    - .expo
    - .turbo
    - out
    - coverage
    - .nyc_output
    - target
    - vendor
    - Pods
    - .bundle
    - .dart_tool
    - __pycache__
    - .pytest_cache
    - .ruff_cache
    - temp
    - vault
    - baseline
    - appstore-screenshots
    - .agents
  exclude_doc_files:            # Case-insensitive filename exclusions
    - SESSION_JOURNAL.md
    - TODO.md
    - to-do.md
    - apple-review-reply.md
    - apple-review-reply-5-1-1.md
  max_file_size_bytes: 262144

doc_budget:
  full_file_max_chars: 8000
  map_chunk_chars: 10000
  chunk_token_estimate_chars: 3.5
  tier1_digest_budget: 16000
  tier2_digest_budget: 3000
  tier2_keywords: [changelog, audit_summary, audit-summary, review, walkthrough, release_notes]

git:
  min_user_commit_share_pct: 5.0

llm:
  provider: gemini              # gemini | ollama | openai | anthropic
  synthesis_model: gemini-2.5-flash
  map_model: gemini-2.5-flash-lite
  temperature: 0.1
  ollama_base_url: http://localhost:11434
  max_retries: 2

pdf:
  page_size: A4                 # A4 | LETTER
  target_pages: 2
  min_page1_fill_ratio: 0.88
  min_last_page_fill_ratio: 0.65
  font_family: Inter

taxonomy_overrides: {}
```

### 4.2 `career_context.yaml` (User Grounding Example)
See root [`career_context.example.yaml`](../career_context.example.yaml) for the complete annotated template (always keep personal contact fields out of version control):
```yaml
full_name: "Alex Rivera"
headline: "Lead Full-Stack Architect & Founding Engineer"
location: "Austin, TX"
phone: "+1 555-019-2834"
email: "alex@example.dev"
linkedin_url: "https://www.linkedin.com/in/alex-rivera-dev"
github_username: "alexriveradev"
git_author_emails:
  - "alex@example.dev"
  - "alexriveradev@users.noreply.github.com"
excluded_directories:
  - "legacy-archive-repo"
denied_claims:
  - "Keycloak"
  - "Kubernetes"
ai_workflow_note: "Utilizes AI-augmented engineering workflows across architecture design, test automation, and release verification."

employment_anchors:
  - id: primary_platform
    organization: "Nova Workforce Labs"
    role_title: "Lead Full-Stack Architect"
    location: "Remote / Hybrid"
    start_date: "May 2024"
    end_date: "Present"
    engagement_type: "Full-Time"
    mapped_project_dirs: ["nova-platform"]
    anonymize_employer: false

  - id: advisory_ventures
    organization: "Independent Ventures & Client Engagements"
    role_title: "Fractional CTO & Solutions Consultant"
    location: "Austin, TX"
    start_date: "May 2024"
    end_date: "Present"
    engagement_type: "Consulting"
    mapped_project_dirs: ["commerce-app", "pdf-logistics", "workforce-suite", "cv-arcade"]
    project_groups:
      "Apex Retail Group": ["commerce-app", "pdf-logistics"]
    non_code_impacts:
      - "Configured Google Merchant Center profiles across 4 retail stores; trained staff on digital operations"
      - "Orchestrated vendor stack: Zoho POS, PhonePe, Razorpay, Shiprocket, MyOperator, Bitespeed"
    anonymize_employer: false

education:
  - degree: "Master of Science (M.S.) in Computer Science"
    institution: "Woolf / Partner Institute"
    accreditation: "MFHEA / ECTS, Europe"
    end_date_or_expected: "Expected: 2026 / 2027"
    specialization: "Distributed Systems, Cloud Architecture, Machine Learning, System Design"
  - degree: "Bachelor of Technology (B.Tech) in Computer Science & Engineering"
    institution: "State Engineering University"
    end_date_or_expected: "Graduated: April 2022"
```

---

## 5. Acceptance Criteria for v0.1.0 / v0.2.0

1. Running `cmr scan` over a multi-gigabyte workspace with deep `node_modules`/`.gradle`/`.next` trees finishes directory discovery in $< 2$ seconds via top-down pruning.
2. Scanning a modular NestJS app (`1 main.ts` + `21` domain modules) never emits `"microservices"`; scanning an Nx monorepo (`9 apps/*/src/main.ts`, excluding `*-e2e`) accurately emits `"9 microservices"`, and workspaces using npm `workspaces` (`UWA`), unprefixed `backend/`/`mobile/` dirs (`HC`), or nested `.git` suites (`GJI`, `RME`, `GestureGame`) group cleanly.
3. `sanitizer.py` guarantees zero public IPv4s, `.env` secrets, or API keys reach `.cmr/scan_report.json`, `.cmr/cache/`, or cloud LLM prompts.
4. `pdf_exporter.py` outputs a balanced PDF matching `target_pages` on `A4` or `LETTER` with active hyperlinks, zero black-box glyph failures, $\ge 88\%$ Page 1 proportional fill, and $\ge 65\%$ final-page fill.
5. `linkedin_exporter.py` guarantees every generated block passes LinkedIn's `220` / `2,600` / `2,000` / `50` platform constraints.
