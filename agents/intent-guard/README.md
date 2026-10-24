# IntentGuard — AI Code Intent & Correctness Verifier

> An autonomous verification agent that audits code produced by AI coding assistants (Cursor, Copilot, Claude, etc.) against original product specs, JIRA tickets, and HLDs to catch specification drift, subtle security misinterpretations, and scope creep before you merge.

---

## Overview

### Problem Statement & Intended Users

- **Problem**: Every day, software engineers feed context — JIRA tickets, High-Level Design documents, meeting notes, acceptance criteria — into AI coding assistants like Cursor, GitHub Copilot, or Claude. These tools generate changes across 10–30 files. The developer glances through the output, thinks _"looks about right"_, and pushes to Git. But **nobody systematically verifies that the generated code actually fulfills the original requirements**. Existing tools (PR review bots, linters, CI tests) check code _quality_ — they don't check **intent alignment**.

- **Intended Users**: Software engineers, tech leads, and engineering teams who use AI coding assistants in their daily workflow and want an automated second opinion on whether the AI's output matches the original task requirements.

### Key Features

- **Intent-to-Implementation Verification**: Takes the same context you gave your coding AI (JIRA ticket, HLD, notes) and cross-references it against the generated code to check if every requirement is correctly addressed.
- **Structured Verdicts**: Returns a JSON report with per-requirement status (`implemented` / `partially_implemented` / `missing` / `misinterpreted`), scope creep detection, specific issues, and an overall pass/fail/needs_revision verdict.
- **Multi-Layer Review**: Extracts structured requirements first, then systematically checks each against the code diff — mimicking how a thorough human reviewer works.
- **Deterministic Diff & AST Tools**: Features built-in deterministic tools (`parse_diff_metadata` and `check_python_syntax`) providing the agent with structural diff metrics and AST validation.
- **FastAPI Server + CLI**: Use as a REST API endpoint in your workflow, or run directly from the terminal with a single command featuring Rich table formatting.
- **Built-in Demo**: Ships with an example JIRA ticket and intentionally flawed AI-generated code to demonstrate the agent's detection capabilities.

---

## Technical Stack

- **Language & Runtime**: Python 3.9+
- **Agent Framework**: [Google Agent Development Kit (ADK)](https://adk.dev/) — provides the LLM agent loop, tool calling, and session management
- **Model**: Google Gemini 3.5 Flash (default) or Gemini 3.8 Flash (via `google-adk`)
- **API Server**: FastAPI + Uvicorn
- **CLI Output**: Rich (terminal tables and formatted output)
- **Data Validation**: Pydantic v2

---

## Architecture & Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     USER INPUT                              │
│  ┌──────────────────────┐  ┌─────────────────────────────┐  │
│  │  Intent Context      │  │  Code Changes               │  │
│  │  • JIRA ticket       │  │  • Unified diff / patch     │  │
│  │  • HLD document      │  │  • Generated file contents  │  │
│  │  • Meeting notes     │  │                             │  │
│  │  • Acceptance criteria│ │                             │  │
│  └──────────┬───────────┘  └──────────────┬──────────────┘  │
└─────────────┼─────────────────────────────┼─────────────────┘
              │                             │
              ▼                             │
   ┌──────────────────────┐                 │
   │  PASS 1              │                 │
   │  Requirement         │                 │
   │  Extraction          │                 │
   │  (LLM analyzes intent│                 │
   │   into structured    │                 │
   │   requirements list) │                 │
   └──────────┬───────────┘                 │
              │ Structured Requirements     │
              ▼                             ▼
   ┌──────────────────────────────────────────┐
   │  PASS 2                                  │
   │  Correctness Cross-Reference             │
   │                                          │
   │  For each requirement:                   │
   │  • Implemented? Partially? Missing?      │
   │  • Misinterpreted?                       │
   │  • Evidence from the diff                │
   │                                          │
   │  For each code change:                   │
   │  • Maps to which requirement?            │
   │  • Scope creep?                          │
   │                                          │
   │  Check for:                              │
   │  • Logic errors   • Security concerns    │
   │  • Wrong algorithms • Missing edge cases │
   │  (Supported by parse_diff_metadata and   │
   │   check_python_syntax tools)             │
   └──────────────────┬───────────────────────┘
                      │
                      ▼
   ┌──────────────────────────────────────────┐
   │  STRUCTURED VERDICT (JSON)               │
   │  • verdict: pass / fail / needs_revision │
   │  • confidence: 0.0 - 1.0                │
   │  • requirements: per-requirement results │
   │  • unmatched_changes: scope creep        │
   │  • issues: [{severity, category, ...}]   │
   │  • summary: human-readable paragraph     │
   └──────────────────────────────────────────┘
```

---

## Prerequisites

Before running this project, ensure you have:

- **Python 3.9+** (`python3 --version`)
- **pip** (included with Python)
- **Google Gemini API Key**: Obtain from [Google AI Studio](https://aistudio.google.com/apikey)

---

## Installation & Setup

1. **Navigate to the project directory:**
   ```bash
   cd agents/intent-guard
   ```

2. **Set up the virtual environment and install dependencies:**
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate   # macOS/Linux
   # .venv\Scripts\activate    # Windows

   pip install -r requirements.txt
   ```

3. **Configure Environment Variables:**
   ```bash
   cp .env.example .env
   ```
   Then edit `.env` and add your Google API key:
   ```bash
   GOOGLE_API_KEY=your_actual_api_key_here
   MODEL_NAME=gemini-3.5-flash
   ```

---

## Configuration & Environment Variables

| Variable Name    | Required | Default            | Description                                |
| :--------------- | :------- | :----------------- | :----------------------------------------- |
| `GOOGLE_API_KEY` | Yes      | *None*             | API key for Google Gemini                  |
| `GEMINI_API_KEY` | Yes      | *None*             | Gemini API key alias                       |
| `MODEL_NAME`     | No       | `gemini-3.5-flash` | Gemini model (`gemini-3.5-flash` or `3.8`) |
| `HOST`           | No       | `0.0.0.0`          | FastAPI server bind address                |
| `PORT`           | No       | `8000`             | FastAPI server port                        |
| `LOG_LEVEL`      | No       | `info`             | Logging verbosity                          |

---

## How to Run

### Option 1: CLI — Built-in Demo

Run the included example scenario (JIRA ticket + intentionally flawed AI code):

```bash
# From the project directory, with venv activated:
python -m intent_guard.cli --example
```

### Option 2: CLI — Your Own Files

```bash
python -m intent_guard.cli \
    --intent path/to/jira_ticket.md \
    --diff path/to/changes.patch \
    --language python
```

Add `--json` for raw JSON output instead of the formatted Rich table.

### Option 3: ADK Web UI

```bash
adk web intent_guard
```

### Option 4: FastAPI Server

```bash
uvicorn intent_guard.server:app --reload
```

Then send a POST request:

```bash
curl -X POST http://localhost:8000/review \
  -H "Content-Type: application/json" \
  -d '{
    "intent_context": "JIRA-1234: Add login endpoint with bcrypt...",
    "code_diff": "diff --git a/auth.py ...",
    "language": "python"
  }'
```

---

## Example Input & Output

### Example Input

**Intent** (JIRA ticket): Add JWT authentication with RS256, bcrypt password hashing, rate limiting, and proper 401 error responses.

**Code diff**: AI-generated code that uses HS256 instead of RS256, SHA-256 instead of bcrypt, omits rate limiting, returns 403 instead of 401, and adds an unrequested `/api/admin/reset` endpoint.

_(Full example files are in `intent_guard/examples/`)_

### Example Output

```json
{
  "verdict": "fail",
  "confidence": 1.0,
  "requirements": [
    {
      "id": "REQ-1",
      "description": "POST /api/auth/login endpoint",
      "status": "implemented",
      "evidence": "app/auth/router.py:46",
      "explanation": "Login endpoint exists and accepts email + password."
    },
    {
      "id": "REQ-2",
      "description": "JWT signed with RS256 algorithm",
      "status": "misinterpreted",
      "evidence": "app/auth/router.py:18",
      "explanation": "Code uses HS256 with a symmetric secret key instead of RS256 with RSA key pair."
    },
    {
      "id": "REQ-3",
      "description": "Passwords hashed with bcrypt (12 rounds)",
      "status": "misinterpreted",
      "evidence": "app/auth/router.py:34",
      "explanation": "Uses SHA-256 hashing instead of bcrypt. SHA-256 is not a password hashing algorithm."
    },
    {
      "id": "REQ-4",
      "description": "Rate limiting on login (5 per IP per 15 min)",
      "status": "missing",
      "evidence": null,
      "explanation": "No rate limiting implementation found anywhere in the diff."
    }
  ],
  "unmatched_changes": [
    {
      "location": "app/admin/router.py",
      "description": "Unrequested /api/admin/reset endpoint that resets all passwords to plaintext",
      "risk_assessment": "Critical security risk — stores passwords in plaintext and was not requested"
    }
  ],
  "issues": [
    {
      "severity": "critical",
      "category": "security_concern",
      "description": "Unrequested admin password reset endpoint POST /api/admin/reset stores passwords in plaintext.",
      "location": "app/admin/router.py:18",
      "requirement_id": null,
      "suggestion": "Delete app/admin/router.py entirely as it is out of scope and introduces critical security vulnerabilities."
    }
  ],
  "summary": "The AI-generated code fails compliance against JIRA-1234 requirements and introduces severe security flaws alongside scope creep..."
}
```

---

## How to Test

```bash
# Run the built-in demo scenario and verify formatted Rich output:
python -m intent_guard.cli --example

# Run with JSON output for programmatic verification:
python -m intent_guard.cli --example --json

# Start the API server and test the health endpoint:
uvicorn intent_guard.server:app &
curl http://localhost:8000/health
```

---

## Known Limitations

- **LLM-dependent accuracy**: The quality of the review depends on the underlying Gemini model's reasoning capabilities.
- **Static Analysis Scope**: v1 performs static diff reasoning and AST checks without active runtime execution or dynamic sandboxing.
- **Manual Diff Input**: Diffs must be supplied manually or via pipeline automation; automatic GitHub PR webhook fetching is slated for v2.

---

## Security & Privacy Considerations

- **Secrets Handling**: API keys are loaded from environment variables via `.env` files, which are excluded from version control by `.gitignore`.
- **Data Privacy**: Intent context and code diffs are sent to the Google Gemini API for processing. If your code is sensitive, consider using a self-hosted model endpoint.
- **Safety**: This agent performs read-only analysis — it does not execute code, modify files, or make any changes to your codebase.

---

## Future Improvements

- [ ] Add deterministic pre-checks (Ruff linting, syntax validation) before LLM review
- [ ] Git integration — automatically fetch diffs from GitHub / GitLab PRs
- [ ] Sandbox code execution — run existing tests against the generated code
- [ ] Test generation — create edge-case tests for the changed code
- [ ] Multi-provider support (Anthropic Claude, OpenAI, local models via Ollama)
- [ ] MCP server mode for integration with coding assistants
- [ ] Persistent review history and trend tracking
- [ ] CI/CD pipeline integration (GitHub Action)

---

## License

*This project is contributed under the repository's open-source license.*


---

## Known Limitations

- **LLM-dependent accuracy**: The quality of the review depends on the underlying Gemini model's reasoning capabilities. Complex architectural requirements may be partially assessed.
- **No code execution**: v1 does not execute the generated code or run tests — it performs static analysis via LLM reasoning only.
- **Single-language focus**: While the agent can review any language, the examples and testing focus on Python.
- **No git integration**: You must manually provide the diff — the agent does not fetch diffs from GitHub/GitLab PRs (planned for v2).
- **Token limits**: Very large diffs may exceed the model's context window.

---

## Security & Privacy Considerations

- **Secrets Handling**: API keys are loaded from environment variables via `.env` files, which are excluded from version control by `.gitignore`.
- **Data Privacy**: Intent context and code diffs are sent to the Google Gemini API for processing. If your code is sensitive, consider using a self-hosted model endpoint.
- **Safety**: This agent performs read-only analysis — it does not execute code, modify files, or make any changes to your codebase.

---

## Future Improvements

- [ ] Add deterministic pre-checks (Ruff linting, syntax validation) before LLM review
- [ ] Git integration — automatically fetch diffs from GitHub / GitLab PRs
- [ ] Sandbox code execution — run existing tests against the generated code
- [ ] Test generation — create edge-case tests for the changed code
- [ ] Multi-provider support (Anthropic Claude, OpenAI, local models via Ollama)
- [ ] MCP server mode for integration with coding assistants
- [ ] Persistent review history and trend tracking
- [ ] CI/CD pipeline integration (GitHub Action)

---

## License

*This project is contributed under the repository's open-source license.*

