# SeniorDev: AI Code Review with Automated Verification

> An AI-assisted code review system that finds security vulnerabilities and performance regressions in pull requests, explains them with cited evidence, and verifies every proposed fix by running it in an isolated sandbox.

FCIS ASU Graduation Project 2026-2027, Computer Science.

## Architecture

Diagram goes here (add docs/architecture.png).

Pipeline: Pull request -> Repository Indexing -> Security / Performance analysis -> Verification Sandbox <-> RAG Agent -> Dashboard and CI gate.

## Modules

| Module | Owner | Description |
|--------|-------|-------------|
| Repository Indexing | TBD | Parses the PR diff into ASTs, call graph and data-flow graph |
| Performance Analysis | TBD | Predicts Big-O complexity before and after a PR, flags regressions |
| Security Analysis | TBD | Semgrep + fine-tuned classifier + reachability filtering |
| RAG Agent | TBD | Cited explanations and candidate fixes from retrieved evidence |
| Verification Sandbox | TBD | Runs findings and fixes in isolated Docker containers |
| Dashboard and CI gate | TBD | Unified findings view and merge-gate decision |

## Quick start

Requires Git and Docker Desktop (running).

    git clone https://github.com/YOUR-USER/SeniorDev-AI-Code-Review-with-Automated-verifiaction.git
    cd SeniorDev-AI-Code-Review-with-Automated-verifiaction
    cp .env.example .env
    docker compose up -d
    docker compose exec db psql -U app -d seniordev

Other commands:

    docker compose down        # stop
    docker compose down -v     # stop and delete all data (fresh database)
    docker compose logs -f db  # view logs

Database init scripts only run on a fresh volume, so use `docker compose down -v` after changing `infra/db/init/`.

## Documentation

- [Architecture](docs/architecture.md)
- [Module contracts](docs/contracts/)
- [Dry-run 01: SQL injection finding](docs/dry-run-01-sql-injection.md)
- [Contributing](CONTRIBUTING.md)

## Team

| Name | GitHub |
|------|--------|
| Omnia Ibrahim  |
| Yomna Mahsoob | |
| Habiba Seif |  |
| Bassmala Abood |  |
| Sandra Hany |  |
| Mariam Mohamed |  |

Supervisors: Dr. Ahmed Salah, TA Saif AlDeen Sameh.
