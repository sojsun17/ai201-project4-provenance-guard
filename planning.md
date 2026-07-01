# Provenance Guard — Planning Document

## Architecture Narrative

A piece of text enters the system via `POST /submit`. It is assigned a unique `content_id`, then passed simultaneously to two independent detection signals: (1) an LLM-based classifier via Groq that evaluates semantic and stylistic coherence, and (2) a stylometric heuristics engine that computes statistical properties of the text (sentence length variance, type-token ratio, punctuation density). Both signals return a score between 0 and 1 (1 = strongly AI). These scores are combined into a single weighted confidence score, which is then mapped to a transparency label. The full decision — signals, score, label, timestamp — is written to a structured audit log. The endpoint returns the `content_id`, attribution result, confidence score, and label text to the caller.

An appeal enters via `POST /appeal` with a `content_id` and `creator_reasoning`. The system looks up the original classification, updates the content's status to `"under_review"`, logs the appeal reason alongside the original decision, and returns a confirmation. No automated re-classification occurs; the record is flagged for human review.

## Architecture

```
Submission Flow:
POST /submit
    │
    ├─── [text] ──→ Signal 1: Groq LLM Classifier ──→ llm_score (0–1)
    │                                                         │
    ├─── [text] ──→ Signal 2: Stylometric Heuristics ──→ style_score (0–1)
    │                                                         │
    └───────────────────────────────────────────────→ Confidence Scorer
                                                             │
                                              [weighted combined score]
                                                             │
                                                   Transparency Label Generator
                                                             │
                                                   Audit Log (SQLite)
                                                             │
                                                   JSON Response
                                          {content_id, attribution, confidence, label}

Appeal Flow:
POST /appeal
    │
    ├─── [content_id] ──→ Lookup original record
    ├─── [creator_reasoning] ──→ Append to audit entry
    └─── Status update: "classified" → "under_review"
                                                   │
                                               Audit Log
                                                   │
                                          JSON Response {status: "appeal_received"}
```

---

## Detection Signals

### Signal 1: Groq LLM Classifier
- **What it measures:** Semantic and stylistic coherence — whether the text "reads" like AI output holistically. The LLM assesses rhetorical patterns, hedging language, structural uniformity, and generic phrasing that AI systems tend to produce.
- **Output:** A float between 0.0 and 1.0 (1.0 = highly likely AI-generated), parsed from a structured JSON response.
- **Blind spots:** Cannot reliably detect lightly edited AI text, heavily stylized human writing, or content from non-native English speakers whose writing may pattern-match to AI outputs due to formality and regularity.

### Signal 2: Stylometric Heuristics
- **What it measures:** Statistical properties that differ structurally between AI and human writing:
  - **Sentence length variance** — AI tends to produce uniform, moderate-length sentences; human writing varies more wildly.
  - **Type-token ratio (TTR)** — Vocabulary diversity. AI often reuses a narrower lexical range.
  - **Punctuation density** — Humans use em-dashes, ellipses, exclamation marks, and comma splices more idiosyncratically.
- **Output:** A float between 0.0 and 1.0 (1.0 = patterns consistent with AI), computed entirely in Python with no external libraries.
- **Blind spots:** Formal human writing (academic papers, legal documents) will score high on uniformity. Short texts (<50 words) produce unreliable statistics.

### Signal Combination
Weighted average, biased toward the LLM signal:
```
confidence = (0.65 × llm_score) + (0.35 × style_score)
```
Rationale: The LLM signal captures semantic intent; stylometrics can over-fire on legitimate formal writing. The asymmetric weight reflects a deliberate bias toward reducing false positives (misclassifying human work as AI).

---

## Uncertainty Representation

| Confidence Range | Attribution Label | Meaning |
|---|---|---|
| 0.00 – 0.35 | `likely_human` | Strong evidence of human authorship |
| 0.36 – 0.64 | `uncertain` | Signals disagree or are inconclusive |
| 0.65 – 1.00 | `likely_ai` | Strong evidence of AI generation |

**Design rationale:** The uncertain band is intentionally wide (29 points). This reflects the asymmetry mentioned in the spec: a false positive (labeling human work as AI) is worse than a false negative. When in doubt, we say "uncertain" and let creators appeal rather than making a confident wrong call.

**What 0.6 means:** The system leans toward AI but is not confident — it falls in the uncertain band and produces the uncertain label, not an AI label.

---

## Transparency Label Variants

### High-Confidence AI (`confidence ≥ 0.65`)
```
⚠️ AI-Generated Content Detected
Our analysis found strong signals that this content was likely generated by an AI tool.
Confidence: [X]%
This label is based on automated analysis and may not be perfect. If you are the author
and believe this is incorrect, you can submit an appeal below.
```

### Uncertain (`0.35 < confidence < 0.65`)
```
🔍 Authorship Unclear
Our system could not determine with confidence whether this content was written by a
human or generated by AI. It will be treated as unverified.
Confidence: [X]%
If you are the author, you may submit an appeal to provide context about your writing process.
```

### High-Confidence Human (`confidence ≤ 0.35`)
```
✅ Likely Human-Written
Our analysis found strong signals that this content was written by a human author.
Confidence: [X]%
Automated analysis is not perfect. This label reflects our best assessment based on
available signals.
```

---

## Appeals Workflow

- **Who can appeal:** Any creator who submitted content (identified by `creator_id` in the original submission).
- **What they provide:** `content_id` (from the submission response) and `creator_reasoning` (free text, up to 2000 chars).
- **What the system does:**
  1. Looks up the original audit log entry by `content_id`.
  2. Updates the entry's `status` field from `"classified"` to `"under_review"`.
  3. Appends `appeal_reasoning` and `appeal_timestamp` to the audit record.
  4. Returns a confirmation JSON with the updated status.
- **What a human reviewer sees:** The full audit entry — original signals, confidence score, attribution label, and the creator's reasoning — in a single structured record.
- **No automated re-classification:** The system flags; humans decide.

---

## Anticipated Edge Cases

1. **Formal academic writing by a human:** A professor writing a dense, structured literature review will have low sentence length variance, formal vocabulary, and hedged language — all patterns the stylometric heuristics associate with AI. The LLM signal may also flag it. This content will likely land in the uncertain band or score as AI, requiring the professor to appeal.

2. **Minimally edited AI output:** A user who prompts an LLM and then does light editing (fixing a name, adjusting a sentence) produces text that is structurally AI but may trigger less certainty from the LLM classifier. The system will likely output a mid-range score (0.5–0.7) — uncertain or weak AI — which is actually the honest answer.

3. **Very short text (under 30 words):** Stylometric metrics become statistically unreliable with few sentences. The system will still run but the style signal is meaningless at this length. Output will lean entirely on the LLM score.

4. **Non-native English speakers:** Writers who learned English formally may produce highly regular, syntactically uniform text that mimics AI stylistic patterns. Combined with hedged phrasing, this can produce false positives.

---

## AI Tool Plan

### Milestone 3 — Submission Endpoint + Signal 1
- **Spec sections provided:** Detection signals (Signal 1), Architecture diagram
- **Request:** Flask app skeleton with `POST /submit` route stub + Groq LLM classifier function that returns a 0–1 score
- **Verification:** Call the function directly with clearly AI and clearly human text; confirm the score direction is correct before wiring into the route

### Milestone 4 — Signal 2 + Confidence Scoring
- **Spec sections provided:** Detection signals (Signal 2), Uncertainty representation, Architecture diagram
- **Request:** Stylometric heuristics function + weighted confidence scoring logic matching the 0.65/0.35 thresholds
- **Verification:** Run all four test cases from the spec; check that AI text scores ≥ 0.65 and casual human text scores ≤ 0.35

### Milestone 5 — Production Layer
- **Spec sections provided:** Transparency label variants, Appeals workflow, Architecture diagram
- **Request:** Label generation function (maps confidence → label text) + `POST /appeal` endpoint
- **Verification:** Submit inputs that produce all three label variants; submit an appeal and confirm `GET /log` shows `"status": "under_review"` with `appeal_reasoning` populated
