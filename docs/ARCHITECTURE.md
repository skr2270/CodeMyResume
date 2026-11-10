# System Architecture — Code My Resume (CMR)

This document specifies the technical design, package structure, Pydantic v2 schemas, workspace grouping rules, and rendering pipeline for **Code My Resume (CMR)**.

---

## 1. High-Level Scan-First 5-Stage Pipeline

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   1. SCANNER     │ ──> │   2. PARSERS     │ ──> │ 3. BOOTSTRAP &   │ ──> │  4. SYNTHESIZER  │ ──> │   5. EXPORTERS   │
│  Pruned Walker   │     │ Offline Topology,│     │  CONTEXT MERGE   │     │  Summarizer, LLM │     │  Pure Stateless  │
│ & Suite Detector │     │ Infra, Docs, Git │     │career_context.yml│     │  + Validator->AST│     │ PDF/MD/HTML/GH/LI│
└──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────────┘
```

1. **Scanner Stage (`cmr.scanner` — Phase 1):** Runs **first** and 100% offline via `os.walk(..., topdown=True)` with immediate directory pruning (`node_modules`, `.venv`, `.git`, `build`, `dist`, `.gradle`, `.next`, `.expo`, `.turbo`, `target`, `temp/`, `vault/`, `baseline/`, `appstore-screenshots/`, `.agents/`) to discover repositories and multi-app workspaces.
2. **Parsers & Sanitizer Stage (`cmr.parsers` — Phase 1):** Deterministically inspects manifests (including npm/pnpm `workspaces` and Gradle), structural topology (microservices vs. modular monolith), deployment/infrastructure configs, **all `*.md` files**, and aggregated multi-`.git` authorship share—scrubbing IPs/tokens via `sanitizer.py` *before* caching or persisting `.cmr/scan_report.json`.
3. **Scan-Bootstrapped Context (`cmr init-context` & `context_merger.py` — Phase 2):** Uses `.cmr/scan_report.json` to pre-populate `.cmr/career_context.yaml` + `.cmr/career_context.schema.json`. Users verify employment dates, `project_groups`, `excluded_directories`, `denied_claims`, and `non_code_impacts`.
4. **Semantic Summarizer, Synthesizer & Validator (`cmr.engine` — Phase 2):** Optionally runs Map-Reduce doc chunk summarization (`engine/summarizer.py` backed by `.cmr/cache/` SHA-256 digests), merges verified telemetry with `CareerContext`, calls `llm_client.py` (with structured-output repair retries), validates every invariant in `validator.py`, and writes **`.cmr/resume_ast.json`**.
5. **Stateless Multi-Format Exporters (`cmr.exporters` — Phase 2):** Pure renderers transforming `ResumeAST` into ATS Markdown, Two-Pass Balanced PDF, Continuous HTML, GitHub Profile `README.md`, and LinkedIn Profile Pack (`LinkedIn_Profile_Pack.md`) per [`docs/EXPORTERS_SPEC.md`](./EXPORTERS_SPEC.md).

---

## 2. Package & Directory Layout (`src/cmr/` Layout)

CMR uses a standard Python `src`-layout matching the `cmr.cli:app` entrypoint declared in [`pyproject.toml`](../pyproject.toml):

```text
CodeMyResume/
├── README.md                          # Project overview, pipeline diagram, and quick start
├── CHANGELOG.md                       # Chronological release and specification history
├── CONTRIBUTING.md                    # Developer setup, offline testing, and PR conventions
├── CODE_OF_CONDUCT.md                 # Community Contributor Covenant v2.1
├── LICENSE                            # MIT License
├── .editorconfig                      # Consistent UTF-8 & LF/CRLF formatting rules
├── .gitattributes                     # Cross-platform Git line-ending normalization
├── .env.example                       # Provider API key & Ollama base URL template
├── config.yaml                        # Default ignore lists, budgets, LLM, and PDF settings
├── career_context.example.yaml        # Anonymized starter context template
├── pyproject.toml                     # Package metadata, package-data, ruff, pytest config
├── requirements.txt                   # Runtime dependencies synced with pyproject.toml
├── .github/
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   └── workflows/
│       └── ci.yml                     # Python 3.10/3.12 CI on Ubuntu & Windows
├── docs/                              # Architectural, parser, and exporter specifications
│   ├── PRD.md                         # Product requirements, CLI contract, exit codes
│   ├── ARCHITECTURE.md                # System design, directory layout, Pydantic schemas
│   ├── PARSERS_SPEC.md                # Manifest, topology, infra, doc chunking, sanitizer, git
│   ├── EXPORTERS_SPEC.md              # ATS MD, ReportLab Two-Pass PDF, HTML, GitHub, LinkedIn
│   ├── PROMPT_ENGINEERING.md          # XYZ-formula prompts, repair loop, and validators
│   ├── SECURITY.md                    # Threat model, zero-read secrets, redaction policy
│   ├── TESTING_AND_QA.md              # Offline test pyramid, pdfplumber bounds, LinkedIn gates
│   └── ROADMAP.md                     # Phased milestones (v0.1.0 -> v1.0.0)
├── src/
│   └── cmr/
│       ├── __init__.py
│       ├── cli.py                     # Typer CLI entrypoint (scan, init-context, generate, diff)
│       ├── models.py                  # Pydantic v2 schemas: CMRConfig, ProjectProfile, CareerContext, ResumeAST
│       ├── assets/
│       │   └── fonts/                 # Bundled OFL/Apache TTF font family (Inter / DejaVuSans)
│       ├── scanner/
│       │   ├── __init__.py
│       │   ├── walker.py              # Pruned os.walk traversal
│       │   └── detector.py            # Workspace, monorepo, and multi-.git suite detector
│       ├── parsers/
│       │   ├── __init__.py
│       │   ├── manifest_parser.py     # package.json (+ workspaces), pyproject.toml, go.mod, Cargo, Gradle
│       │   ├── topology_detector.py   # Distributed microservices vs modular monolith classifier
│       │   ├── infra_detector.py      # Docker, Cloud Run, EC2/Nginx/PM2, Netlify, Vercel, EAS
│       │   ├── doc_parser.py          # Full-file *.md extractor + heading chunker (offline-pure)
│       │   ├── sanitizer.py           # Deterministic redaction of IPv4, API keys, JWTs, DB URIs
│       │   └── git_parser.py          # Multi-repo git shortlog authorship & commit date aggregator
│       ├── engine/
│       │   ├── __init__.py
│       │   ├── summarizer.py          # Optional Map-Reduce chunk summarization + .cmr/cache/
│       │   ├── context_merger.py      # Joins ProjectProfiles with career_context.yaml & project_groups
│       │   ├── llm_client.py          # Unified client (google-genai, Ollama, OpenAI, Anthropic) + repair loop
│       │   ├── prompt_templates.py    # Grounded system and few-shot XYZ prompts
│       │   └── validator.py           # Topology, allowlist/denylist, secret-leak, metric, LinkedIn validator
│       └── exporters/
│           ├── __init__.py
│           ├── markdown_exporter.py   # Pure renderer: ResumeAST -> ATS Markdown (.md)
│           ├── html_exporter.py       # Pure renderer: ResumeAST -> Continuously flowing HTML (.html)
│           ├── pdf_exporter.py        # Pure renderer: ResumeAST -> Two-Pass ReportLab PDF (.pdf)
│           ├── github_exporter.py     # Pure renderer: ResumeAST -> GitHub Profile README.md + diff
│           └── linkedin_exporter.py   # Pure renderer: ResumeAST -> LinkedIn Pack (strict char caps)
└── tests/
    ├── conftest.py                    # Offline tmp_path workspace fixtures & mocked LLMClient
    ├── test_scanner.py
    ├── test_parsers.py
    ├── test_engine.py
    └── test_exporters.py
```

---

## 3. Core Pydantic v2 Data Schemas (`src/cmr/models.py`)

### 3.0 `CMRConfig` (`config.yaml` — Runtime Configuration & Defaults)

Validated configuration schema supporting precedence (`CLI flags > workspace config.yaml > packaged defaults`):

```python
class ScannerConfig(BaseModel):
    ignore_dirs: List[str] = Field(default_factory=list)
    exclude_doc_files: List[str] = Field(default_factory=list)
    max_file_size_bytes: int = 262144

class DocBudgetConfig(BaseModel):
    full_file_max_chars: int = 8000
    map_chunk_chars: int = 10000
    chunk_token_estimate_chars: float = 3.5
    tier1_digest_budget: int = 16000
    tier2_digest_budget: int = 3000
    tier2_keywords: List[str] = Field(default_factory=list)

class GitConfig(BaseModel):
    min_user_commit_share_pct: float = 5.0

class LLMConfig(BaseModel):
    provider: Literal["gemini", "ollama", "openai", "anthropic"] = "gemini"
    synthesis_model: str = "gemini-2.5-flash"
    map_model: str = "gemini-2.5-flash-lite"
    temperature: float = 0.1
    ollama_base_url: str = "http://localhost:11434"
    max_retries: int = 2

class PDFConfig(BaseModel):
    page_size: Literal["A4", "LETTER"] = "A4"
    target_pages: int = 2
    min_page1_fill_ratio: float = 0.88
    min_last_page_fill_ratio: float = 0.65
    font_family: str = "Inter"

class CMRConfig(BaseModel):
    scanner: ScannerConfig = Field(default_factory=ScannerConfig)
    doc_budget: DocBudgetConfig = Field(default_factory=DocBudgetConfig)
    git: GitConfig = Field(default_factory=GitConfig)
    llm: LLMConfig = Field(default_factory=LLMConfig)
    pdf: PDFConfig = Field(default_factory=PDFConfig)
    taxonomy_overrides: Dict[str, List[str]] = Field(default_factory=dict)
```

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

class DocChunkDigest(BaseModel):
    file_path: str
    tier: Literal["1A", "1B", "2"]
    sha256: str
    heading: str
    raw_excerpt: str
    extracted_metrics: List[str] = Field(default_factory=list)
    llm_summary: Optional[str] = None

class ProjectProfile(BaseModel):
    schema_version: str = "1.0"
    name: str
    root_path: str
    topology: TopologyType = "unknown"
    topology_evidence: List[str] = Field(default_factory=list)
    service_or_module_count: int = 0

    # Multi-repo & manifest discovery
    workspace_repos_scanned: List[str] = Field(default_factory=list)
    manifest_paths: List[str] = Field(default_factory=list)

    # Aggregated authorship & dates (from git_parser across all child .git roots)
    is_fork_or_low_authorship: bool = False
    user_commit_share_pct: float = 0.0
    first_commit_date: Optional[str] = None
    last_commit_date: Optional[str] = None

    # Verified stack (from manifest_parser & infra_detector)
    languages: Dict[str, float] = Field(default_factory=dict)
    frameworks: List[str] = Field(default_factory=list)
    databases: List[str] = Field(default_factory=list)
    testing_tools: List[str] = Field(default_factory=list)
    deployment_targets: List[str] = Field(default_factory=list)
    infra_configs_found: List[str] = Field(default_factory=list)

    # Budgeted & sanitized documentation signals (from doc_parser)
    md_files_discovered: List[str] = Field(default_factory=list)
    candidate_org_hints: List[str] = Field(default_factory=list)
    doc_chunks: List[DocChunkDigest] = Field(default_factory=list)
    core_architecture_excerpt: Optional[str] = None
    changelog_or_audit_excerpt: Optional[str] = None
    extracted_metrics: List[str] = Field(default_factory=list)
```

### 3.2 `CareerContext` (`career_context.yaml` — Timeline & Human Grounding)

```python
class EmploymentAnchor(BaseModel):
    id: str
    organization: str
    role_title: str
    location: str
    start_date: str
    end_date: str = "Present"
    engagement_type: str = "Full-Time / Contract"
    mapped_project_dirs: List[str] = Field(default_factory=list)
    project_groups: Dict[str, List[str]] = Field(default_factory=dict)
    non_code_impacts: List[str] = Field(default_factory=list)
    anonymize_employer: bool = False

class EducationEntry(BaseModel):
    degree: str
    institution: str
    accreditation: Optional[str] = None
    start_date: Optional[str] = None
    end_date_or_expected: str
    specialization: Optional[str] = None
    honors_or_gpa: Optional[str] = None

class CareerContext(BaseModel):
    full_name: str
    headline: str
    location: str
    phone: Optional[str] = None
    email: str
    linkedin_url: Optional[str] = None
    github_username: Optional[str] = None
    git_author_emails: List[str] = Field(default_factory=list)
    excluded_directories: List[str] = Field(default_factory=list)
    denied_claims: List[str] = Field(default_factory=list)
    ai_workflow_note: Optional[str] = None
    max_flagship_projects: int = 6
    employment_anchors: List[EmploymentAnchor] = Field(default_factory=list)
    education: List[EducationEntry] = Field(default_factory=list)
```

### 3.3 `ResumeAST` (Single Source of Truth: `.cmr/resume_ast.json`)

Every exporter reads strictly from `ResumeAST` (including optional `linkedin` & bullet `source_projects` provenance) so Markdown, HTML, PDF, GitHub, and LinkedIn outputs stay 100% pure and synchronized:

```python
class ExperienceBullet(BaseModel):
    lead_label: str                                 # e.g., "Workforce Governance (Rate My Employee - RME)"
    text: str                                       # XYZ-formula accomplishment text
    sub_bullets: List[str] = Field(default_factory=list)
    source_projects: List[str] = Field(default_factory=list)  # Enables bullet-level topology checks

class ExperienceBlock(BaseModel):
    anchor_id: str
    organization: str
    role_title: str
    date_range: str
    location: str
    tagline: Optional[str] = None
    bullets: List[ExperienceBullet] = Field(default_factory=list)

class FlagshipProjectCard(BaseModel):
    name: str
    group_name: Optional[str] = None
    tagline: str
    architecture_summary: str
    tech_stack: List[str]
    highlights: List[str]
    repo_url: Optional[str] = None

class LinkedInExperienceItem(BaseModel):
    title: str
    employment_type: str
    company_name: str
    location: str
    date_range: str
    description: str                                # Strictly <= 2000 chars

class LinkedInSection(BaseModel):
    headline: str                                   # Strictly <= 220 chars
    about: str                                      # Strictly <= 2600 chars
    experiences: List[LinkedInExperienceItem] = Field(default_factory=list)
    skills: List[str] = Field(default_factory=list) # Strictly <= 50 items
    cleanup_notes: List[str] = Field(default_factory=list)

class ContactInfo(BaseModel):
    email: str
    phone: Optional[str] = None
    linkedin_url: Optional[str] = None
    github_url: Optional[str] = None
    location: str

class CompetencyCategory(BaseModel):
    category: str                                   # e.g., "Backend & Distributed"
    skills: List[str] = Field(default_factory=list)

class ResumeAST(BaseModel):
    schema_version: str = "1.0"
    generated_at: str
    target_persona: Optional[str] = None
    full_name: str
    headline: str
    contact_line: ContactInfo
    professional_summary: str
    ai_workflow_note: Optional[str] = None
    competency_matrix: List[CompetencyCategory]     # Typed ordered list (Gemini response_schema safe)
    experiences: List[ExperienceBlock]
    flagship_projects: List[FlagshipProjectCard]
    education: List[EducationEntry]
    certifications: List[str] = Field(default_factory=list)
    linkedin: Optional[LinkedInSection] = None
```

---

## 4. Multi-Repo Scanner & Workspace Grouping Strategy (`cmr.scanner`)

To handle every real-world developer layout without fragmenting full-stack products:

1. **Top-Down In-Place Pruning (`walker.py`):** Mutates `dirs[:]` in `os.walk(root, topdown=True)` before descending into dependency/build caches (`node_modules`, `.git`, `.venv`, `venv`, `env`, `.yarn`, `build`, `dist`, `.gradle`, `.next`, `.nuxt`, `.expo`, `.turbo`, `out`, `coverage`, `target`, `vendor`, `Pods`, `.bundle`, `.dart_tool`, `__pycache__`) and scratch folders (`temp`, `vault`, `baseline`, `appstore-screenshots`, `.agents`).
2. **4-Pattern Project & Workspace Grouping (`detector.py`):**
   * **Pattern A — Standalone Repo:** Single `.git` + root manifest (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `composer.json`, `pubspec.yaml`, `pom.xml`).
   * **Pattern B — Workspace Monorepo (`UWA`, `HC`):** Single root `.git` and/or root `package.json` with `workspaces` (`apps/*`, `packages/*`), `nx.json`, `turbo.json`, `pnpm-workspace.yaml`, or unprefixed multi-tier child directories (`backend/`, `mobile/`, `web/`, `portal/`, `admin/`). Child manifests automatically aggregate into the parent `ProjectProfile`.
   * **Pattern C — Nested-`.git` Product Suite (`GJI`, `RME`):** Umbrella folder lacking a root `.git` that holds `*.code-workspace` and/or sibling `.git` subdirectories matching role patterns (`^(.*[-_])?(backend|api|server|portal|web|frontend|admin|mobile|react-native|app|ios|android)$`). Grouped into one `ProjectProfile` while tracking all nested repo paths in `workspace_repos_scanned`.
   * **Pattern D — Multi-Platform Sibling Umbrella (`GestureGame`):** Umbrella directory lacking root `.git`, `package.json`, or `.code-workspace`, yet immediately containing `2+` distinct platform sub-repos (e.g., `FluxRush/` web + `FluxRushMobile/` native). Collapsed cleanly into one umbrella `ProjectProfile`.

---

## 5. Stateless Exporters Overview (`cmr.exporters`)

All rendering rules, typography calibration algorithms, `pdfplumber` coordinate math, CSS print constraints, GitHub badge templates, `cmr diff`, and LinkedIn character caps are specified in detail in **[`docs/EXPORTERS_SPEC.md`](./EXPORTERS_SPEC.md)**.
