# Architecture

## Overview
SeniorDev analyzes each pull request for security vulnerabilities and performance regressions, explains findings with cited evidence, and verifies every proposed fix in a sandbox before showing it to the developer.

## Components
- Repository Indexing: diff parser, AST (tree-sitter), call graph and data-flow, metadata
- Performance Analysis: complexity classifier, regression detector
- Security Analysis: Semgrep, reachability filter, vulnerability classifier, CWE/CVE mapping
- RAG Agent: planner, retrieval, evidence check, generator, confidence
- Verification Sandbox: Docker, runtime enforcer, test generator, executor, metrics
- Dashboard and CI gate

## Data flow
See docs/dry-run-01-sql-injection.md for one finding followed through every module.

## Shared database
One PostgreSQL instance with pgvector: relational tables (code graph, findings, verdicts, artifacts) and vector columns (code and CWE/CVE embeddings).
