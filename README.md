# ai-negotiation-platform
An autonomous multi-agent negotiation and settlement platform with private mandates, database-enforced guardrails, and auditable decision-making.

## Project Overview

The AI Negotiation Platform is an autonomous multi-agent system designed to facilitate secure B2B negotiations between independent Buyer and Seller AI agents.

The platform uses three specialized agents:

- **Buyer Agent:** Negotiates on behalf of the buyer while respecting its private budget and purchasing constraints.
- **Seller Agent:** Negotiates on behalf of the seller while following its private minimum price and selling conditions.
- **Neutral Arbiter:** Helps resolve negotiation deadlocks through structured arbitration and human escalation when necessary.

### The Core Problem

AI agents can generate proposals that violate business rules, reveal sensitive information, or attempt unauthorized actions.

Traditional prompt-based restrictions alone are not sufficient to enforce critical business constraints.

### The Solution

This platform places PostgreSQL at the center of decision enforcement.

- Private negotiation mandates are stored in the database.
- Every proposed action is validated against business constraints.
- Settlement decisions are transactionally enforced by PostgreSQL.
- n8n orchestrates the workflows.
- Groq provides the runtime LLM.

The result is a controlled negotiation system where AI agents can negotiate autonomously, but cannot independently override binding business rules.

### Key Features

- Independent Buyer and Seller AI agents
- Private negotiation mandates
- Database-enforced business constraints
- Transaction-safe settlement
- Neutral arbitration and human escalation
- Idempotency and duplicate-action protection
- Auditable negotiation decisions

### Technology Stack

- **AI Orchestration:** n8n
- **Runtime LLM:** Groq (`openai/gpt-oss-120b`)
- **Database:** Supabase PostgreSQL 17
- **Backend Enforcement:** PostgreSQL Functions



## Demo Video

[▶ Watch the Full Project Demo on YouTube](https://youtu.be/CJDLmhsynDA)

This 4.5-minute demo walks through the multi-agent negotiation workflow, database-enforced guardrails, security controls, and settlement process.
