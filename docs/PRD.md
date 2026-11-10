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
| **Fractional CTO / Independent Consultant** | Needs to group multiple client repos cleanly (e.g., mobile app + logistics CLI under one brand) and hide sensitive/unverifiable employers. | `project_groups`, `anonymize_employer`, and `excluded_directories` inside `career_context.yaml`. |
| **Open-Source / Privacy-Conscious Developer** | Refuses to upload proprietary code or production runbooks to third-party SaaS resume websites. | 100% local CLI scanning, `sanitizer.py` IP/token redaction, `.cmr/cache/` incremental hashing, and optional local **Ollama** inference. |

---

## 3. CLI Command Contract (`cmr`)

All commands default to storing intermediate state inside `.cmr/` in the working directory so users don't have to pass repeated file paths.

### 3.1 `cmr scan <PATH>`
Recursively scans a local projects directory and generates `.cmr/scan_report.json`.

| Flag | Default | Description |
| :--- | :--- | :--- |
| `PATH` *(arg)* | Required | Root directory containing one or more repositories (e.g., `C:\Saikumar\Projects`). |
| `--output`, `-o` | `.cmr/scan_report.json` | Destination path for discovered `ProjectProfile` JSON list. |
| `--config`, `-c` | `config.yaml` | Optional YAML override for ignore dirs, token budgets, and author emails. |
| `--no-cache` | `False` | Bypass `.cmr/cache/` SHA-256 doc summary cache and re-digest all files. |
| `--offline` | `False` | Run deterministic manifest/topology/infra/git/regex extraction only (no LLM Map-Reduce pass). |

### 3.2 `cmr init-context`
Bootstraps a pre-populated `.cmr/career_context.yaml` from scan results so the user confirms timelines rather than writing YAML from scratch.

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--from-scan` | `.cmr/scan_report.json` | Input scan report used to pre-populate project mappings, git dates, and `candidate_org_hints`. |
| `--output`, `-o` | `.cmr/career_context.yaml` | Destination YAML file to review and customize. |
| `--interactive` | `False` | Prompt interactively for employment anchor bounds and excluded folders. |

### 3.3 `cmr generate`
Merges `scan_report.json` + `career_context.yaml`, synthesizes & validates `ResumeAST`, and renders requested targets.

| Flag | Default | Description |
| :--- | :--- | :--- |
| `--scan` | `.cmr/scan_report.json` | Verified scan report JSON. |
| `--context` | `.cmr/career_context.yaml` | User-reviewed career context YAML. |
| `--output`, `-o` | `./output` | Output directory for generated resume, GitHub, and LinkedIn files. |
| `--format`, `-f` | `pdf,md,html,github,linkedin` | Comma-separated export targets (`ast` writes `.cmr/resume_ast.json` only). |
| `--pages` | `2` | Target PDF page budget (`1` or `2`). |
| `--strict` | `True` | Fail non-zero if `validator.py` flags any topology, secret, or LinkedIn char-limit violation. |

### 3.4 `cmr diff`
Compares newly generated `README.md` (or `ResumeAST`) against an existing file on disk or a live GitHub profile repo before overwriting.

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
| `3` | PDF pagination budget failure (unable to fit `target_pages` or Page 1 density $< 88\%$ within font scale bounds). |

---

## 4. Configuration & Context File Contracts

### 4.1 `config.yaml` (Global Scanner & Engine Defaults)
```yaml
scanner:
  ignore_dirs:
    - node_modules
    - .git
    - .venv
    - venv
    - .yarn
    - build
    - dist
    - .gradle
    - .next
    - vendor
    - Pods
    - .bundle
    - __pycache__
    - temp
    - vault
    - baseline
    - appstore-screenshots
    - .agents
  exclude_doc_files:
    - SESSION_JOURNAL.md
    - TODO.md
    - to-do.md
    - apple-review-reply.md
    - apple-review-reply-5-1-1.md

doc_budget:
  full_file_max_chars: 8000
  map_chunk_chars: 10000
  tier1_digest_budget: 16000
  tier2_digest_budget: 3000

git:
  min_user_commit_share_pct: 5.0

llm:
  provider: gemini              # gemini | ollama | openai | anthropic
  model: gemini-2.5-flash

pdf:
  target_pages: 2
  min_page1_fill_ratio: 0.88
  min_last_page_fill_ratio: 0.65
```

### 4.2 `career_context.yaml` (User Grounding Example)
```yaml
full_name: "Sai Kumar Rayavarapu"
headline: "Lead Full-Stack Architect & Founding Engineer"
location: "Visakhapatnam, India"
phone: "+91 7036110147"
email: "sk.r.2270@gmail.com"
linkedin_url: "https://www.linkedin.com/in/sai-kumar-rayavarapu-167a6a2b0"
github_username: "skr2270"
git_author_emails:
  - "sk.r.2270@gmail.com"
ai_workflow_note: "Utilizes AI-augmented development workflows across research, architecture, testing, and deployment."

employment_anchors:
  - id: gji
    organization: "Gig Jobs India Private Limited"
    role_title: "Lead Full-Stack Architect"
    location: "Remote / Hybrid"
    start_date: "May 2024"
    end_date: "Present"
    mapped_project_dirs: ["GJI"]

  - id: fractional_cto
    organization: "Fractional CTO & Solutions Consultant"
    role_title: "Client Ventures"
    location: "Visakhapatnam, India"
    start_date: "May 2024"
    end_date: "Present"
    mapped_project_dirs: ["HC", "PDF Consolidator", "RME", "GestureGame", "UWA", "kikiphotography", "CodeMyResume"]
    project_groups:
      "The Horizon Collections": ["HC", "PDF Consolidator"]
    non_code_impacts:
      - "Configured Google Merchant Center profiles across 4 retail stores; trained staff on digital operations"
      - "Orchestrated vendor stack: Zoho POS, PhonePe, Razorpay, Shiprocket, Breeze (super.money), MyOperator, Bitespeed, Linkfliq (Hey Pipr)"

education:
  - degree: "Master of Science (M.S.) in Computer Science"
    institution: "Scaler Neovarsity / Woolf"
    accreditation: "MFHEA / ECTS, Europe"
    start_date: "2024"
    end_date_or_expected: "Expected: 2026 / 2027"
    specialization: "Advanced Distributed Systems, Scalable Cloud Architecture, Machine Learning, System Design"
  - degree: "Bachelor of Technology (B.Tech) in Computer Science & Engineering"
    institution: "Viswanadha Institute of Technology and Management (JNTUK)"
    end_date_or_expected: "Graduated: April 2022"
```

---

## 5. Acceptance Criteria for v0.1.0 / v0.2.0

1. Running `cmr scan` over a multi-gigabyte workspace with deep `node_modules`/`.gradle` trees finishes directory discovery in $< 2$ seconds via top-down pruning.
2. Scanning a modular NestJS app (`1 main.ts` + `21` modules) never emits `"microservices"`; scanning an Nx monorepo (`9 apps/*/main.ts`) accurately emits `"9 microservices"`.
3. `sanitizer.py` guarantees zero public IPv4s, `.env` secrets, or API keys reach cloud LLM prompts.
4. `pdf_exporter.py` outputs a balanced 2-page PDF with active hyperlinks, zero black-box glyph failures, and $\ge 88\%$ Page 1 vertical fill.
5. `linkedin_exporter.py` guarantees every generated block passes LinkedIn's `220` / `2,600` / `2,000` / `50` platform constraints.
