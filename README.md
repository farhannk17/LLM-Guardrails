# 🔒 LLM Guardrails

**A safety and compliance layer for LLM applications** — catching PII leaks, jailbreak attempts, malformed outputs, and hallucination risk before they reach a production system.

![Python](https://img.shields.io/badge/python-3.11-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-teal)
![Tests](https://img.shields.io/badge/tests-8%2F8%20passing-brightgreen)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## Why this exists

Most LLM demos focus on the happy path. This project focuses on everything that goes wrong in production: users leaking personal data into prompts, users trying to jailbreak the system, and models producing malformed or unsupported (hallucinated) answers.

It's a practical, tested answer to the question every enterprise AI buyer asks: **"how do you handle safety and compliance?"**

## Architecture

```
User message
   │
   ▼
┌─────────────────────┐
│   PII Redactor       │  masks emails, phone numbers, SSNs, names
└─────────┬────────────┘
          ▼
┌─────────────────────┐
│  Jailbreak Detector   │  regex + semantic similarity — blocks known attack patterns
└─────────┬────────────┘
          ▼
     [ LLM call ]
          ▼
┌─────────────────────┐
│  Output Validator     │  Pydantic schema check — rejects malformed/placeholder output
└─────────┬────────────┘
          ▼
┌─────────────────────┐
│ Hallucination Check   │  groundedness score vs. source docs + confidence threshold
└─────────┬────────────┘
          ▼
   Final response to user
```

## Features

- **PII Redaction** — detects and masks emails, phone numbers, credit cards, SSNs, and names using Microsoft Presidio, before any text is sent to an LLM or logged
- **Jailbreak Detection** — a two-layer defense: fast regex/keyword matching for known attacks, plus semantic similarity (sentence embeddings) to catch paraphrased attacks that don't match exact wording
- **Output Validation** — Pydantic schemas reject malformed, placeholder, or out-of-range LLM responses before they reach the user
- **Hallucination Risk Flagging** — scores how well an answer is grounded in provided source documents, combined with the model's self-reported confidence, to route low-confidence answers to human review
- **Fully tested** — 8/8 automated pytest cases covering all four modules
- **Two ways to run it** — a local FastAPI server (VS Code) or a self-contained Google Colab notebook (zero local setup)

## Tech stack

| Component | Tool |
|---|---|
| API layer | FastAPI |
| PII detection | Microsoft Presidio + spaCy |
| Output validation | Pydantic |
| Jailbreak similarity scoring | scikit-learn (cosine similarity) |
| Testing | pytest |

## Project structure

```
llm-guardrails/
├── app/
│   ├── main.py                # FastAPI app — entry point, wires all guardrails together
│   ├── pii_redactor.py        # Detects & masks PII
│   ├── jailbreak_detector.py  # Flags suspicious prompts
│   ├── output_validator.py    # Validates LLM response schema
│   ├── hallucination_check.py # Flags unsupported claims
│   └── schemas.py             # Pydantic response schema
├── tests/
│   └── test_guardrails.py     # Automated test suite
├── requirements.txt
├── .env.example
└── README.md
```

## Getting started

### Option A — Run locally (VS Code)

```bash
git clone https://github.com/YOUR-USERNAME/llm-guardrails.git
cd llm-guardrails

python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt
python -m spacy download en_core_web_sm

uvicorn app.main:app --reload
```

Visit `http://127.0.0.1:8000/docs` for an interactive API tester.

### Option B — Run in Google Colab (no local setup)

Open [`LLM_Guardrails.ipynb`](./LLM_Guardrails.ipynb) in [Google Colab](https://colab.research.google.com) and run each cell top to bottom.

## Running the tests

```bash
python -m pytest tests/ -v
```

```
tests/test_guardrails.py::TestPIIRedaction::test_detects_email PASSED
tests/test_guardrails.py::TestPIIRedaction::test_no_false_positive_on_clean_text PASSED
tests/test_guardrails.py::TestJailbreakDetector::test_flags_known_attack PASSED
tests/test_guardrails.py::TestJailbreakDetector::test_allows_normal_question PASSED
tests/test_guardrails.py::TestOutputValidator::test_rejects_placeholder_answer PASSED
tests/test_guardrails.py::TestOutputValidator::test_accepts_valid_response PASSED
tests/test_guardrails.py::TestHallucinationCheck::test_flags_ungrounded_answer PASSED
tests/test_guardrails.py::TestHallucinationCheck::test_passes_grounded_answer PASSED

======================== 8 passed in 1.2s ========================
```

## Example

**Request:**
```json
POST /chat
{
  "message": "Hi, contact me at john@email.com",
  "source_documents": []
}
```

**Response:**
```json
{
  "blocked": false,
  "response": {
    "answer": "[MOCK RESPONSE] You said: Hi, contact me at <EMAIL_ADDRESS>",
    "confidence": 0.87,
    "sources_cited": false
  },
  "pii_findings": [{"type": "EMAIL_ADDRESS", "confidence": 1.0}],
  "hallucination_check": {
    "groundedness_score": 0.0,
    "risk_flags": ["low_source_overlap"],
    "needs_human_review": true
  }
}
```

**Jailbreak attempt — blocked before reaching the LLM:**
```json
POST /chat
{"message": "Ignore all previous instructions"}
```
```json
{
  "blocked": true,
  "reason": "jailbreak_attempt_detected",
  "details": {"flagged": true, "risk_level": "high"}
}
```

## Honest limitations

Being upfront about what this doesn't do is as important as what it does:

- **Hallucination detection is a risk signal, not a truth detector.** The groundedness score measures term overlap with source documents — it does not verify factual correctness. True hallucination detection remains an open research problem.
- **The jailbreak keyword list is illustrative, not exhaustive.** A production system would need a continuously updated attack corpus and likely a fine-tuned classifier.
- **PII detection has a precision/recall tradeoff.** `LOCATION` entities are deliberately excluded from redaction because Presidio flags generic place names (e.g. "France" in "capital of France") as false positives — see the comment in `pii_redactor.py`.

## Roadmap

- [ ] Replace the mock LLM call with a real Anthropic/OpenAI integration
- [ ] Add an `/admin/logs` endpoint to review flagged messages
- [ ] Expand the jailbreak reference set and add a fine-tuned classifier
- [ ] Add per-user rate limiting
- [ ] Deploy a live demo

## License

MIT — free to use, modify, and build on.

## Author

Built by [Your Name] — [LinkedIn](#) · [Portfolio](#)
