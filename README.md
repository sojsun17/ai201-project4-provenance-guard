# Provenance Guard

A backend content attribution API for creative platforms. Classifies submitted text as likely human-written or likely AI-generated using a multi-signal detection pipeline, returns confidence-calibrated transparency labels, and supports creator appeals.

---

## Architecture Overview

A submission travels through the following path:

1. `POST /submit` receives `{text, creator_id}`
2. A unique `content_id` (UUID) is generated
3. **Signal 1 (Groq LLM):** The text is sent to `llama-3.3-70b-versatile` with a structured prompt requesting a 0–1 AI-likelihood score and one-sentence reasoning
4. **Signal 2 (Stylometric Heuristics):** Pure Python analysis computes sentence length variance, type-token ratio, and punctuation density — combined into a 0–1 score
5. **Confidence Scorer:** `confidence = (0.65 × llm_score) + (0.35 × style_score)`
6. **Transparency Label Generator:** Maps confidence to one of three label texts
7. **Audit Log:** Full decision record written to SQLite
8. Response returned with `content_id`, `attribution`, `confidence`, `label`, both signal scores

For appeals: `POST /appeal` with `content_id` and `creator_reasoning` → status updated to `under_review` in the audit log → confirmation returned.

```
POST /submit ──→ Groq LLM ──────────→ llm_score ──→ Confidence Scorer ──→ Label ──→ Audit Log ──→ Response
              ──→ Stylometrics ──→ style_score ──┘
POST /appeal ──→ Lookup by content_id ──→ Update status ──→ Append appeal ──→ Audit Log ──→ Response
```

---

## Setup

```bash
git clone <your-repo-url>
cd ai201-project4-provenance-guard
python -m venv .venv
source .venv/bin/activate       # Mac/Linux
# .venv\Scripts\activate        # Windows

pip install -r requirements.txt
```

Create a `.env` file (never commit this):
```
GROQ_API_KEY=your_key_here
```

Run the server:
```bash
python app.py
```

---

## API Reference

### `POST /submit`
Accepts a piece of text for attribution analysis.

**Request:**
```json
{"text": "...", "creator_id": "user-123"}
```

**Response:**
```json
{
  "content_id": "3f7a2b1e-...",
  "attribution": "likely_ai",
  "confidence": 0.78,
  "confidence_pct": 78,
  "llm_score": 0.85,
  "style_score": 0.63,
  "llm_reasoning": "The text uses uniform hedged corporate phrasing.",
  "label": "⚠️ AI-Generated Content Detected\n...",
  "status": "classified"
}
```

Rate limited: **10 requests/minute, 100 requests/day** per IP.

---

### `POST /appeal`
Contest a classification.

**Request:**
```json
{"content_id": "3f7a2b1e-...", "creator_reasoning": "I wrote this myself..."}
```

**Response:**
```json
{
  "content_id": "3f7a2b1e-...",
  "status": "appeal_received",
  "message": "Your appeal has been recorded and the content is now under review.",
  "original_attribution": "likely_ai",
  "original_confidence": 0.78
}
```

---

### `GET /log`
Returns the most recent audit log entries.

```bash
curl http://localhost:5000/log
```

### `GET /status/<content_id>`
Returns a single audit record by ID.

---

## Detection Signals

### Signal 1: Groq LLM Classifier
Sends the submitted text to `llama-3.3-70b-versatile` with a system prompt asking it to score AI likelihood 0–1 and return structured JSON. The LLM captures holistic semantic and stylistic coherence — hedged language, structural uniformity, generic phrasing, lack of personal specificity.

**Why this signal:** LLMs are uniquely good at recognizing AI output because they were trained on similar patterns. It captures properties no statistical heuristic can — the "feel" of AI-generated text.

**What it misses:** Lightly edited AI output. Non-native English speakers whose formal writing patterns match AI. Very short texts where there's insufficient signal.

---

### Signal 2: Stylometric Heuristics

Three pure-Python metrics combined into a 0–1 score:

| Metric | What it measures | AI pattern |
|---|---|---|
| Sentence length variance | Standard deviation of word counts per sentence | Low variance (uniform) |
| Type-token ratio (TTR) | Unique words / total words | Mid-range TTR (0.55–0.80) |
| Punctuation density | Frequency of human-idiosyncratic punctuation (em-dash, ellipsis, !, ?, parens) | Low density |

**Why this signal:** Completely independent of the LLM — measures structural properties, not semantic meaning. AI text tends to be statistically more regular. This signal catches cases where the LLM is uncertain.

**What it misses:** Formal human writing (academic papers, legal documents) scores high on uniformity and triggers false positives. Very short texts (<2 sentences) produce unreliable statistics.

---

## Confidence Scoring

### Combination formula
```
confidence = (0.65 × llm_score) + (0.35 × style_score)
```

The LLM signal is weighted higher because it captures semantic coherence that stylometrics cannot. The 65/35 split also partially compensates for stylometrics over-firing on formal human writing.

### Score → Attribution mapping

| Confidence | Attribution | Label |
|---|---|---|
| 0.65 – 1.00 | `likely_ai` | ⚠️ AI-Generated Content Detected |
| 0.36 – 0.64 | `uncertain` | 🔍 Authorship Unclear |
| 0.00 – 0.35 | `likely_human` | ✅ Likely Human-Written |

The uncertain band is intentionally wide (29 points) to minimize false positives — misclassifying human work as AI is treated as the worse error.

### Example submissions showing score variation

**Example 1 — Clearly AI-generated (high confidence)**
> "Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous, it is equally essential to consider the ethical implications. Furthermore, stakeholders across various sectors must collaborate to ensure responsible deployment."

```
llm_score:   0.80
style_score: 0.50
confidence:  0.695  → likely_ai (70%)
llm_reasoning: "The text features generic corporate phrasing and a uniform sentence
structure, which are characteristic of AI-generated text."
```

**Example 2 — Clearly human-written (low confidence)**
> "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it and i was thirsty for like three hours after. my friend got the spicy version and said it was better. probably wont go back unless someone drags me there"

```
llm_score:   0.00
style_score: 0.5122
confidence:  0.1793  → likely_human (18%)
llm_reasoning: "The text exhibits characteristics of human-written language, including
informal grammar, personal anecdotes, and emotional directness."
```

The gap between these two cases (70% vs 18% confidence) demonstrates the scoring produces meaningful variation across clearly different inputs.

---

## Transparency Label Variants

All three label variants are shown below with the exact text the system returns:

### High-Confidence AI (`confidence ≥ 0.65`)
```
⚠️ AI-Generated Content Detected
Our analysis found strong signals that this content was likely generated by an AI tool. Confidence: [X]%
This label is based on automated analysis and may not be perfect. If you are the author and believe this is incorrect, you can submit an appeal.
```

### Uncertain (`0.35 < confidence < 0.65`)
```
🔍 Authorship Unclear
Our system could not determine with confidence whether this content was written by a human or generated by AI. It will be treated as unverified. Confidence: [X]%
If you are the author, you may submit an appeal to provide context about your writing process.
```

### High-Confidence Human (`confidence ≤ 0.35`)
```
✅ Likely Human-Written
Our analysis found strong signals that this content was written by a human author. Confidence: [X]%
Automated analysis is not perfect. This label reflects our best assessment based on available signals.
```

---

## Rate Limiting

**Limits applied to `POST /submit`:** 10 requests per minute, 100 requests per day (per IP address).

**Reasoning:**
- A real writer submitting their own work might post once every few minutes at most — 10/minute is generous for legitimate use.
- 100/day prevents a script from flooding the system overnight to probe the classifier or exhaust the Groq free tier.
- The per-minute limit stops burst abuse; the per-day limit stops sustained low-rate abuse.
- Neither limit would interrupt normal creative workflow.

**Rate limit test output (12 rapid requests — exceeds 10/minute limit):**
```
200
200
200
200
200
200
200
200
200
200
429
429
```

---

## Audit Log

Every attribution decision is written to SQLite. Real output from `GET /log` after running the test suite:

```json
{
  "count": 3,
  "entries": [
    {
      "appeal_reason": null,
      "appeal_time": null,
      "attribution": "likely_ai",
      "confidence": 0.7243,
      "content_id": "7462a46e-263c-46f0-b569-2ae71c30f453",
      "creator_id": "test-3",
      "llm_score": 0.8,
      "style_score": 0.5837,
      "status": "classified",
      "timestamp": "2026-06-30T20:34:41.517546+00:00"
    },
    {
      "appeal_reason": null,
      "appeal_time": null,
      "attribution": "likely_human",
      "confidence": 0.1793,
      "content_id": "f485309a-e72a-42f9-8c01-0a1975256a17",
      "creator_id": "test-2",
      "llm_score": 0.0,
      "style_score": 0.5122,
      "status": "classified",
      "timestamp": "2026-06-30T20:34:34.827098+00:00"
    },
    {
      "appeal_reason": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical.",
      "appeal_time": "2026-06-30T20:34:50.434126+00:00",
      "attribution": "likely_ai",
      "confidence": 0.695,
      "content_id": "b0c1aa8d-d10c-48af-b0e1-ac7cb602dfc7",
      "creator_id": "test-1",
      "llm_score": 0.8,
      "style_score": 0.5,
      "status": "under_review",
      "timestamp": "2026-06-30T20:34:11.410863+00:00"
    }
  ]
}
```

The third entry shows a complete appeal flow: `status` updated to `under_review`, `appeal_reason` populated, and `appeal_time` recorded.

---

## Known Limitations

**1. Formal human writing produces false positives.** A professor writing a structured academic essay — uniform sentences, formal vocabulary, hedged language — will consistently score in the AI range. Both signals are calibrated against casual human writing patterns. A deployed system would need a domain-aware signal or a profession/context field in the submission to correct for this.

**2. Very short texts are unreliable.** The stylometric signal requires at least 2–3 sentences for meaningful statistics. A haiku, a tweet-length submission, or a single paragraph will lean entirely on the LLM score, which is itself less reliable without adequate context. The system should flag these with a warning rather than returning a confident label.

---

## Spec Reflection

**One way the spec helped:** Writing the three transparency label variants in `planning.md` before touching any code forced a concrete decision about the uncertain band threshold. When I initially sketched a 0.5 cutoff, I noticed the label text at 0.51 ("high-confidence AI") would be misleading — it's barely over the midpoint. The spec writing step pushed the thresholds to 0.35/0.65, making "uncertain" a real category rather than a rounding artifact.

**One way implementation diverged from the spec:** The TTR metric in stylometrics ended up less reliable than planned. Mid-range TTR (0.55–0.80) was supposed to be the AI signal, but in testing, both formal human writing and casual AI outputs fall into this range for different reasons. The metric became a weak discriminator and I reduced its weight within the stylometric combination. A more principled approach would use character-level n-gram analysis instead of word-level TTR.

---

## AI Usage

**Instance 1 — LLM system prompt design:** I directed the AI to help draft the system prompt for Signal 1. It produced a working prompt, but the initial version asked for a verbose explanation rather than structured JSON, which made parsing fragile. I revised the output format instruction to explicitly request `{"score": float, "reasoning": "one sentence"}` and added explicit examples of what 0.0 / 0.5 / 1.0 represent.

**Instance 2 — Stylometric scoring logic:** I asked the AI to implement the three stylometric metrics based on my spec. It produced a reasonable skeleton but the variance normalization was wrong — it used a fixed divisor of 5 for standard deviation, which caused most inputs to cap at 1.0. I overrode the divisor to 12 based on empirical testing with the four sample inputs from the milestone, which produced a more calibrated spread.

---

## Tech Stack

| Component | Tool |
|---|---|
| API framework | Flask 3.x |
| Detection signal 1 | Groq (`llama-3.3-70b-versatile`) |
| Detection signal 2 | Pure Python stylometrics |
| Rate limiting | Flask-Limiter 3.x |
| Audit log | SQLite (built-in) |
| Environment | python-dotenv |