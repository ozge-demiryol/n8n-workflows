# AI Customer Support Agent

> AI-powered customer support automation with LLM-as-Judge evaluation and hallucination reduction — enterprise-grade n8n workflow.

🇹🇷 For Turkish version of this README file, see [README-TR.md](README-TR.md)

---

## Table of Contents

- [AI Customer Support Agent](#ai-customer-support-agent)
  - [Table of Contents](#table-of-contents)
  - [Problem](#problem)
  - [Solution \& Vision](#solution--vision)
  - [Outcomes \& Success Criteria](#outcomes--success-criteria)
  - [Features](#features)
  - [Architecture Overview](#architecture-overview)
  - [Architecture Decisions](#architecture-decisions)
    - [LLM-as-Judge Pattern](#llm-as-judge-pattern)
  - [Installation](#installation)
    - [Prerequisites](#prerequisites)
    - [Steps](#steps)
  - [Usage](#usage)
    - [Basic example](#basic-example)
  - [Configuration](#configuration)
  - [Deployment](#deployment)

---

## Problem

Manual customer support handling is slow, expensive, and prone to human error. Naive AI automation introduces "hallucinations"—where the AI invents policies, prices, or facts—risking brand reputation and customer trust. Businesses need automated responses that guarantee 100% factual accuracy against a grounded knowledge base.

---

## Solution & Vision

This automation system implements a dual-layer validation system: every AI-generated draft is evaluated by a secondary "Judge LLM" against a knowledge base before reaching the customer. Our vision is to eliminate 90% of routine support tickets while maintaining a human-in-the-loop safety net for complex issues.

---

## Outcomes & Success Criteria

| Metric             | Before       | After (target) |
| ------------------ | ------------ | -------------- |
| Response Time      | 2–4 hours    | < 2 minutes    |
| Hallucination Rate | Unknown/High | < 1%           |
| Automation Rate    | 0%           | 70–85%         |

> [!NOTE]
> Success is defined by the "Judge" node returning a score of 1 based on category accuracy, grounding, and proper escalation.

---

## Features

- **Multi-Step Classification** — Categorizes emails into Setup, Security, Pricing, Billing, Legal, Sales, or HR.
- **RAG Integration** — Uses Supabase Vector Store and Gemini Embeddings for factual retrieval.
- **LLM-as-Judge Evaluation** — Gemini 3 Flash based judge scores replies for accuracy before sending them customers.
- **Fail-Safe Escalation** — Routes complex or sensitive issues (e.g., hacked accounts) to human agents.
- **Automated Knowledge Refresh** — Daily scheduler synchronizes Google Sheets data with the vector database.

---

## Architecture Overview

```
[Webhook/Gmail] 
       ↓
[GPT-4o Mini Classifier]
       ↓
[AI Agent + RAG]
       ↓
[Judge LLM Evaluation]
       ├─→ Score 1 → [Send Email]
       └─→ Score 0 → [Human Escalation]

[Daily Schedule] 
       ↓
[Sync Sheets to Supabase]
```

**Tech stack:**

| Technology         | Role             | Why chosen                                           |
| ------------------ | ---------------- | ---------------------------------------------------- |
| n8n                | Orchestration    | Low-code flexibility with powerful error handling.   |
| GPT-4o Mini        | Classifier/Agent | High performance-to-cost ratio for routine tasks.    |
| Gemini 3 Flash     | Judge LLM        | Stronger model for more accurate evaluation.         |
| Gemini Embedding 2 | Vectorization    | Superior semantic search capabilities.               |
| Supabase           | Vector Store     | Scalable, open-source Postgres-based vector search.  |
| Google Sheets      | Knowledge Base   | Easy for non-technical staff to update product data. |

---

## Architecture Decisions

### LLM-as-Judge Pattern

**Context:** AI agents can sometimes ignore system prompts or fabricate data (hallucinations). A single layer of validation is insufficient for production use.

**Decision:** Implement a separate LLM node specifically to grade the output of the first agent against the knowledge base before delivery.

**Alternatives considered:**
- Single agent with stronger system prompt — insufficient isolation of concerns.
- Manual human review of every response — bottleneck and not cost efficient.

**Trade-offs:**
- ✅ Massive reduction in incorrect replies.
- ✅ Automated auditing of performance.
- ⚠️ Increased token cost per email (~2 API calls per ticket).
- ⚠️ Slightly higher latency (approx. +3 seconds per email).

---

## Installation

### Prerequisites

- n8n (v1.5+ recommended)
- OpenAI API key
- Google Cloud Console access (for Sheets and Gemini)
- Supabase account

### Steps

1. Download `customer_support_inbox_evaluation.json` from this repository.
2. In n8n, click **Workflows** → **Import from File**.
3. Configure your credentials for OpenAI, Google Gemini, and Supabase.
4. Update `YOUR_SPREADSHEET_ID` in all Google Sheets nodes with your knowledge base spreadsheet ID.
5. Set the **Webhook** node's URL as your inbox webhook endpoint (e.g., Gmail forwarding or Zapier).

---

## Usage

### Basic example

Once active, send a POST request to the webhook URL:

```json
{
  "from": "customer@example.com",
  "subject": "Setup issue",
  "body": "How do I connect the API?"
}
```

The workflow will:
1. Classify the email into a category.
2. Query Supabase for relevant documentation.
3. Generate a response via the AI Agent.
4. Judge the response for accuracy.
5. Send the email or escalate to Slack if quality is low.

---

## Configuration

| Variable          | Required | Description                              |
| ----------------- | -------- | ---------------------------------------- |
| `openai_api_key`  | ✅        | Used for classification and judgment.    |
| `supabase_url`    | ✅        | Vector database endpoint.                |
| `spreadsheet_id`  | ✅        | Google Sheets ID for the knowledge base. |
| `judge_threshold` | ❌        | Score threshold for auto-sending (0–1).  |

---

## Deployment

The workflow runs as an active trigger on n8n Cloud or self-hosted. Configure via environment variables or n8n Secrets. For high volume, enable **Queue Mode** to prevent blocking on API latency.
