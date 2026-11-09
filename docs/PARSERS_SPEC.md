# Manifest & Documentation Parsers Specification

This document defines how **Code My Resume (CMR)** extracts signals, dependency taxonomies, and technical depth from project repositories.

---

## 1. Supported Manifests

| Ecosystem | Manifest File | Primary Keys Extracted |
| :--- | :--- | :--- |
| **Node.js / TypeScript** | `package.json` | `name`, `dependencies`, `devDependencies`, `workspaces`, `scripts` |
| **Python** | `pyproject.toml`, `requirements.txt` | `[project.dependencies]`, `[tool.poetry.dependencies]`, packages |
| **Go** | `go.mod` | `module`, `require (...)` directives |
| **Rust** | `Cargo.toml` | `[dependencies]`, `[workspace.members]` |
| **PHP** | `composer.json` | `require`, `require-dev` |
| **Containers & Cloud** | `docker-compose.yml`, `Dockerfile` | Base images, linked services (e.g., `redis`, `postgres`, `rabbitmq`) |

---

## 2. Dependency Taxonomy Mapping

Raw package names are mapped to high-level engineering skills using a built-in taxonomy rule set:

```python
TAXONOMY_RULES = {
    # Backend & Frameworks
    "@nestjs/core": ("NestJS", "Backend Architecture"),
    "fastify": ("Fastify", "Backend Architecture"),
    "express": ("Express.js", "Backend Architecture"),
    "@apollo/server": ("GraphQL (Apollo)", "API Architecture"),
    "bullmq": ("BullMQ (Distributed Queues)", "Distributed Systems"),
    
    # Frontend & 3D Web
    "next": ("Next.js", "Frontend & Full-Stack"),
    "react": ("React.js", "Frontend & Full-Stack"),
    "three": ("Three.js (WebGL)", "3D Graphics"),
    "@react-three/fiber": ("React Three Fiber", "3D Graphics"),
    "tailwindcss": ("Tailwind CSS", "Styling & Design"),
    "gsap": ("GSAP", "Animation Engineering"),
    
    # Mobile
    "react-native": ("React Native", "Mobile Engineering"),
    "@shopify/react-native-skia": ("Shopify React Native Skia", "High-Performance Graphics"),
    "react-native-vision-camera": ("React Native Vision Camera", "Computer Vision & Mobile"),
    "expo": ("Expo SDK", "Mobile Engineering"),
    
    # Databases & Caching
    "pg": ("PostgreSQL", "Databases"),
    "typeorm": ("TypeORM", "Data Modeling"),
    "ioredis": ("Redis", "Caching & Distributed State"),
    "mongoose": ("MongoDB", "NoSQL Databases"),
    
    # Python & Machine Learning
    "reportlab": ("ReportLab", "Document Intelligence & PDF"),
    "mediapipe": ("MediaPipe", "Computer Vision"),
    "torch": ("PyTorch", "Machine Learning & AI"),
    "fastapi": ("FastAPI", "Backend Architecture"),
    "pydantic": ("Pydantic", "Data Validation")
}
```

---

## 3. Documentation Ingestion Strategy

### 3.1 Target Files
The documentation parser searches for and reads:
1. `README.md` / `README.markdown` / `readme.txt`
2. `docs/ARCHITECTURE.md` / `docs/architecture.md`
3. `docs/PRD.md` / `docs/prd.md`
4. `docs/SYSTEM_DESIGN.md`
5. `CHANGELOG.md`
6. `openapi.json` / `swagger.json` / `docs/api-*.md`

### 3.2 Content Sanitization
To avoid bloating LLM context windows, document content is sanitized:
* Code blocks larger than 50 lines are truncated to signatures/interfaces.
* Long ASCII diagrams are summarized.
* Base64 embedded images are stripped out.

---

## 4. Git History & Telemetry Extraction

Using local `git` commands (or `pygit2` / `GitPython`), CMR extracts:
* **Active Tenure:** Date of first commit and date of most recent commit (e.g., `May 2024 – Present`).
* **Velocity Metrics:** Total commit count, number of releases/tags.
* **Top Contributors:** Validates that the active user is the primary author or a key contributor to the repository.

