# utomotive-Requirements-to-HIL-Test-Case-Generator-n8n-LLM-

# Automotive Requirements to HIL Test Cases (n8n + LLM)

An [n8n](https://n8n.io) workflow that turns automotive requirement documents (PDF or plain text) into structured **Hardware-in-the-Loop (HIL) test cases**. It then checks the result for coverage gaps, fills them with a second LLM pass, and exports a traceability matrix, a CSV and a summary report.

It works with local models through **Ollama** or with cloud APIs (**OpenAI, Google Gemini, xAI Grok, Groq, Mistral, DeepSeek, OpenRouter**) by changing one setting.

---

## Features

- **Flexible input:** send JSON text or upload a PDF to a webhook, or run the built-in sample with the manual trigger.
- **Requirement splitting:** requirement IDs (`REQ-…`, `SYS-…`, `SWR-…` and others, configurable by regex) are detected deterministically, so each requirement is processed on its own. If no IDs are found, it falls back to paragraph chunking.
- **Structured extraction:** for each requirement the LLM returns requirement type, ASIL level, ECU, operating conditions, signals, expected results, boundary conditions and ambiguities.
- **Test case generation:** positive, negative, boundary (min-delta, min, max, max+delta), timing and fault-injection tests, each with concrete stimulus steps and measurable expected results.
- **Automated coverage analysis (JavaScript):**
  - requirement coverage and traceability matrix
  - duplicate and near-duplicate test detection (Jaccard similarity on `signal=value` pairs)
  - missing positive, negative, boundary, timing and fault-injection tests (fault injection is required for ASIL or diagnostic requirements)
  - expected-result and operating-condition coverage
  - signal validation against an optional signal dictionary
  - duplicate or undefined requirement IDs and ambiguous requirement text
- **Gap-fill pass:** the LLM is asked only for the missing tests, then everything is re-analysed.
- **Multi-provider:** any OpenAI-compatible `/chat/completions` endpoint, with retries, token limits and error recovery.
- **Exports:** JSON response, CSV (one row per test case) and a Markdown summary report.

---

## Workflow overview

```
Webhook / Manual Trigger
        |
   Has File? --yes--> Extract From PDF
        |                    |
        +--------------------+
                 |
          Prepare Input  -->  Config  -->  Resolve Provider
                                                  |
                                         Split Requirements
                                                  |
                                    LLM: Extract Test Cases
                                                  |
                                          Parse LLM Output
                                                  |
                                               Analyze
                                                  |
                                          Gaps to Fill? --yes--> Build Gap Prompts --> LLM: Fill Gaps --> Parse Gap Output
                                                  |no                                                           |
                                                  +-------------------------> Finalize Report <----------------+
                                                                                    |
                                                                          Respond to Webhook
```

---

## Requirements

- n8n (self-hosted or cloud)
- **One** of:
  - [Ollama](https://ollama.com) running locally (for example `llama3.1:8b`, with a context window of 8k or more), or
  - an API key from a supported provider

---

## Setup

1. In n8n, go to **Workflows → Import from file** and select `req_to_testcases_n8n_cloud_llm.json`.
2. Open the **Config** node and set:
   - `default_provider`: `openai`, `gemini`, `grok`, `groq`, `mistral`, `deepseek`, `openrouter`, `ollama` or `custom`
   - the matching API key field (for example `gemini_api_key`)
   - `default_model`: optional, leave empty to use the provider's default
3. Click **Execute workflow** on the **Manual Trigger**. This runs a built-in sample BCM (Body Control Module) specification.

### Supported providers

| Provider | `default_provider` | Default model (change if needed) |
|---|---|---|
| OpenAI | `openai` | `gpt-4.1-mini` |
| Google Gemini | `gemini` | `gemini-2.5-flash` |
| xAI Grok | `grok` | `grok-4` |
| Groq | `groq` | `llama-3.3-70b-versatile` |
| Mistral | `mistral` | `mistral-large-latest` |
| DeepSeek | `deepseek` | `deepseek-chat` |
| OpenRouter | `openrouter` | `openai/gpt-4.1-mini` |
| Ollama (local) | `ollama` | `llama3.1:8b` |
| Any OpenAI-compatible API | `custom` | set `custom_base_url` and `default_model` |

> Model names change often. Check your provider's current model list.

### Configuration options

| Field | Description |
|---|---|
| `req_id_pattern` | Regex used to detect requirement IDs |
| `dup_threshold` | Similarity (0 to 1) above which two tests count as duplicates (default `0.8`) |
| `gap_fill_enabled` | Run the second LLM pass to close coverage gaps |
| `max_tokens` | Output token limit per LLM call (default `6000`). Increase it if responses get cut off |

### Free ways to test

- **Google Gemini:** free API key from [Google AI Studio](https://aistudio.google.com).
- **Groq:** free API key from [Groq Console](https://console.groq.com).
- **Ollama:** fully local and free.

Free-tier limits change, so check each provider's current limits.

---

## Usage

### Option 1: Manual trigger

Run the workflow from the n8n editor to process the built-in sample.

### Option 2: Webhook (JSON)

```bash
curl -X POST http://localhost:5678/webhook/req-to-testcases \
  -H "Content-Type: application/json" \
  -d '{
    "document_text": "REQ-BCM-001: When IgnSts is ON (2) and AmbLight_lux is below 400 lux for more than 2 s, the BCM shall switch on the low-beam headlamps (HeadlampCmd = 1) within 300 ms.",
    "signal_dictionary": ["IgnSts", "AmbLight_lux", "HeadlampCmd"],
    "provider": "gemini",
    "model": "gemini-2.5-flash"
  }'
```

`signal_dictionary`, `provider` and `model` are optional. Use `/webhook-test/...` while testing in the editor, and `/webhook/...` once the workflow is active.

### Option 3: Webhook (PDF upload)

```bash
curl -X POST http://localhost:5678/webhook/req-to-testcases \
  -F "data=@spec.pdf"
```

---

## Output

The webhook returns JSON with:

| Field | Content |
|---|---|
| `summary` | Counts and coverage percentages (requirement, expected-result, boundary, negative) |
| `test_cases` | Every test case: ID, type, priority, preconditions, stimulus steps, expected results, pass criteria, HIL notes, origin, status and flags |
| `traceability` | Requirement to test case matrix |
| `gaps` | Remaining coverage gaps, ranked by severity |
| `duplicates` | Detected duplicate or overlapping tests |
| `extraction_errors` | Requirements the LLM output could not be parsed for |
| `csv` | CSV export, one row per test case |
| `report_markdown` | Human-readable summary |

### Example test case

```json
{
  "tc_id": "TC-REQ-BCM-001-01",
  "req_id": "REQ-BCM-001",
  "title": "Switch on low beam when ambient light is below threshold",
  "type": "positive",
  "priority": "high",
  "preconditions": ["IgnSts=2"],
  "stimulus": [{ "signal": "AmbLight_lux", "value": "399", "unit": "lux", "wait_ms": 2100 }],
  "expected": [{ "signal": "HeadlampCmd", "value": "1", "tolerance": "", "within_ms": 300 }],
  "pass_criteria": "HeadlampCmd = 1 within 300 ms after AmbLight_lux stays below 400 lux for more than 2 s"
}
```

*(Illustrative example. Actual output depends on the model.)*

---

## Notes and limitations

- Generated test cases are **drafts and must be reviewed by an engineer**. The prompts forbid inventing signals, thresholds or timings, and unclear points are reported as ambiguities, but LLMs can still make mistakes.
- Output quality depends on the model. Small local models are less reliable than larger cloud models.
- Long documents make one LLM call per requirement, plus one more per requirement that has gaps. Watch rate limits and costs on cloud APIs.
- **Security:** API keys are stored in the Config node. Do not commit or share an exported workflow with keys filled in. Use n8n credentials for production use.

---

## Tech stack

n8n, JavaScript (Code nodes), REST APIs, Webhooks, JSON, OpenAI-compatible LLM APIs, Ollama
