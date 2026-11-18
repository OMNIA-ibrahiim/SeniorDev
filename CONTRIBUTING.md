# Contributing

How we work. Everyone follows the same routine every time, so the code stays in sync and `dev` always works.

## Branches

| Branch | Purpose | Who pushes |
|---|---|---|
| `main` | Stable and demo-ready. Updated only at milestones (seminars, final) | Leader, by PR from `dev` |
| `dev` | Integration. Everything is merged here | Nobody directly, only by PR |
| `module/indexing` | Repository Indexing | Indexing pair |
| `module/performance` | Performance Analysis | Performance pair |
| `module/security` | Security Analysis | Security pair |
| `module/rag-sandbox` | RAG Agent and Verification Sandbox | RAG and Sandbox pair |

Dashboard work (owner to be decided): create `feat/dashboard-<topic>` from `dev`.

In the commands below, replace `module/security` with your own branch.

---

## The routine

### 1. Start of every session: get fully up to date

```powershell
git checkout module/security
git pull origin module/security
git pull origin dev
```

- The first pull gets your partner's work. The second gets everyone else's merged work.
- Commit or stash your own changes first.
- If a conflict appears, fix it now, before writing new code (see "If you get a conflict").

### 2. Move your card on the board

Put your issue in **In progress** and assign yourself.

### 3. Code, and commit small and often

```powershell
git add .
git commit -m "feat(security): add semgrep runner"
```

Message format: `type(module): what you did`

- Types: `feat` (new), `fix` (bug), `docs`, `test`, `chore`
- Example: `fix(sandbox): kill container on timeout`

Commit after every small piece that works, not once a day.

### 4. Before every push

```powershell
git status
```

Check the list. None of these may appear: `.env`, datasets, model weights, secrets.

Also:

- Run your tests. They must pass.
- Added a library? Add it to `modules/<your-module>/requirements.txt`.
- Changed how to run it? Update your module `README.md`.

Then push:

```powershell
git pull origin module/security
git push origin module/security
```

Push at the end of every session, so work is never only on your laptop.

### 5. When a piece is finished: open a pull request into `dev`

1. On GitHub, click **Compare and pull request**. The base must be **dev**.
2. Write a clear title: `feat(security): add semgrep runner`.
3. Fill in the PR template:

   - **What does this PR do?** Two or three sentences: what you built and what it returns or changes.
   - **Related issue:** `Closes #23` (use the real issue number, so the card moves to Done when it merges).
   - **Checklist:** tick only what is true:
     - Code runs locally
     - Tests added or updated
     - Docs updated
     - Module contract in `docs/contracts/` updated (if the interface changed)
     - No secrets or `.env` committed

4. On the right side, set:
   - **Labels:** your `module:*` label
   - **Milestone:** the same as the issue
   - **Reviewer:** someone from **another pair**
5. Move your card to **In review**.
6. Wait for CI to go green and for approval.
7. Merge with **Create a merge commit**. Do not squash, and do not delete the branch.

Example description:

```markdown
## What does this PR do?
Adds the Semgrep runner: runs Semgrep on the changed files of a PR and
returns findings with rule_id and a location object.

## Related issue
Closes #23

## Checklist
- [x] Code runs locally
- [x] Tests added or updated
- [x] Docs updated
- [x] Module contract in docs/contracts/ updated (if the interface changed)
- [x] No secrets or .env committed
```

PR rules:

- Keep it small enough to review in about 15 minutes.
- Nobody merges their own PR before it is approved.
- Do not tick a box that is not true. If a test is missing, add it.

### 6. After the merge

```powershell
git checkout module/security
git pull origin dev
```

Tell your partner and the other pairs in the group chat if your change affects them.

---

## If you get a conflict

When `git pull origin dev` says `CONFLICT`:

1. Do not panic, and do not force-push.
2. Run `git status` to see the conflicted files.
3. Open each file and find the `<<<<<<<`, `=======` and `>>>>>>>` markers. Keep the right code from both sides, and delete the markers.
4. Run:
   ```powershell
   git add .
   git commit -m "chore: merge dev into module/security"
   ```
5. Run your tests again.
6. Not sure which code is right? Ask the owner of that file or the leader before deleting anything.

---

## Easy-to-forget list

- **Contract changed?** Update `docs/contracts/` in the same PR, and tell the pairs that use it.
- **Shared types** (`modules/shared/`) need approval from **all** module owners.
- **Never commit** `.env`, datasets, model weights or passwords. Datasets and models go in `data/`, which git ignores.
- **Large files:** GitHub rejects anything over 100 MB.
- **Do not edit another pair's folder.** Open an issue or ask them.
- **Decisions** (tool choices, rules) go in `docs/adr/`, not only in chat.
- **Found a bug in someone else's module?** Open a Bug issue.
- **Weekly:** make sure your work has been merged into `dev` at least once a week.

## Local setup

```powershell
cp .env.example .env
docker compose up -d
```

Docker Desktop must be running first.

## Rules

- Never push directly to `main` or `dev`.
- Never commit `.env` or secrets.
- Merge small pieces often, not everything at the end.

---

## Quick reference (commands only)

**1. Start of session**
```powershell
git checkout module/security
git pull origin module/security
git pull origin dev
```

**2. Board:** move your card to In progress and assign yourself.

**3. Code and commit**
```powershell
git add .
git commit -m "feat(security): what you did"
```

**4. Before pushing**
```powershell
git status
git pull origin module/security
git push origin module/security
```
Check `git status` for secrets and data files, and run the tests first.

**5. Pull request into dev**
1. Open a PR from your branch into `dev`.
2. Fill in the PR template: what it does, `Closes #<issue number>`, and tick the checklist honestly.
3. Set the label, the milestone and a reviewer from another pair.
4. Move the card to In review.
5. Wait for green CI and one approval.
6. Merge with "Create a merge commit". Do not squash. Do not delete the branch.

**6. After the merge**
```powershell
git checkout module/security
git pull origin dev
```