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
> Success is defined by the Judge LLM returning a **"pass"** verdict. Each reply is scored across three dimensions (0–2 each):
> - **Category Identification** — 2: exact issue type match / 1: adjacent match / 0: wrong category
> - **Grounding** — 2: all claims verifiable / 1: one minor unverifiable claim / 0: hallucination present
> - **Escalation Handling** — 2: correct decision / 1: incomplete escalation / 0: missed or unnecessary escalation
>
> Verdict: all 2s → **pass** (auto-send) · any 1, no 0s → **review** (human queue) · any 0 → **fail** (immediate escalation). A score of 0 in any dimension forces **fail** regardless of others.

---

## Features

- **Multi-Step Classification** — Categorizes emails into Setup, Security, Pricing, Billing, Legal, Sales, or HR.
- **RAG Integration** — Uses Supabase Vector Store and Gemini Embeddings for factual retrieval.
- **LLM-as-Judge Evaluation** — Gemini 2.5 Flash judge scores replies across three independent dimensions (category, grounding, escalation) before sending them to customers.
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
       ├─→ pass   → [Send Email]
       ├─→ review → [Human Queue]
       └─→ fail   → [Human Escalation]

[Daily Schedule] 
       ↓
[Sync Sheets to Supabase]
```

**Tech stack:**

| Technology         | Role             | Why chosen                                           |
| ------------------ | ---------------- | ---------------------------------------------------- |
| n8n                | Orchestration    | Low-code flexibility with powerful error handling.   |
| GPT-4o Mini        | Classifier/Agent | High performance-to-cost ratio for routine tasks.    |
| Gemini 2.5 Flash   | Judge LLM        | Stronger model for more accurate multi-dimension evaluation. |
| Gemini Embedding 2 | Vectorization    | Superior semantic search capabilities.               |
| Supabase           | Vector Store     | Scalable, open-source Postgres-based vector search.  |
| Google Sheets      | Knowledge Base   | Easy for non-technical staff to update product data. |

---

## Architecture Decisions

### LLM-as-Judge Pattern

**Context:** AI agents can sometimes ignore system prompts or fabricate data (hallucinations). A single layer of validation is insufficient for production use.

**Decision:** Implement a separate LLM node specifically to grade the output of the first agent against the knowledge base before delivery. The judge evaluates each reply across three independent dimensions and returns a structured verdict.

**Rubric:**

| Dimension | 2 — Full credit | 1 — Partial credit | 0 — Fail |
|---|---|---|---|
| **Category Identification** | Exact issue type addressed (e.g. billing dispute, account compromise) | Related but not exact category (e.g. billing-account overlap treated as purely billing) | Wrong category or core problem ignored |
| **Grounding** | Every policy, price, timeline, and instruction is plausible; nothing fabricated | One unverifiable claim that is not actively harmful or misleading | Fabricated policy, invented price, false timeline, or harmful instruction |
| **Escalation Handling** | Complex/sensitive issues escalated; routine issues handled without unnecessary escalation | Escalation attempted but insufficient (no clear path, or after problematic self-service advice) | Required escalation missed entirely, or unnecessary escalation on a routine request |

**Verdict logic:**

| Condition | Verdict | Action |
|---|---|---|
| All three dimensions score 2 | **pass** | Auto-send reply |
| Any dimension scores 1, none score 0 | **review** | Route to human agent queue |
| Any dimension scores 0 | **fail** | Immediate escalation |

> A `grounding`, `escalation`, or `category` score of 0 always forces **fail**, regardless of the other scores.

**Alternatives considered:**
- Single agent with stronger system prompt — insufficient isolation of concerns.
- Manual human review of every response — bottleneck and not cost efficient.

**Trade-offs:**
- ✅ Massive reduction in incorrect replies.
- ✅ Automated auditing of performance with granular per-dimension scores.
- ✅ Three-way routing prevents borderline replies from being auto-sent or silently dropped.
- ⚠️ Increased token cost per email (~3 API calls per ticket).
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
5. Send the email, route to human review queue, or escalate — based on the judge verdict.

---

## Configuration

| Variable          | Required | Description                              |
| ----------------- | -------- | ---------------------------------------- |
| `openai_api_key`  | ✅        | Used for classification and judgment.    |
| `supabase_url`    | ✅        | Vector database endpoint.                |
| `spreadsheet_id`  | ✅        | Google Sheets ID for the knowledge base. |
| `judge_threshold` | ❌        | Override verdict threshold: minimum score per dimension to count as "pass" (default: all must be 2). |

---

## Deployment

The workflow runs as an active trigger on n8n Cloud or self-hosted. Configure via environment variables or n8n Secrets. For high volume, enable **Queue Mode** to prevent blocking on API latency.
