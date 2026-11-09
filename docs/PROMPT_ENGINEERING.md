# Prompt Engineering & Ground-Truth Enforcement

This document outlines the system prompts, few-shot examples, and anti-hallucination guardrails used by **Code My Resume (CMR)** to synthesize engineering accomplishments.

---

## 1. The Core Philosophy: Ground-Truth Only

Many AI resume tools fabricate achievements, imaginary percentages, or non-existent tools. CMR enforces **strict ground-truth grounding**:
1. Every tool, framework, and language listed must exist in the parsed manifest or documentation.
2. Every system capability (e.g., "automated DigiLocker KYC" or "ReportLab GST Rule 55 generation") must have direct textual evidence from the repository.
3. If an exact metric (e.g., "sub-50ms latency", "60-120 FPS", "9 microservices", "49 screens") is specified in the docs, it is emphasized. If no metric exists, the model focuses on **scale, architectural patterns, and business domain impact** without inventing arbitrary fake numbers.

---

## 2. Google XYZ-Formula Prompt

CMR instructs the LLM using Google's renowned formula:

$$\text{Accomplished [X]} \text{ as measured by [Y]} \text{ by doing [Z]}$$

### System Prompt Template

```text
You are an elite Principal Systems Architect and Executive Technical Recruiter.
Your objective is to read ground-truth technical documentation, architecture specs, 
and dependency manifests from a software project, and transform them into 2 to 4 
high-impact, executive-level engineering resume bullet points.

RULES:
1. ALWAYS start bullet points with strong, varied action verbs:
   (Architected, Engineered, Spearheaded, Implemented, Designed, Streamlined, Mitigated).
2. Follow Google's XYZ formula:
   "Accomplished [X] as measured by [Y] by doing [Z]".
3. STRICT GROUND-TRUTH: Only mention technologies, libraries, and capabilities that 
   actually appear in the provided Project Context. Do NOT invent libraries or metrics.
4. Focus on architectural complexity, distributed systems resilience, security postures, 
   concurrency, and business domain value.
5. Format output with bold keyphrases for ATS readability:
   "• **[Domain/Impact Area]:** Detailed technical accomplishment description..."
```

---

## 3. Few-Shot Demonstration Examples

### Input Project Context (Example: FluxRush):
```yaml
Project: FluxRushMobile
Tech Stack: React Native, Shopify Skia, React Native Vision Camera, MediaPipe
Key Docs Info:
  - 60-120 FPS on native GPU context
  - Decoupled Web Worker hand tracking loop (20-30 FPS) from canvas renderer
  - <50ms input latency
  - Kalman filter, 1Euro filter, dead-reckoning velocity predictor
  - Zero-GC ring buffer object pool
```

### Generated Output:
```markdown
• **Computer Vision & Gesture Engine:** Engineered a real-time finger-tracking arcade engine decoupling MediaPipe hand landmark inference in Web Workers (20–30 FPS) from GPU-backed canvas rendering (60–120 FPS, <50ms latency) using 1Euro velocity smoothing and O(1) spatial hash grid collision partitioning.
• **High-Performance Graphic Architecture:** Developed cross-platform mobile rendering using Shopify React Native Skia and React Native Vision Camera, implementing zero-GC ring buffer object pooling to eliminate memory stutter on native GPU threads.
```

---

## 4. Anti-Hallucination Post-Validation

Before emitting the resume points, CMR runs a validation filter:
1. **Keyword Overlap Check:** Verifies that all capitalized technical terms in the generated bullets exist in the project's dependency manifest or documentation token list.
2. **Flagged Metric Check:** Any percentage over 80% or metric that does not match a regex pattern in the source docs triggers a review flag.
