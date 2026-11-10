# Prompt Engineering & Ground-Truth Enforcement

This document defines the synthesis prompts, anti-recency rules, chronology guardrails, and post-generation validators inside `src/cmr/engine/`.

---

## 1. Core Grounding Principles

Many AI resume tools fabricate achievements, hallucinate numbers, or misrepresent architectures. CMR enforces **strict multi-layer grounding**:

1. **Structural Topology Fidelity:** The LLM must respect the `topology` classification from `topology_detector.py`:
   * `distributed_microservices` $\rightarrow$ Allowed to state *"N microservices"*.
   * `modular_monolith` $\rightarrow$ **Forbidden** from calling internal modules "microservices"; must state *"domain-driven modular architecture with N business modules"*.
2. **Anti-Recency & Full-Scope Weighting:** When synthesizing an `EmploymentAnchor` backed by a polyglot workspace (e.g., web portal + admin + mobile + backend + deployment configs) alongside a recent audit report or changelog, bullets must represent **end-to-end SDLC ownership** (Architecture, Frontend/Mobile, Backend, Security, and Deployment)—never collapsing a multi-year Lead role into just recent audits.
3. **Strict Chronology & Employer Boundaries:** Projects mapped to `EmploymentAnchor A` (`May 2024 – Present`) must **never** bleed into `EmploymentAnchor B` (`March 2021 – May 2024`), and vice versa. Excluded/anonymized employers from `career_context.yaml` are never emitted.
4. **Non-Code Impact & FinOps Extraction:** Fractional CTO duties, partner integrations (e.g., POS, payment gateways, courier APIs), staff training, ad/marketing collaboration, AI-augmented engineering workflows, and verified cloud cost optimization analyses (e.g., decommissioning unused brokers to reduce spend) are elevated alongside code achievements.
5. **Academic Formatting Rule:** Ongoing degrees always render using `Expected Completion` / `Expected Graduation` (e.g., `2024 – Present (Expected: 2026 / 2027)`), never raw ID card strings such as `"Validity"`.
6. **Legacy Resume & Framework Non-Extrapolation Guard:** Never carry forward outdated stack items (e.g., `Keycloak`, `Kubernetes`) from an older resume unless verified in active `package.json` / `infra_detector` manifests, and never extrapolate frameworks (e.g., claiming `Next.js` or `WebGL` on a `Vite + Vanilla JS + HTML5 Canvas 2D` app).

---

## 2. Google XYZ-Formula Synthesis Prompt (`prompt_templates.py`)

CMR formats engineering accomplishments using Google's XYZ formula:

$$\text{Accomplished [X]} \text{ as measured by [Y]} \text{ by doing [Z]}$$

### System Prompt Template

```text
You are an elite Principal Systems Architect and Executive Technical Recruiter.
Your objective is to synthesize verified ProjectProfiles (structural topology, manifests,
infra configs, budgeted docs) and CareerContext (employment anchor dates, non-code impacts,
AI-augmented workflows) into a single valid ResumeAST JSON payload.

MANDATORY RULES:
1. STRONG ACTION VERBS: Begin bullets with varied executive verbs
   (Architected, Spearheaded, Engineered, Orchestrated, Deployed, Hardened, Streamlined).
2. GOOGLE XYZ FORMULA: Phrase accomplishments as
   "Accomplished [X] as measured by [Y] by doing [Z]" using bold lead-in labels:
   "**[Domain / Ownership Area]:** ..."
3. TOPOLOGY ACCURACY: Never label a `modular_monolith` as "microservices". Use the exact
   `topology` and `service_or_module_count` provided in ProjectProfile.
4. FULL-SCOPE BALANCE: Do not over-index on Tier 2 docs (changelogs/audits). Ensure bullets
   reflect full-stack breadth (UI, mobile, API, database, security, and production deployment)
   whenever present in the workspace.
5. STRICT LANGUAGE GROUNDING: Only include languages/tools present in non-forked
   ProjectProfiles (`is_fork_or_low_authorship == false`) or explicitly confirmed in CareerContext.
6. CHRONOLOGICAL ISOLATION: Only associate a project with the exact EmploymentAnchor
   to which it is mapped in CareerContext.
7. NON-CODE LEADERSHIP: Integrate `non_code_impacts` (vendor integrations, store ops,
   staff training, agency guidance) naturally into consulting / Fractional CTO roles.
8. CLIENT SUB-GROUPING: When `project_groups` maps sibling repos under one client
   (e.g., HC + PDF Consolidator), keep their shared business impact cleanly structured.
```

---

## 3. Unified LLM Client (`llm_client.py`)

`llm_client.py` provides a clean provider abstraction with **structured JSON output** matching the `ResumeAST` Pydantic schema:

* **Gemini (Default Cloud):** Uses Google's unified **`google-genai`** SDK (`from google import genai`) with `response_schema=ResumeAST` for deterministic structured output.
* **Ollama (100% Offline Privacy Mode):** Local inference for teams with strict proprietary code policies.
* **OpenAI / Anthropic:** Optional adapters configured via `CMR_LLM_PROVIDER` in `.env`.

---

## 4. Few-Shot Grounded Demonstration

### Input Context (`ProjectProfile` + `CareerContext` Snippet):
```yaml
EmploymentAnchor:
  organization: "Fractional CTO & Principal Solutions Consultant"
  date_range: "May 2024 – Present"
  mapped_projects: ["HC", "PDF Consolidator", "RME"]
  non_code_impacts:
    - "Configured Google Merchant profiles across 4 retail stores; trained staff on digital operations"
    - "Integrated Zoho POS, PhonePe, Razorpay, Shiprocket, Breeze (super.money), MyOperator, Bitespeed"

ProjectProfile (RME):
  topology: "modular_monolith"
  topology_evidence: ["Single src/main.ts", "src/app.module.ts importing 21 domain modules"]
  service_or_module_count: 21
  deployment_targets: ["Linux VPS", "PM2", "Nginx", "SSL/TLS"]
  extracted_metrics: ["27 schema migrations", "49 mobile screens", "16KB page alignment"]
```

### Validated Output Bullets:
```markdown
• **Digital Transformation & Headless Commerce (The Horizon Collections):** Spearheaded end-to-end technology and multi-store scaling across 4 retail locations—engineering a headless React Native mobile app (Expo SDK 58 CNG, Shopify GraphQL, Fastify BFF), orchestrating partner integrations (Zoho POS, PhonePe, Razorpay, Shiprocket, Breeze, MyOperator, Bitespeed), configuring Google Merchant profiles, and training store staff on digital operations.
• **Workforce Governance Ecosystem (Rate My Employee - RME):** Architected and deployed an employee verification platform powered by a domain-driven NestJS modular backend (21 business modules, Apollo GraphQL, PostgreSQL/PostGIS with 27 migrations), 49-screen React Native app (Android 15, 16KB page alignment), Next.js 15 admin portal, DigiLocker KYC, and Linux VPS (PM2/Nginx/SSL) production hosting.
```

---

## 5. Post-Generation Validation (`validator.py`)

Before persisting `resume_ast.json` or calling any exporter, `validator.py` runs deterministic checks:

1. **Topology Terminology Guard:** Flags/fails if `"microservices"` appears inside a bullet whose source `ProjectProfile.topology == "modular_monolith"`.
2. **Fork & Legacy-Claim Leak Guard:** Verifies that every skill, framework, or deployment orchestrator in `ResumeAST.competency_matrix` and experience bullets exists in non-forked `ProjectProfile` dependencies/infra configs (`is_fork_or_low_authorship == false`) or `CareerContext`—blocking leftover legacy terms (e.g., `Keycloak`, `Kubernetes`) and extrapolated UI/graphics claims (e.g., `Next.js` or `WebGL` when `Canvas 2D` + `Vite` is detected).
3. **Chronology & Exclusion Guard:** Ensures zero strings from `CareerContext.excluded_directories` or anonymized company names appear in `ResumeAST`, and verifies project names match their assigned `EmploymentAnchor`.
4. **Metric Grounding Check:** Cross-references numeric metrics in generated bullets (e.g., `27 migrations`, `49 screens`, `60-120 FPS`, `64.5% AWS cost reduction`) against `ProjectProfile.extracted_metrics` and `CareerContext` notes, flagging ungrounded statistics for review.
5. **Secret & IP Leak Check:** Ensures no raw IPv4 addresses, API tokens, JWTs, or `[REDACTED_*]` placeholders leak into `ResumeAST`.
6. **LinkedIn Character-Limit Enforcement:** Verifies headline ($\le 220$), summary ($\le 2,600$), experience block ($\le 2,000$), and skill count ($\le 50$) constraints prior to emitting `LinkedIn_Profile_Pack.md`.
