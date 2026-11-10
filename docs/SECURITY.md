# Security, Privacy & Threat Model (`SECURITY.md`)

**Code My Resume (CMR)** inspects private developer workspaces, git histories, infrastructure configs, and operational runbooks. Security and confidentiality are foundational design constraints.

---

## 1. Core Security & Privacy Principles

1. **Local-First & Zero Telemetry:** CMR does not phone home, collect analytics, or upload repositories to third-party servers.
2. **Strict Zero-Read File Blocklist:** Credential files, private keys, and runtime environment files are never opened or parsed:
   * `.env`, `.env.*` *(only `.env.example` / `.env.template` key names are inspected for cloud service detection)*
   * `*.pem`, `*.key`, `*.p12`, `*.keystore`, `*.jks`
   * `google-services.json`, `GoogleService-Info.plist`, `*service-account*.json`
   * `id_rsa*`, `id_ed25519*`, `.npmrc`, `.pypirc`
3. **Optional 100% Air-Gapped Mode (`Ollama` + `--offline`):** Users handling NDA-protected repositories can execute `cmr scan --offline` for deterministic parsing or route synthesis exclusively through a local Ollama daemon with zero outbound network traffic.

---

## 2. Threat Boundary & Pre-LLM Redaction (`sanitizer.py`)

```text
[Local Repositories] ──> [doc_parser / infra_detector] ──> [sanitizer.py (Redaction)] ──> [Cloud LLM / Cache]
   • .env BLOCKED          • Reads *.md & configs          • Strips IPv4s, JWTs,         • Only sanitized
   • *.key BLOCKED                                           API keys, DB URIs             digests leave disk
```

Before any digest or prompt reaches a cloud LLM provider (`google-genai`, OpenAI, Anthropic) or `.cmr/cache/`, `sanitizer.py` enforces deterministic regex redaction:

| Sensitive Pattern | Detection Scope | Replacement Token |
| :--- | :--- | :--- |
| **Public IPv4 Addresses** | Non-RFC1918 server IPs inside deployment runbooks (`103.x.x.x`, `18.x.x.x`, excluding semantic version numbers) | `[REDACTED_IP]` |
| **API Keys & Cloud Secrets** | Google (`AIza...`), OpenAI (`sk-...`), GitHub (`ghp_...`, `github_pat_...`), AWS (`AKIA...`), Slack (`xoxb-...`), high-entropy storefront tokens | `[REDACTED_TOKEN]` |
| **JWTs & Bearer Headers** | `eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+` and `Authorization: Bearer ...` | `[REDACTED_JWT]` |
| **Database Connection URIs** | `postgres(ql)?://[^:]+:[^@]+@`, `mongodb(\+srv)?://[^:]+:[^@]+@`, `redis(s)?://[^@]*@` | `[REDACTED_URI]` |
| **SSH / Private Host Commands** | `ssh root@...`, `scp ...@...:` | `[REDACTED_SSH]` |

---

## 3. Post-Synthesis Leak Verification (`validator.py`)

Even after `sanitizer.py` cleans inputs, `validator.py` inspects the final `ResumeAST` (`resume_ast.json`) before any exporter runs:
1. Fails (exit code `2`) if any `[REDACTED_*]` marker accidentally appears in generated resume copy.
2. Fails if any IPv4 address, bearer token, or connection string pattern matches in `ResumeAST` bullets.
3. Enforces `CareerContext.excluded_directories` and `anonymize_employer` rules so omitted/confidential employer names never appear in output files.

---

## 4. Local State & `.gitignore` Hygiene

* Intermediate workspace artifacts (`.cmr/scan_report.json`, `.cmr/career_context.yaml`, `.cmr/resume_ast.json`, `.cmr/cache/`) and `.env` API keys are explicitly ignored in `.gitignore` so users never accidentally commit personal contact info, API keys, or private career context to public GitHub repos.

---

## 5. Vulnerability Disclosure

If you discover a security issue, secret bypass pattern, or path-traversal edge case in CMR:
* **Contact:** `sk.r.2270@gmail.com` (Sai Kumar Rayavarapu — Maintainer)
* **SLA:** Initial acknowledgment within 48 hours; patch target within 7 business days.
* Please do **not** open public GitHub issues for unpatched secret-leak vulnerabilities.
