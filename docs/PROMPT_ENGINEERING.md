# Prompt Engineering & Ground-Truth Enforcement

This document defines the synthesis prompts, Map-Reduce doc summarization contract, structured-output repair retries, and post-generation validators inside `src/cmr/engine/`.

---

## 1. Core Grounding Principles

Many AI resume tools fabricate metrics, extrapolate frameworks, or misrepresent architectures. CMR enforces **strict multi-layer grounding**:

1. **Structural Topology Fidelity:** Every `ExperienceBullet` records `source_projects`. If a source `ProjectProfile.topology == "modular_monolith"`, the LLM is **forbidden** from calling internal modules `"microservices"`; it must state *"domain-driven modular architecture with N business modules"*.
2. **Anti-Recency & Full-Scope Weighting:** Bullets backed by Tier 1A/1B docs and polyglot infra configs must capture end-to-end ownership (UI, mobile, API, database, security, deployment), preventing Tier 2 changelogs/audits from collapsing a multi-year Lead role into recent audits alone.
3. **Strict Chronology & Employer Boundaries:** Projects mapped to `EmploymentAnchor A` (`May 2024 – Present`) never bleed into `EmploymentAnchor B` (`March 2021 – May 2024`). `excluded_directories` and `anonymize_employer` rules remove omitted names cleanly.
4. **Non-Code Impact & FinOps Extraction:** Fractional CTO duties, POS/payment/logistics integrations, store staff training, ad-team alignment, AI-augmented engineering notes, and verified AWS/cloud FinOps optimizations are surfaced alongside code metrics.
5. **Academic Formatting Rule:** Ongoing degrees render with `Expected` completion dates (`Expected: 2026 / 2027`); completed degrees emit concise graduation dates (`Graduated: April 2022`).
6. **Allowlist + Legacy Denylist Guard:** Never carry forward legacy items declared in `CareerContext.denied_claims` (e.g., `Keycloak`, `Kubernetes`) unless actively verified on disk, and never extrapolate frameworks (e.g., `Next.js` or `WebGL` on a `Vite + Vanilla JS + HTML5 Canvas 2D` app).

---

## 2. Google XYZ-Formula Synthesis Prompt (`prompt_templates.py`)

CMR formats engineering accomplishments using Google's XYZ formula:

$$\text{Accomplished [X]} \text{ as measured by [Y]} \text{ by doing [Z]}$$

### System Prompt Template

```text
You are an elite Principal Systems Architect and Executive Technical Recruiter.
Synthesize the sanitized ProjectProfiles (structural topology, manifests, infra configs,
budgeted doc summaries) and CareerContext (employment anchors, project_groups, denied_claims,
non_code_impacts, ai_workflow_note) into a single valid ResumeAST JSON payload.

MANDATORY RULES:
1. STRONG ACTION VERBS: Begin bullets with executive verbs
   (Architected, Spearheaded, Engineered, Orchestrated, Deployed, Hardened, Streamlined).
2. GOOGLE XYZ FORMULA: Phrase accomplishments as
   "Accomplished [X] as measured by [Y] by doing [Z]" with a bold lead_label.
3. TOPOLOGY ACCURACY & PROVENANCE: Populate `source_projects` on every ExperienceBullet.
   Never label a `modular_monolith` project as "microservices".
4. FULL-SCOPE BALANCE: Weight Tier 1A/1B architecture & deployment ownership above Tier 2 audits.
5. STRICT STACK GROUNDING: Only emit languages/tools verified in non-forked ProjectProfiles
   (`is_fork_or_low_authorship == false`) or `CareerContext.non_code_impacts`.
   Never emit items listed in `CareerContext.denied_claims`.
6. CHRONOLOGICAL ISOLATION: Bind projects strictly to their assigned `EmploymentAnchor`.
7. LINKEDIN SECTION SYNTHESIS: Populate `ResumeAST.linkedin` cleanly within hard limits:
   headline <= 220 chars, about <= 2600 chars, each experience description <= 2000 chars,
   and <= 50 verified skills.
8. ZERO LEAKED SECRETS: Never emit IPv4 literals, tokens, connection URIs, or `[REDACTED_*]` tags.
```

---

## 3. Unified LLM Client & Repair Loop (`llm_client.py`)

`llm_client.py` provides a clean provider abstraction with Pydantic validation guarantees:

* **Gemini (`google-genai`, Default Cloud):** Uses `from google import genai` with `response_schema=ResumeAST` and `temperature=0.1`. Exponential backoff handles HTTP 429 rate limits cleanly.
* **Ollama (100% Air-Gapped Local Inference):** Connects to `OLLAMA_BASE_URL` over `httpx` with strict JSON formatting prompts.
* **OpenAI / Anthropic (Optional `[llms]` Extras):** Selected via `CMR_LLM_PROVIDER`.
* **Markdown Fence Strip & Validation-Repair Loop:** Open-weight/Ollama models frequently wrap JSON in ```` ```json ```` fences or miss nested schema keys. `llm_client.py` strips code fences, parses with `ResumeAST.model_validate_json()`, and runs up to **`config.llm.max_retries` (`2`)** automatic error-feedback repair turns if Pydantic validation fails.

---

## 4. Few-Shot Grounded Demonstration (Synthetic Placeholder Context)

### Input Context Snippet:
```yaml
EmploymentAnchor:
  id: advisory_ventures
  organization: "Independent Ventures & Client Engagements"
  role_title: "Fractional CTO & Solutions Consultant"
  date_range: "May 2024 – Present"
  mapped_project_dirs: ["commerce-app", "pdf-logistics", "workforce-suite"]
  project_groups:
    "Apex Retail Group": ["commerce-app", "pdf-logistics"]
  non_code_impacts:
    - "Configured Google Merchant profiles across 4 retail stores; trained staff on digital ops"

ProjectProfile (workforce-suite):
  topology: "modular_monolith"
  topology_evidence: ["Single src/main.ts", "src/app.module.ts importing 21 domain modules"]
  service_or_module_count: 21
  deployment_targets: ["Ubuntu 24.04 VPS", "PM2", "Nginx TLS 1.3"]
  extracted_metrics: ["27 schema migrations", "49 mobile screens", "16KB page alignment"]
```

### Validated `ExperienceBullet` Output:
```json
{
  "lead_label": "Digital Transformation & Headless Commerce (Apex Retail Group)",
  "text": "Spearheaded multi-store scaling across 4 retail locations—engineering a headless React Native app (Expo SDK, GraphQL, Fastify BFF), integrating Zoho POS/Razorpay/Shiprocket, configuring Google Merchant profiles, and training store teams.",
  "sub_bullets": [],
  "source_projects": ["commerce-app", "pdf-logistics"]
},
{
  "lead_label": "Workforce Governance Ecosystem (Workforce Suite)",
  "text": "Architected an employee verification platform powered by a domain-driven NestJS modular backend (21 business modules, Apollo GraphQL v4, PostgreSQL 16/PostGIS with 27 migrations), 49-screen React Native app (Android 15 16KB alignment), Next.js admin portal, and Ubuntu VPS (PM2/Nginx TLS 1.3) deployment.",
  "sub_bullets": [],
  "source_projects": ["workforce-suite"]
}
```

---

## 5. Post-Generation Validation (`validator.py`)

Before persisting `.cmr/resume_ast.json` or calling any exporter, `validator.py` runs deterministic checks (returning **Exit Code `2`** in `--strict` mode on failure):

1. **Bullet-Level Topology Guard:** Uses `ExperienceBullet.source_projects` to verify `"microservices"` never appears on a bullet sourced exclusively from `modular_monolith` projects.
2. **Allowlist + Explicit Denylist Guard:** Checks `ResumeAST.competency_matrix` skills against verified non-forked `ProjectProfile` dependencies/infra + `CareerContext.non_code_impacts`, fails on any `CareerContext.denied_claims` leak (e.g., `Keycloak`, `Kubernetes`), and checks framework contradiction rules (`"WebGL"`/`"Three.js"` requires `three`; `"Next.js"` requires `next`).
3. **Chronology & Exclusion Guard:** Blocks `CareerContext.excluded_directories` and anonymized names from all `ResumeAST` fields, and enforces project-to-anchor mapping.
4. **Metric Grounding Check:** Cross-checks numeric claims in bullets against `ProjectProfile.extracted_metrics` and `CareerContext` notes, flagging ungrounded statistics.
5. **Secret & IP Leak Check:** Ensures zero raw server IPv4s, API tokens, JWTs, DB URIs, or `[REDACTED_*]` placeholders survive into `ResumeAST`.
6. **LinkedIn Character-Limit Guard:** Enforces `len(headline) <= 220`, `len(about) <= 2600`, `all(len(exp.description) <= 2000)`, and `len(skills) <= 50` on `ResumeAST.linkedin`.
