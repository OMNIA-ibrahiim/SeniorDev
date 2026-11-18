# Dry-run 01: one SQL-injection finding, end to end

**Status:** Draft for team review
**Sources:** GP27 project proposal + Seminar 1 slides (module diagrams on slides 11, 14, 17, 20)
**Goal:** Follow one finding through every module on paper, write down every place where one module's output does not match the next module's input, and fix it before anyone writes code.

> Not reviewed: I only read the text of slides 9, 15, 18, 21, 22 and 25, not their images. If those hold interface details, check them against this document.

---

## 1. The finding we trace

Pull request **#142** on `acme/shop-api` changes `UserDao.java`. Line 42 builds an SQL string by concatenating user input.

Expected journey:

```
GitHub webhook -> Indexing -> Security -> Sandbox (is it real?) -> RAG (explain + fix)
   -> Sandbox (does the fix work?) -> RAG (final answer) -> Dashboard / CI gate
```

---

## 2. Hop-by-hop trace

| # | Hop | What the sender produces today | What the receiver needs | Result |
|---|-----|-------------------------------|-------------------------|--------|
| H1 | Webhook -> Indexing | Not defined anywhere | repo, PR number, base and head commit, the diff | **Gap (M14)** |
| H2 | Indexing -> Postgres | "code graph" (no table layout) | Tables Security, Performance and RAG can all query | **Gap (M14)** |
| H3 | Indexing -> Security | `pr`, `repo`, `diff: "142.diff"`, `changed_files` | Which graph version to read, which symbols changed | **Mismatch (M1, M5, M13)** |
| H4 | Security -> Sandbox | `type`, `cwe`, `location: "UserDao.java:42"`, `severity`, `vulnerable`, `confidence` | A finding id, the Semgrep rule, a precise location, a defined job type | **Mismatch (M1, M3, M5, M8)** |
| H5 | Sandbox -> RAG | `verdict`, `tests_before/after`, `semgrep`, `evidence` (only shown for patches) | A verdict on the *finding* (confirmed or false positive) plus evidence | **Gap (M3, M10)** |
| H6 | RAG input shape | `pillar`, `type`, `cwe`, `location` | Same shape for performance findings; `pillar` has no producer | **Mismatch (M6)** |
| H7 | RAG -> Sandbox | `candidate_fix: "PreparedStatement"`, `required_tests: ["payload rejected"]` | A real diff and a runnable test | **Mismatch (M2, M4)** |
| H8 | Sandbox -> RAG | `verdict: "verified"`, counts as strings | Structured result, plus perf metrics and overfitting-guard result | **Mismatch (M10)** |
| H9 | RAG -> Dashboard | `explanation`, `citations`, `candidate_fix`, `confidence` | Final status, severity, verdict, merge-gate inputs | **No contract (M7)** |

---

## 3. Mismatches

### Blockers (the hand-off cannot work as written)

**M1. No `finding_id` anywhere.**
Sandbox input says `"finding": "CWE-89"`. A CWE is a category, not an identifier. Two SQL injections in one PR would be indistinguishable, and there is no way to attach a verdict, a patch or a citation to one specific finding.
*Fix:* Security mints `finding_id` (e.g. `fnd_142_0007`). Every later message carries it. `run_id` identifies one analysis run, because PR 142 can be pushed to several times.

**M2. RAG's fix is a description, the Sandbox needs a diff.**
RAG outputs `candidate_fix: "PreparedStatement"`. Sandbox input expects `patch: "fix-142.diff"`. Nobody is responsible for turning the first into the second.
*Fix:* RAG outputs a unified diff (`patch.diff`) plus a short `fix_summary`. Producing the diff is part of the Generator step.

**M3. Verdict vocabulary and job types are inconsistent.**
The proposal says findings are `confirmed / false-positive` and fixes are `verified / failed / uncertain`. The slide shows only `"verified"`, and only one job type, `verify_patch`. There is no defined job for "is this finding real?".
*Fix:* Two job types (`confirm_finding`, `verify_patch`) and two separate enums (section 5).

**M4. Who writes the tests, and is it AI?**
- Proposal: the sandbox is "deterministic, rule-based, explicitly non-AI", and uses pytest and Python profiling (`tracemalloc`, `time.perf_counter`).
- Slides: a "Test Generator" writes **JUnit** tests for each finding.
- The MVP language is Java, so pytest and Python profilers cannot run the target code.
- RAG's `required_tests: ["payload rejected"]` is free text, so a machine cannot act on it.

*Proposed fix:* keep the "no AI" claim honest by making the Test Generator **template-based**. There is one test template per CWE (for CWE-89: send a fixed list of injection payloads, assert none change query behavior). RAG's `required_tests` becomes a list of IDs from that catalog (e.g. `sqli.payload_rejected`). Profiling for Java uses JMH or plain `System.nanoTime` harnesses, not Python tools. Update the proposal wording accordingly.

**M5. `location` is a `"file:line"` string with no commit.**
Line 42 moves once the patch is applied, so "same place" cannot be checked before and after. It also does not link to a node in the call graph.
*Fix:* a structured `location` object with file, line range, `symbol`, `symbol_id` (graph node), and the `head_sha` it refers to.

**M6. Security and Performance findings have different shapes, and RAG only fits one.**
- Security output: `type`, `cwe`, `location`, `severity`, `vulnerable` (boolean), `confidence`.
- Performance output: `method`, `before`, `after`, `regression` (`"yes"`), `cost_driver`. No location, no severity, no confidence.
- RAG input has `pillar`, `type`, `cwe`, `location`. Neither producer sets `pillar`, and a performance finding has nowhere to put `before/after`.

*Fix:* one shared `Finding` envelope with a `pillar` field and a pillar-specific `details` block (section 5).

**M7. Dashboard and CI gate have no contract and no owner.**
The proposal's architecture has module 6 (Unified Findings View, Merge Gate Decision Logic). The slides have no diagram for it, and the team table in section 8 of the proposal assigns nobody to it, nor to the GitHub App/webhook or the FastAPI layer.
*Fix:* decide an owner and write the contract (section 5, hop H9). This is a team decision, not a technical one.

### Should fix before coding

**M8. Reachability and the Semgrep rule are missing from Security's output.**
Reachability filtering is the headline novelty, yet the output has no reachability field, and the slide's four stages (Static, Preprocessing, Classification, Result) do not show a Reachability Filter. The Sandbox also re-runs Semgrep after a patch, but it is never told *which rule* fired.
*Fix:* add `rule_id` and a `reachability` object (reachable, entry point, path). Also decide whether findings dropped as unreachable are stored with status `filtered_unreachable` (needed to measure false-positive reduction) or discarded.

**M9. Two `confidence` fields, different meanings.**
Security `0.91` is classifier probability. RAG `0.87` is confidence in the explanation and fix. Both end up next to the same finding.
*Fix:* `classifier_confidence` and `fix_confidence`.

**M10. Sandbox output is not machine-usable.**
`"tests_before": "3 fail"` and `"semgrep": "clean"` are strings. Missing: `job_id`, performance measurements (slide 23 shows time at n = 1,000 to 8,000), the overfitting-guard result (the module's main contribution), and the reason for an `uncertain` verdict. Also unclear: does RAG wait for the result (synchronous) or poll for it?
*Fix:* structured counts, `metrics`, `overfit_guard`, `reason`, and an asynchronous job with `job_id` and `status` (container start plus scaled profiling will take too long for a blocking call).

**M11. Citations are untyped and RAG cannot abstain.**
`citations: ["CWE-89", "UserDao.java:42"]` mixes two kinds of reference, which makes "citation validation" (a proposal deliverable) hard. The output has no way to say "insufficient evidence", which the proposal promises.
*Fix:* typed citations and `status: answered | abstained`.

**M12. Inconsistent value types.**
`vulnerable: true` is a boolean, `regression: "yes"` is a string. Complexity appears as `"O(n^2)"`. Pick one convention: booleans for yes/no, and a fixed list of complexity classes matching the CodeComplex labels.

**M13. Files are referenced by name only.**
`142.diff`, `fix-142.diff`, `run-142-b.log` have no stated location.
*Fix:* an `artifacts` table in Postgres (id, kind, path or blob, sha256) and pass `artifact_id` values.

**M14. Indexing has no contract, and the graph tooling is unclear.**
The slides list **Joern**; the proposal lists **tree-sitter** and "PostgreSQL tables" and never mentions Joern. Tree-sitter produces syntax trees but not data-flow, so the data-flow edges have to come from somewhere.
*Fix:* decide the tool (likely tree-sitter for AST and Joern for call and data-flow, imported into Postgres), then write `docs/contracts/indexing.md` with the table layout (section 5).

**M15. Two result channels.**
Slides show the Sandbox both returning a result to RAG *and* writing to Postgres.
*Fix:* Postgres is the single source of truth. The Sandbox writes there, and the return value is just `{job_id, status}`.

### Infrastructure issues that affect step 1

**M16. BM25 is not part of pgvector.**
Slide 17 shows "BM25 keyword index" under pgvector. pgvector only provides vector search. Options: Postgres built-in full-text search (`tsvector`, not exactly BM25), a BM25 extension (this changes the Docker image from step 1), or the `rank_bm25` Python library over stored text. The embedding column also needs a fixed dimension, which depends on `jina-code-embeddings-0.5B`. Check its model card before writing the migration.

**M17. (Same as M4, listed for the proposal.)** The proposal's deliverables say pytest and `tracemalloc`; update them to JUnit and a Java harness.

---

## 4. Decisions the team has to make

| # | Decision | Suggested answer |
|---|----------|------------------|
| D1 | Test Generator: template-based or LLM-written? | Template per CWE (keeps "no AI" true) |
| D2 | Who owns Dashboard, GitHub App/webhook, FastAPI gateway? | Assign now; nobody owns them today |
| D3 | Graph tooling: tree-sitter only, or tree-sitter + Joern? | Both, importing into Postgres |
| D4 | Are unreachable findings stored or dropped? | Stored with a status, for evaluation |
| D5 | BM25 approach | Postgres full-text search first; revisit if retrieval metrics are weak |
| D6 | Sandbox calls sync or async? | Async job + polling |

---

## 5. Corrected contracts

### 5.1 Shared types

```json
// Location
{
  "file": "src/main/java/com/acme/dao/UserDao.java",
  "start_line": 42,
  "end_line": 44,
  "symbol": "UserDao.findUser(String)",
  "symbol_id": "sym_8f21",
  "head_sha": "9c1e7ab"
}
```

```json
// Citation
{ "type": "cwe | cve | code | test_output | reference", "ref": "CWE-89", "detail": "optional line range or URL" }
```

```text
Finding status (one lifecycle, stored on the findings row):
  detected -> filtered_unreachable
  detected -> confirmed | false_positive | inconclusive          (set by confirm_finding)
  confirmed -> fix_proposed -> fix_verified | fix_failed | fix_uncertain
  confirmed -> abstained                                          (RAG had insufficient evidence)

Finding verdict enum  (confirm_finding): confirmed | false_positive | inconclusive
Patch verdict enum    (verify_patch):    verified | failed | uncertain
```

### 5.2 H1 Webhook -> Indexing

```json
{ "run_id": "run_142_1", "repo": "acme/shop-api", "pr": 142,
  "base_sha": "a13b9d0", "head_sha": "9c1e7ab", "diff_artifact_id": "art_001" }
```
Indexing writes `files`, `symbols`, `call_edges`, `dataflow_edges` (each keyed by `run_id`) and emits `{ "run_id": "run_142_1", "status": "index_ready" }`.

### 5.3 H3 Indexing -> Security (and Performance)

```json
{ "run_id": "run_142_1", "changed_symbols": ["sym_8f21"], "changed_files": ["src/main/java/com/acme/dao/UserDao.java"] }
```

### 5.4 H4 Security -> Postgres -> Sandbox (`confirm_finding`)

```json
{
  "finding_id": "fnd_142_0007",
  "run_id": "run_142_1",
  "pillar": "security",
  "status": "detected",
  "location": { "...": "see Location" },
  "severity": "high",
  "classifier_confidence": 0.91,
  "details": {
    "type": "SQL Injection",
    "cwe": "CWE-89",
    "rule_id": "java.lang.security.audit.sqli.tainted-sql-string",
    "reachability": { "reachable": true, "entry_point": "UserController.search", "path": ["sym_11a","sym_8f21"] }
  }
}
```
Sandbox request: `{ "job_type": "confirm_finding", "finding_id": "fnd_142_0007" }`. The Sandbox reads everything else from Postgres.

### 5.5 H5 Sandbox -> RAG (finding verdict)

```json
{
  "job_id": "job_0031", "finding_id": "fnd_142_0007", "job_type": "confirm_finding",
  "status": "done",
  "verdict": "confirmed",
  "tests": { "ids": ["sqli.payload_rejected"], "passed": 0, "failed": 3, "total": 3 },
  "evidence_artifact_ids": ["art_017"]
}
```

### 5.6 RAG input (same envelope for both pillars)

RAG receives the `finding_id`, reads the finding row and its evidence from Postgres. For a performance finding, `pillar` is `"performance"` and `details` holds:
```json
{ "method_symbol_id": "sym_77c", "before": "O(n)", "after": "O(n^2)", "regression": true, "cost_driver": "nested loop" }
```

### 5.7 H7 RAG -> Sandbox (`verify_patch`)

```json
{
  "finding_id": "fnd_142_0007",
  "status": "answered",
  "explanation": "User input from UserController.search reaches the SQL string at line 42.",
  "citations": [
    { "type": "cwe", "ref": "CWE-89" },
    { "type": "code", "ref": "UserDao.java", "detail": "42-44" }
  ],
  "fix_summary": "Replace string concatenation with a PreparedStatement.",
  "patch_artifact_id": "art_021",
  "required_test_ids": ["sqli.payload_rejected"],
  "fix_confidence": 0.87
}
```
If evidence is insufficient: `"status": "abstained"` and `explanation`, `patch_artifact_id` are null. Sandbox request: `{ "job_type": "verify_patch", "finding_id": "fnd_142_0007", "patch_artifact_id": "art_021" }`.

### 5.8 H8 Sandbox -> RAG (patch verdict)

```json
{
  "job_id": "job_0032", "finding_id": "fnd_142_0007", "job_type": "verify_patch",
  "status": "done",
  "verdict": "verified",
  "reason": null,
  "tests_before": { "passed": 0, "failed": 3, "total": 3 },
  "tests_after":  { "passed": 3, "failed": 0, "total": 3 },
  "semgrep": { "rule_id": "java.lang.security.audit.sqli.tainted-sql-string", "findings_after": 0 },
  "overfit_guard": { "ran": true, "passed": true, "checks": ["existing_tests_unchanged", "extra_payload_set"] },
  "metrics": { "runtime_ms_by_n": null, "peak_memory_mb": 212 },
  "evidence_artifact_ids": ["art_024"]
}
```
For a performance finding, `metrics.runtime_ms_by_n` holds the scaled-input timings (e.g. `{"1000": 10, "2000": 40, ...}`) and `reason` explains an `uncertain` verdict.

### 5.9 H9 -> Dashboard / CI gate

```json
{
  "finding_id": "fnd_142_0007", "pr": 142, "pillar": "security",
  "severity": "high", "status": "fix_verified",
  "explanation": "...", "citations": [ ... ],
  "patch_artifact_id": "art_021", "patch_verdict": "verified",
  "blocks_merge": true
}
```
Gate rule to agree on (D2): block when severity is high or above and the finding is `confirmed` and not yet `fix_verified`.

---

## 6. Next steps

1. Team review of section 4; settle D1 to D6 in one meeting.
2. Turn section 5 into one file per module in `docs/contracts/`, using `_TEMPLATE.md`. Each owner signs off on their own file. Record D1, D3 and D5 as ADRs in `docs/adr/`.
3. Update the proposal text where it now disagrees (M4, M14, M17).
4. Run a **second dry-run with a performance finding** (O(n) to O(n^2) in `findUser()`). It will test the parts this run only touched: the scaled-input profiling job and `details` for the performance pillar.
5. Create `docs/contracts/` issues for step 3 (per-module backlog) from the mismatch list above.
