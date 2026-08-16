# System Design Notes

Write-ups of systems I've designed and built in production, generalized so
they're shareable without any company-specific or proprietary details —
the problem, the architecture, and the reasoning behind each decision.

Each one follows the same shape: **requirements → high-level design → deep
dives** into the parts that were actually hard, with the trade-offs made
explicit instead of glossed over. No implementation code here — these are
design write-ups, closer to an interview answer key than a codebase.

## Case studies

| # | System | What it's about |
|---|--------|------------------|
| 01 | [Real-Time Change-Data-Capture Monitoring Pipeline](case-studies/01-real-time-audit-pipeline.md) | Event-driven pipeline for real-time state-change auditing with zero data loss |
| 02 | [Promotional Override System with Guardrails](case-studies/02-promotional-override-guardrails.md) | Rules engine for safely bypassing automated guardrails, with abuse protection and auto-expiration |
| 03 | [Multi-Tenant AI Agent Platform with Shared Tooling](case-studies/03-multi-tenant-ai-agent-platform.md) | Shared platform letting many teams run isolated AI assistants on common infrastructure |
| 04 | [Automated Financial Ledger Reconciliation System](case-studies/04-automated-ledger-reconciliation.md) | Idempotent, self-correcting accounting automation for third-party-funded transactions |
| 05 | [Phased, Reversible Deprecation Framework](case-studies/05-phased-deprecation-framework.md) | Safely retiring a widely-consumed computed signal with unknown downstream consumers |

## Why these are written this way

Real production systems are built inside a specific company, with specific
internal tools and specific names. None of that is reproduced here — every
case study describes the pattern, not the company, so it's safe to share
and (mostly) transferable to any stack.

## About

Backend / distributed systems engineer. More at my portfolio site and on
[LinkedIn](https://linkedin.com/in/sreekar-manchukonda).
