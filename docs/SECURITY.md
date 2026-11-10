# Security, Privacy & Threat Model (`SECURITY.md`)

**Code My Resume (CMR)** inspects private developer workspaces, git histories, infrastructure configs, and operational runbooks. Security and confidentiality are foundational design constraints.

---

## 1. Core Security & Privacy Principles

1. **Local-First & Zero Telemetry:** CMR does not phone home, collect telemetry, or upload repositories to third-party servers.
2. **Offline-First `cmr scan`:** Default scanning (`cmr scan`) executes 100% locally with zero outbound network requests unless `--summarize` is explicitly enabled.
3. **Strict Zero-Read File Blocklist:** Credential files, private keys, and runtime environment files are never opened or parsed:
   * `.env`, `.env.*` *(only `.env.example` / `.env.template` key names are inspected for cloud service hints)*
   * `*.pem`, `*.key`, `*.p12`, `*.keystore`, `*.jks`
   * `google-services.json`, `GoogleService-Info.plist`, `*service-account*.json`
   * `id_rsa*`, `id_ed25519*`, `.npmrc`, `.pypirc`
4. **Optional 100% Air-Gapped Mode (`Ollama`):** Users handling NDA-protected repositories can synthesize `ResumeAST` exclusively through a local Ollama daemon (`OLLAMA_BASE_URL`) with zero external API traffic.

---

## 2. Threat Boundary & Mandatory Pre-Persistence Redaction (`sanitizer.py`)

```text
[Local Repositories] ──> [doc_parser / infra_detector / topology_detector] ──> [sanitizer.py] ──> [.cmr/scan_report.json & .cmr/cache/ & Cloud LLM]
   • .env BLOCKED          • Extracts *.md, infra & topology strings           • Strips server IPs,  • Only sanitized state ever leaves
   • *.key BLOCKED                                                               JWTs, keys, URIs      parsers or hits disk cache
```

`sanitizer.py` inspects **all strings** emitted by `cmr.parsers` **before** persisting `.cmr/scan_report.json`, writing `.cmr/cache/`, or sending prompts to a cloud LLM (`google-genai`, OpenAI, Anthropic):

| Sensitive Pattern | Detection Scope & False-Positive Exclusions | Replacement Token |
| :--- | :--- | :--- |
| **Public IPv4 Addresses** | Valid `0–255` dotted quads excluding RFC 1918 (`10.x`, `172.16–31.x`, `192.168.x`), `127.0.0.1`, `0.0.0.0`, and semantic/SDK version strings (`v1.2.3.4`, `16.0.10`, `10.0.22621.1`). Includes commented CORS IPs in server bootstrap configs if ever read by `infra_detector`. | `[REDACTED_IP]` |
| **API Keys & Cloud Secrets** | Google (`AIza...`), OpenAI (`sk-...`), GitHub (`ghp_...`, `github_pat_...`), AWS (`AKIA...`), Slack (`xoxb-...`), high-entropy storefront tokens | `[REDACTED_TOKEN]` |
| **JWTs & Bearer Headers** | `eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+` and `Authorization: Bearer ...` | `[REDACTED_JWT]` |
| **Database Connection URIs** | `postgres(ql)?://[^:]+:[^@]+@`, `mongodb(\+srv)?://[^:]+:[^@]+@`, `redis(s)?://[^@]*@` | `[REDACTED_URI]` |
| **SSH / Private Host Commands** | `ssh root@...`, `scp ...@...:` | `[REDACTED_SSH]` |

---

## 3. Post-Synthesis Leak Verification (`validator.py`)

Even after `sanitizer.py` cleans inputs, `validator.py` inspects the final `ResumeAST` (`resume_ast.json`) before any exporter runs:
1. Fails (Exit Code `2` in `--strict` mode) if any `[REDACTED_*]` marker accidentally appears in generated resume copy.
2. Fails if any non-local IPv4 address, bearer token, or connection string pattern matches in `ResumeAST` fields.
3. Enforces `CareerContext.excluded_directories`, `denied_claims`, and `anonymize_employer` rules so omitted names or legacy claims never appear in outputs.

---

## 4. Public Docs & `.gitignore` Hygiene

* Intermediate workspace artifacts (`.cmr/`), `.env` API keys, generated PDFs/HTMLs, and personal `career_context.yaml` files are ignored in [`.gitignore`](../.gitignore) so users never accidentally commit phone numbers, API keys, or confidential employer histories to public repositories.
* Starter templates (`career_context.example.yaml`, `docs/PRD.md`, `.env.example`) must always use synthetic placeholder data rather than real credentials or personal phone numbers.

---

## 5. Vulnerability Disclosure

If you discover a security issue, secret bypass pattern, or path-traversal edge case in CMR:
* **Contact:** `sk.r.2270@gmail.com` (Sai Kumar Rayavarapu — Maintainer)
* **SLA:** Initial acknowledgment within 48 hours; patch target within 7 business days.
* Please do **not** open public GitHub issues for unpatched secret-leak vulnerabilities.
