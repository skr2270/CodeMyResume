# System Architecture — Code My Resume (CMR)

This document specifies the technical design, data structures, and pipeline flow for **Code My Resume (CMR)**.

---

## 1. High-Level Pipeline

The CMR processing pipeline consists of four sequential stages:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   1. SCANNER     │ ──> │   2. EXTRACTOR   │ ──> │  3. SYNTHESIZER  │ ──> │   4. EXPORTER    │
│  Directory Walk  │     │  Manifests & Docs│     │  LLM Formatting  │     │ PDF, MD, GitHub  │
└──────────────────┘     └──────────────────┘     └──────────────────┘     └──────────────────┘
```

1. **Scanner Stage:** Discovers repository root boundaries across local directories while enforcing strict ignore rules (`node_modules`, `.git`, `build`, etc.).
2. **Extractor Stage:** Parses language manifests (`package.json`, `pyproject.toml`, `Cargo.toml`), reads markdown documentation (`README.md`, `docs/*.md`, PRDs), and inspects git commit telemetry.
3. **Synthesizer Stage:** Feeds structured repository profiles into an LLM engine instructed with Google's XYZ formula, producing high-impact, verifiable engineering accomplishments.
4. **Exporter Stage:** Compiles structured output into multiple targets: ATS Markdown, ReportLab publication PDF (2-page budget), and GitHub Profile `README.md`.

---

## 2. Directory Structure of CMR

```
CodeMyResume/
├── README.md                          # Project overview and quick start
├── pyproject.toml                     # Python package configuration
├── requirements.txt                   # Production dependencies
├── config.yaml                        # Default scanner and LLM settings
├── docs/                              # Architectural documentation
│   ├── ARCHITECTURE.md                # System design & pipeline specification
│   ├── PARSERS_SPEC.md                # Language & manifest parser details
│   ├── PROMPT_ENGINEERING.md          # LLM prompt schemas and XYZ guidelines
│   └── ROADMAP.md                     # Engineering roadmap & releases
├── src/
│   ├── __init__.py
│   ├── cli.py                         # Command-line interface (Typer/Click)
│   ├── scanner/
│   │   ├── __init__.py
│   │   ├── walker.py                  # Recursive directory walker with pruning
│   │   └── detector.py                # Repo boundary detector
│   ├── parsers/
│   │   ├── __init__.py
│   │   ├── manifest_parser.py         # package.json, pyproject.toml, etc.
│   │   ├── doc_parser.py              # README, PRD, and markdown extraction
│   │   └── git_parser.py              # Commit date ranges and active durations
│   ├── engine/
│   │   ├── __init__.py
│   │   ├── llm_client.py              # Unified LLM provider (Gemini, Ollama, OpenAI)
│   │   ├── prompt_templates.py        # System and few-shot prompt definitions
│   │   └── validator.py               # Anti-hallucination and grounding check
│   └── exporters/
│       ├── __init__.py
│       ├── markdown_exporter.py       # ATS-friendly markdown resume
│       ├── pdf_exporter.py            # ReportLab publication-ready PDF engine
│       └── github_exporter.py         # GitHub Profile README.md generator
└── tests/
    ├── test_scanner.py
    ├── test_parsers.py
    └── test_exporters.py
```

---

## 3. Core Data Schema: `ProjectProfile`

Each scanned project is converted into an intermediate typed representation (`ProjectProfile`):

```python
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from datetime import datetime

@dataclass
class ProjectProfile:
    name: str                                    # Project identifier / directory name
    root_path: str                               # Absolute filesystem path
    active_date_range: Optional[str] = None     # e.g., "May 2024 – Present" (from Git/Docs)
    primary_language: str = "Unknown"
    languages: List[str] = field(default_factory=list)
    frameworks: List[str] = field(default_factory=list)
    databases: List[str] = field(default_factory=list)
    cloud_and_infra: List[str] = field(default_factory=list)
    
    # Document content
    readme_summary: Optional[str] = None
    architecture_overview: Optional[str] = None
    hard_problems_solved: List[str] = field(default_factory=list)
    
    # Generated outputs
    resume_bullets: List[str] = field(default_factory=list)
    github_card_summary: Optional[str] = None
```

---

## 4. Multi-Repo Scanner Strategy

To achieve high scanning performance across multi-gigabyte project directories:

1. **Top-Down Pruning:** Folders named `node_modules`, `.git`, `.venv`, `.yarn`, `build`, `dist`, `.gradle`, and `vendor` are skipped **before** descending into their child trees.
2. **Boundary Heuristics:** A directory is classified as an independent repository if it contains:
   * A `.git/` folder, OR
   * A package manifest (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`), OR
   * A root `README.md` paired with a source folder (`src/`, `lib/`, `apps/`).
3. **Monorepo Awareness:** Workspaces containing `pnpm-workspace.yaml`, `nx.json`, `turbo.json`, or root `apps/` folders are decomposed into their constituent sub-applications while retaining shared infrastructure context.

---

## 5. Exporter Specifications

### 5.1 ATS Markdown Exporter
* Standard ASCII and GitHub Flavored Markdown.
* Strict section headings: `PROFESSIONAL SUMMARY`, `TECHNICAL COMPETENCIES`, `PROFESSIONAL EXPERIENCE`, `FLAGSHIP ENGINEERING PROJECTS`, `EDUCATION`.
* Zero unparseable tables or multi-column layouts for maximum ATS compatibility.

### 5.2 PDF Exporter (ReportLab Engine)
* Enforces exact **2-Page Page Budget** using ReportLab Flowables.
* Header with primary navy banner (`#1A365D`) and subtitle accent (`#2B6CB0`).
* Compact table-based technical skills section for optimal vertical density.
* Explicit `PageBreak()` management to prevent orphan job bullets.

### 5.3 GitHub Profile Exporter
* Generates an aesthetic, developer-first `README.md`.
* Shields.io badge badges for languages and social coordinates.
* Categorized skill matrix table with clear domain boundaries.
* Project showcase with architectural bullet points and deep links.
