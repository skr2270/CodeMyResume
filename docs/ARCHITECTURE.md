# System Architecture — Code My Resume (CMR)

This document specifies the technical design, package structure, Pydantic schemas, and rendering pipeline for **Code My Resume (CMR)**.

---

## 1. High-Level Scan-First 5-Stage Pipeline

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   1. SCANNER     │ ──> │   2. PARSERS     │ ──> │ 3. BOOTSTRAP &   │ ──> │  4. SYNTHESIZER  │ ──> │   5. EXPORTERS   │
│  Pruned Walker   │     │ Topology/Infra/  │     │  CONTEXT MERGE   │     │  LLM + Validator │     │  Single-Source   │
│  & Repo Detector │     │ ALL *.md & Git   │     │career_context.yml│     │  -> ResumeAST    │     │ PDF/MD/HTML/GH   │
└──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────────┘
```

1. **Scanner Stage (`cmr.scanner`):** Runs **first** via `os.walk(..., topdown=True)` with immediate directory pruning (`node_modules`, `.venv`, `.git`, `build`, `dist`, `.gradle`, `temp/`, `vault/`, `baseline/`, `appstore-screenshots/`, `.agents/`) to discover every repository and workspace boundary.
2. **Parsers, Sanitizer & Detectors Stage (`cmr.parsers`):** Inspects **all `*.md` files** across every project, structural topology (microservices vs. modular monolith), deployment/infrastructure configs, language manifests, and git authorship share—scrubbing IPs/tokens via `sanitizer.py`, caching digests in `.cmr/cache/`, and persisting `.cmr/scan_report.json`.
3. **Scan-Bootstrapped Career Context (`cmr init-context --from-scan` + `context_merger`):** Uses `.cmr/scan_report.json` to pre-populate `.cmr/career_context.yaml` with discovered folder mappings, git date bounds, and detected company/client hints. The user then confirms/edits dates, adds employer exclusions, defines `project_groups`, adds non-code business impact, and refines education metadata before merging.
4. **Semantic Synthesizer & Validator (`cmr.engine`):** Generates a unified **`ResumeAST`** (`.cmr/resume_ast.json`) using Google's XYZ formula while enforcing strict topology, legacy-claim, chronology, secret-leak, and anti-hallucination validation.
5. **Stateless Multi-Format Exporters (`cmr.exporters`):** Renders synchronized outputs—ATS Markdown, dynamic ReportLab PDF (1–2 page budget), Print-Ready HTML, GitHub Profile `README.md`, and LinkedIn Profile Pack (`LinkedIn_Profile_Pack.md`)—from the single `ResumeAST` file so formats never drift out of sync.

---

## 2. Package & Directory Layout (`src/cmr/` Layout)

CMR uses a standard Python `src`-layout matching the `cmr.cli:app` entrypoint declared in `pyproject.toml`:

```text
CodeMyResume/
├── README.md                          # Project overview, pipeline diagram, and quick start
├── CHANGELOG.md                       # Chronological release and specification history
├── CONTRIBUTING.md                    # Developer setup, offline testing, and PR conventions
├── LICENSE                            # MIT License
├── pyproject.toml                     # Package metadata, setuptools src-layout, and CLI scripts
├── requirements.txt                   # Pinned runtime dependencies
├── config.yaml                        # (Planned) Default ignore lists, token budgets, and LLM settings
├── career_context.example.yaml        # (Planned) Template for anchors, exclusions, and non-code impact
├── docs/                              # Architectural & parser specifications
│   ├── PRD.md                         # Product requirements, CLI contract, exit codes, and YAML examples
│   ├── ARCHITECTURE.md                # System design, directory layout, and Pydantic schemas
│   ├── PARSERS_SPEC.md                # Manifest, topology, infra, doc chunking, sanitizer, git specs
│   ├── PROMPT_ENGINEERING.md          # XYZ-formula prompts, anti-recency rules, and validation checks
│   ├── SECURITY.md                    # Threat model, zero-read secrets, and sanitizer redaction policy
│   ├── TESTING_AND_QA.md              # Offline test pyramid, pdfplumber bounds, and LinkedIn char gates
│   └── ROADMAP.md                     # Phased engineering milestones
├── src/
│   └── cmr/
│       ├── __init__.py
│       ├── cli.py                     # Typer CLI entrypoint (scan, init-context, generate, diff)
│       ├── models.py                  # Pydantic v2 schemas: ProjectProfile, CareerContext, ResumeAST
│       ├── scanner/
│       │   ├── __init__.py
│       │   ├── walker.py              # Pruned os.walk traversal
│       │   └── detector.py            # Repo & polyglot workspace boundary detector
│       ├── parsers/
│       │   ├── __init__.py
│       │   ├── manifest_parser.py     # package.json, pyproject.toml (tomllib/tomli), go.mod, Cargo.toml
│       │   ├── topology_detector.py   # Distributed microservices vs modular monolith classifier
│       │   ├── infra_detector.py      # Docker, Cloud Run, EC2/Nginx/PM2, Netlify, Vercel, EAS detector
│       │   ├── doc_parser.py          # Full-file *.md scanner + Map-Reduce chunker + SHA-256 cache
│       │   ├── sanitizer.py           # Pre-LLM redaction of IPv4, API tokens, JWTs, and secrets
│       │   └── git_parser.py          # Author commit share %, fork detection, and commit dates
│       ├── engine/
│       │   ├── __init__.py
│       │   ├── context_merger.py      # Joins ProjectProfiles with career_context.yaml & project_groups
│       │   ├── llm_client.py          # Unified client (google-genai, Ollama, OpenAI, Anthropic)
│       │   ├── prompt_templates.py    # Grounded system and few-shot XYZ prompts
│       │   └── validator.py           # Topology, legacy-claim, secret-leak, chronology, metric validator
│       └── exporters/
│           ├── __init__.py
│           ├── markdown_exporter.py   # Pure renderer: ResumeAST -> ATS Markdown (.md)
│           ├── html_exporter.py       # Pure renderer: ResumeAST -> Continuously flowing HTML (.html)
│           ├── pdf_exporter.py        # Pure renderer: ResumeAST -> Two-Pass ReportLab PDF (.pdf)
│           ├── github_exporter.py     # Pure renderer: ResumeAST -> GitHub Profile README.md
│           └── linkedin_exporter.py   # Pure renderer: ResumeAST -> LinkedIn Pack (strict char caps)
└── tests/
    ├── test_scanner.py
    ├── test_parsers.py
    ├── test_engine.py
    └── test_exporters.py
```

> [!IMPORTANT]
> **Packaging & Python 3.10 Requirements:** `pyproject.toml` must configure `[tool.setuptools.packages.find]` with `where = ["src"]` so `cmr.cli:app` resolves cleanly on `pip install -e .`. Dependencies must use the unified **`google-genai`** SDK (replacing deprecated `google-generativeai`), **`tomli>=2.0.0; python_version < "3.11"`** for Python 3.10 TOML parsing, and **`pdfplumber`** for automated PDF page-density verification.

---

## 3. Core Pydantic Data Schemas (`src/cmr/models.py`)

### 3.1 `ProjectProfile` (Raw Extracted Repository Truth)

```python
from typing import Dict, List, Literal, Optional
from pydantic import BaseModel, Field

TopologyType = Literal[
    "distributed_microservices",
    "modular_monolith",
    "polyglot_fullstack_workspace",
    "standalone_app",
    "cli_or_library",
    "unknown",
]

class ProjectProfile(BaseModel):
    name: str
    root_path: str
    topology: TopologyType = "unknown"
    topology_evidence: List[str] = Field(default_factory=list)  # e.g., ["9 apps/*/src/main.ts", "nx.json"]
    service_or_module_count: int = 0

    # Authorship & dates (from git_parser)
    is_fork_or_low_authorship: bool = False
    user_commit_share_pct: float = 0.0
    first_commit_date: Optional[str] = None
    last_commit_date: Optional[str] = None

    # Verified stack (from manifest_parser & infra_detector)
    languages: Dict[str, float] = Field(default_factory=dict)    # language -> LOC/file weight
    frameworks: List[str] = Field(default_factory=list)
    databases: List[str] = Field(default_factory=list)
    deployment_targets: List[str] = Field(default_factory=list)  # e.g., ["Docker", "AWS ElastiCache", "PM2/Nginx", "EAS Build"]
    infra_configs_found: List[str] = Field(default_factory=list)

    # Budgeted documentation signals (from doc_parser scanning ALL *.md files)
    md_files_discovered: List[str] = Field(default_factory=list)
    candidate_org_hints: List[str] = Field(default_factory=list)  # Hints to bootstrap career_context.yaml
    core_architecture_excerpt: Optional[str] = None
    changelog_or_audit_excerpt: Optional[str] = None
    extracted_metrics: List[str] = Field(default_factory=list)
```

### 3.2 `CareerContext` (`career_context.yaml` — Timeline & Human Grounding)

Code repositories alone cannot tell CMR which employer a project belonged to, which employers must be omitted due to BGV/confidentiality, what non-code operational leadership occurred (e.g., POS vendor integrations, staff training, marketing/Google Merchant setup), or accurate education dates. `career_context.yaml` solves this cleanly:

```python
class EmploymentAnchor(BaseModel):
    id: str
    organization: str                               # e.g., "Gig Jobs India Private Limited" or "Fractional CTO"
    role_title: str
    location: str
    start_date: str                                 # e.g., "May 2024"
    end_date: str = "Present"
    engagement_type: str = "Full-Time / Contract"
    mapped_project_dirs: List[str] = Field(default_factory=list)   # Links local folders to exact dates
    project_groups: Dict[str, List[str]] = Field(default_factory=dict)  # e.g., {"The Horizon Collections": ["HC", "PDF Consolidator"]}
    non_code_impacts: List[str] = Field(default_factory=list)      # Staff training, vendor ops, ad teams, AI tooling
    anonymize_employer: bool = False

class EducationEntry(BaseModel):
    degree: str
    institution: str
    accreditation: Optional[str] = None             # e.g., "MFHEA / ECTS, Europe"
    start_date: Optional[str] = None
    end_date_or_expected: str                       # Standard academic timeline (e.g., "Expected: 2026 / 2027")
    specialization: Optional[str] = None

class CareerContext(BaseModel):
    full_name: str
    headline: str
    location: str
    phone: str
    email: str
    linkedin_url: Optional[str] = None
    github_username: Optional[str] = None
    git_author_emails: List[str] = Field(default_factory=list)
    excluded_directories: List[str] = Field(default_factory=list)
    ai_workflow_note: Optional[str] = None
    employment_anchors: List[EmploymentAnchor] = Field(default_factory=list)
    education: List[EducationEntry] = Field(default_factory=list)
```

### 3.3 `ResumeAST` (Single Source of Truth: `resume_ast.json`)

Every exporter renders strictly from `ResumeAST` so Markdown, HTML, PDF, and GitHub `README.md` never diverge:

```python
class ExperienceBlock(BaseModel):
    organization: str
    role_title: str
    date_range: str
    location: str
    tagline: Optional[str] = None
    bullets: List[str] = Field(default_factory=list)

class FlagshipProjectCard(BaseModel):
    name: str
    tagline: str
    architecture_summary: str
    tech_stack: List[str]
    highlights: List[str]
    repo_url: Optional[str] = None

class ResumeAST(BaseModel):
    full_name: str
    headline: str
    contact_line: Dict[str, str]
    professional_summary: str
    competency_matrix: Dict[str, List[str]]         # Domain -> Verified Skills
    experiences: List[ExperienceBlock]
    flagship_projects: List[FlagshipProjectCard]
    education: List[EducationEntry]
```

---

## 4. Multi-Repo Scanner & Pruning Strategy

To traverse multi-gigabyte developer workspaces quickly without hanging on dependencies:

1. **Top-Down In-Place Pruning (`walker.py`):** Uses `os.walk(root, topdown=True)` and mutates `dirs[:]` in place to exclude dependency/build trees (`node_modules`, `.git`, `.venv`, `venv`, `.yarn`, `build`, `dist`, `.gradle`, `.next`, `vendor`, `Pods`, `.bundle`, `__pycache__`) **and** non-product scratch/archive trees (`temp`, `vault`, `baseline`, `appstore-screenshots`, `.agents`) **before** descending into them.
2. **Workspace & Polyglot Grouping (`detector.py`):**
   * Recognizes standalone repos containing `.git/`, `package.json`, `pyproject.toml`, `Cargo.toml`, or `go.mod`.
   * Recognizes **multi-repo project suites** (e.g., a parent folder `GJI/` or `RME/` containing `*-backend`, `*-portal`, `*-admin`, `*-mobile` sibling repos) and groups them into a unified `ProjectProfile` so full-stack ownership isn't fragmented into isolated backend-only entries.

---

## 5. Stateless Exporter Specifications

### 5.1 ATS Markdown Exporter (`markdown_exporter.py`)
* Emits clean GitHub Flavored Markdown with strict ATS-parseable headings (`PROFESSIONAL SUMMARY`, `TECHNICAL COMPETENCIES`, `PROFESSIONAL EXPERIENCE`, `EDUCATION`).
* Avoids multi-column tables inside the resume body for deterministic parsing by Workday, Greenhouse, and Lever.

### 5.2 Dynamic ReportLab PDF Exporter (`pdf_exporter.py`)
* Renders `ResumeAST` dynamically with **zero hardcoded resume copy and zero manual `PageBreak()` calls** (so Page 1 never truncates prematurely with dead bottom whitespace).
* **Unicode & Hyperlink Safety:** Registers a clean TrueType font family (e.g., `DejaVuSans` / `Inter`) or applies a WinAnsi glyph sanitizer for fallback Type1 rendering (preventing black `■` boxes for emojis or `₹`), renders clickable `<a href="...">` PDF annotations on email/LinkedIn/GitHub, and prevents single-word wrapping on role headers.
* **Orphan & Header Protection:** Sets `keepWithNext=True` on section headings and role taglines, and wraps `[job_header, job_tag, lead_bullet]` inside `KeepTogether(...)` blocks so headings never detach from their content across page boundaries.
* **Two-Pass Density & Balance Calibration:** Uses `pdfplumber` post-render bounds measurement (`body_bottom_y` excluding the footer canvas region) to iteratively tune body font scale (`7.6pt` $\rightarrow$ `8.8pt`), leading (`10.2pt` $\rightarrow$ `12.4pt`), and vertical spacing (`3pt` $\rightarrow$ `8pt`) until:
  * Total pages match `target_pages` (typically 2 pages on A4 `841.9pt`).
  * Page 1 achieves **$\ge 88\%$ vertical fill** (`unused_above_margin <= 50pt`).
  * Final page achieves **$\ge 65\%$ vertical fill** without spilling onto an overflow page.
* **Page Footer Canvas Callback:** Renders subtle page attribution (`Name — Resume | Page X`) via `onFirstPage` and `onLaterPages`.

### 5.3 Print-Ready HTML Exporter (`html_exporter.py`)
* Renders `ResumeAST` as a **single continuously flowing container** (no artificial `.page` wrapper `div` splits with `page-break-after: always` that create half-empty print pages).
* Uses `@page { size: A4; margin: 11mm 14mm; }`, `break-after: avoid` on `.section-title, .job-header, .job-desc-tag`, and `break-inside: avoid` on bullets and education entries.

### 5.4 GitHub Profile Exporter (`github_exporter.py`)
* Renders `ResumeAST` into an aesthetic profile `README.md`.
* Includes Shields.io badges for primary social/contact links, an ASCII/Markdown technical stack matrix table, flagship architecture cards, and education summary. Paired with `cmr diff` to preview diffs before updating a live profile repo.

### 5.5 LinkedIn Profile Pack Exporter (`linkedin_exporter.py`)
* Renders `ResumeAST` into a structured, copy-pasteable `LinkedIn_Profile_Pack.md` (with cleanup instructions and Projects/Education blocks) that strictly enforces LinkedIn UI character limits:
  * **Headline:** $\le 220$ chars
  * **About:** $\le 2,600$ chars
  * **Each Experience Description:** $\le 2,000$ chars (validated prior to export)
  * **Skills:** $\le 50$ verified production skills
