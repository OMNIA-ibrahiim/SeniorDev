# Contributing

## Workflow
1. Pick an issue from the project board and assign yourself.
2. Create a branch from `main`:
   - `feat/short-description`
   - `fix/short-description`
   - `docs/short-description`
3. Commit using Conventional Commits (https://www.conventionalcommits.org):
   - `feat(security): add semgrep orchestrator`
   - `fix(sandbox): handle container timeout`
   - `docs: update architecture`
4. Open a pull request using the template. Link the issue with `Closes #12`.
5. Get at least one approval and a passing CI run.
6. Squash and merge, then delete the branch.

## Local setup

    cp .env.example .env
    docker compose up -d

## Rules
- Never push directly to `main`.
- Never commit `.env` or secrets.
- If you change a module's interface, update its file in `docs/contracts/` in the same PR.
