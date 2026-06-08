# Loghub-SRE — Benchmark Gap Analysis & Cross-Benchmark Comparison

**Date:** 2026-06-08
**Scope:** Full pass over the committed benchmark (180 tasks), its oracle/grader
design, and a comparison against peer agent benchmarks to answer one question:
**do we need to ship tools / an MCP server like other benchmarks, or not?**

This is an analysis document. It changes no task. Every claim below is grounded
in a file in this repo; paths are cited inline.

---

## 0. TL;DR

1. **Inventory drift.** The repo has **180 tasks**, the README headlines **160**.
   The entire **20-task `rem-*` remediation family (T6)** is undocumented in the
   README and absent from every leaderboard/agent run.
2. **The `rem-*` (outcome-oriented) family is largely gameable.** Of its 11
   equally-weighted checks, **10 are obtainable without reading a single log
   line** — because the answer is leaked in `topology.json`, the "5 allowed
   mitigations" collapse to a single constant (`restart_component` in 20/20
   tasks), the config-drift scaffolding is inert (config == known_good in 20/20),
   and the evidence check accepts any real log line. The oracle/nop gate does not
   catch this because it only validates the two endpoints.
3. **Grader/oracle elsewhere is sound** and was actively hardened (fp Gap 1, corr
   Gap 6, seq trigger-ordering). v1/v2 are in good shape; the new T6 family is not.
4. **Tools/MCP verdict:** **No MCP server is needed.** This is an
   *investigation + structured-answer* benchmark, which sits in the
   SWE-bench / Terminal-Bench "shell + pytest-on-final-state" family, **not** the
   τ-bench / MCP-Universe "ship the agent function-calling tools" family. The
   `rem-*` family already has the *right* tool pattern — **in-container CLI
   binaries** (`/app/bin/check_health`, `/app/bin/apply_mitigation`) — which is
   exactly how Terminal-Bench delivers tools. Reach for Harbor's per-task MCP
   support only if you deliberately pivot to measuring MCP/tool-orchestration as
   the skill, which would be a different benchmark.

---

## 1. What the benchmark is (structure recap)

A Harbor-compatible benchmark for SRE log investigation. **180 tasks** across 5
Loghub datasets (HDFS, Hadoop, BGL, Thunderbird, OpenStack) and **7 skill
families**:

| Family | Prefix | Count | Skill | Answer schema | Status |
|---|---|---|---|---|---|
| Anomaly localization | (bare) | 60 | find+classify the anomaly | `…-answer-v2` | documented, agent-run |
| False-positive triage | `fp-` | 25 | discriminate noise vs incident | `…-v2-fp` | documented, hardened (Gap 1) |
| Temporal sequence | `seq-` | 20 | order events + trigger role | `…-v2-seq` | documented, hardened |
| Cross-component correlation | `corr-` | 20 | causal chain across files | `…-v2-corr` | documented, hardened (Gap 6) |
| Severity classification | `sev-` | 15 | P0–P3 calibration | `…-v2-sev` | documented |
| Log-template extraction | `tmpl-` | 20 | LogParser-style partition | `…-v2-tmpl` | documented |
| **Remediation (outcome)** | **`rem-`** | **20** | **diagnose → mitigate → recover** | **`…-v3-remediation`** | **UNDOCUMENTED, never agent-run** |

**Task layout** (`docs/repo-map.md`): each `tasks/<slug>/` has `instruction.md`,
`task.toml` (Harbor metadata), `environment/Dockerfile`+`data/`, a hidden
`solution/` (oracle: `solve.sh` + `derive_answer.py` + `oracle_hints.json`), and
a hidden `tests/` (`test.sh` + `test_state.py` + `expected.json`).

**Grader.** `tests/test.sh` runs pytest, converts CTRF results to a **fractional
reward** `passed / non_skipped` written to `/logs/verifier/reward.txt`
(`docs/scoring.md`). **Oracle.** `solution/solve.sh` reconstructs the answer from
visible logs + `oracle_hints.json` (coordinates only — not the gold answer), so
the oracle can't `cat` the ground truth. **Gate.** `make oracle-nop` asserts
oracle = 1.0 and nop = 0.0 on every task; `make static` runs 12 structural
checks; `ci_checks/check-oracle-leak.sh` checks no `expected.json` leaf string
leaks into `environment/`.

This is a genuinely well-engineered substrate. The gaps below are about
**discriminative power** (does a high score require the skill?), not about the
plumbing.

---

## 2. Gaps

Severity legend: 🔴 critical (a high score is achievable without the skill) ·
🟠 important · 🟡 minor.

### 🔴 G1 — The `rem-*` remediation family is largely gameable

This is the newest and most ambitious family (the "outcome-oriented" pivot in
`docs/HARDENING_REDO_REPORT.md`). Three shipped-data facts undercut it. All were
verified across **all 20** `rem-*` tasks.

**G1a — `topology.json` leaks the root component (20/20).**
`environment/data/topology.json` contains `"root_component": "hdfs-namenode"`
verbatim, and the instruction explicitly points the agent at it
("a component dependency map at `/app/topology.json`"). The instruction *also*
asks the agent to "identify the root-cause component (the one whose failure is
upstream)" — but the answer is in a file it's told to read. So
`test_root_component_matches` and the target half of
`test_mitigation_target_matches` are solved by `jq .root_component topology.json`.
This directly contradicts the HARDENING report's own quality claim #1
("Instruction not answer-leaking … agent must derive the root component").

**G1b — the "5 allowed mitigations" are a constant.** Expected
`mitigation.action` is **`restart_component` in all 20 tasks** (verified across
every `rem-*/tests/expected.json`). The enum
`{restart_component, rollback_config, increase_quota, disable_route, mark_noop}`
is theater: an agent that *always* answers `restart_component` passes the action
half of `test_mitigation_target_matches` on the entire family. The "choose the
right mitigation for the root cause" framing collapses to a single answer.

**G1c — the outcome signal doesn't discriminate the action.**
`bin/apply_mitigation` sets every component to `healthy` for **any** non-`noop`
action as long as `target == root` (see the `else` branch in
`environment/data/bin/apply_mitigation`). So restart / rollback / quota /
disable_route are interchangeable for the post-mitigation health check
(`test_post_mitigation_state`, the highest-weighted check). The environment has
no action-specific physics, so the "outcome-oriented" reward is decoupled from
action correctness — correctness is judged only by the string-equality in G1b.

**G1d — config-drift scaffolding is inert (20/20).** `config/<c>.json` is
**byte-identical** to `config/<c>.known_good.json` in every component of every
rem task. The HARDENING report claims "current config (broken for
rollback_config root causes)" — false for the shipped set. There is no drift to
detect and `rollback_config` is never correct, so the config files carry zero
diagnostic signal.

**G1e — evidence check accepts any real line.**
`test_causal_chain_evidence_real` only asserts the cited `snippet` appears at the
cited line of the cited file (`snippet in actual`). It does **not** check the
line equals the ground-truth `evidence_line`. So `grep`-ing any real log line
satisfies it.

**Net effect.** Of the 11 equally-weighted rem checks, only
`test_root_cause_matches` (#4) genuinely requires diagnosis from logs. A
"topology-reader" strategy — read `topology.json`, cite any 2 real log lines,
run `apply_mitigation --action restart_component --target <root>` — passes
roughly **10 of 11** checks (~0.9 reward) **with no investigation**. The
oracle/nop gate misses this entirely because it only checks that the oracle hits
1.0 and nop hits 0.0; it never probes the middle. This is the same class of bug
the team already caught and fixed for `fp` (Gap 1) and `corr` (Gap 6) — it just
hasn't been applied to T6, which has never been run against a real agent.

### 🟠 G2 — Inventory & documentation drift (160 vs 180)

`README.md` headlines **160 tasks (60 v1 + 100 v2)** throughout ("What's in the
box", repo layout, install). `docs/repo-map.md` still says "60 published tasks".
The 20 `rem-*` tasks exist, pass `make validate-all` (the HARDENING report
confirms 180×12 static + 180 oracle/nop green), and are absent from all
user-facing docs and the leaderboard. A reader cloning the repo gets 20 tasks
they were never told about, in a family that's the least validated.

### 🟠 G3 — No real-agent signal on 11% of the suite

The only leaderboard entry (DeepSeek V4-flash, `docs/REPORT_DEEPSEEK_V4_FLASH.md`)
ran the **160** v1+v2 tasks. The 20 `rem-*` tasks have **never been run against a
real agent** — only oracle (1.0) and nop (0.0). Given G1, the first real-agent
run will almost certainly expose the gameability. This should happen *before* the
family is documented as a headline feature.

### 🟠 G4 — Evaluation methodology gaps (the report's own open Gaps 2–5)

`docs/REPORT_DEEPSEEK_V4_FLASH.md` documents these and they remain open:
- **n = 1, no variance/SEM.** The leaderboard reports a single number per task;
  Harbor's parity protocol wants n ≥ 2 (recommends ≥ 3). Not externally citable
  as-is.
- **Single model × single harness** (DeepSeek × mini-swe-agent) — no comparison
  axis to separate model capability from harness behavior.
- **Token/cost not captured** (mini-swe-agent reports $0.00 for DeepSeek;
  real cost only from the provider dashboard).

These are honestly disclosed already; flagging for completeness.

### 🟡 G5 — Known, defensible weaknesses (documented tradeoffs, not bugs)

- **BGL/Thunderbird inline-label validation** accepts any cited line whose inline
  alert tag maps to the expected root cause, instead of exact `(file, line)`
  (`docs/scoring.md`). Reasonable for huge slices, but evidence *precision* is
  not tested for 2 of 5 datasets.
- **`recommended_action` / safe-action checks** are closed-set membership, an
  acknowledged proxy ("agent didn't invent `rm -rf /`"), not a judgment measure.
- **No leaderboard helper** — aggregation is manual via `harbor view` / per-job
  `result.json` (`docs/scoring.md` says so explicitly). `tools/analysis/*`
  exists (summarize/diff/failure-modes/quality_report) but isn't wired into a
  published path.

---

## 3. Cross-benchmark comparison

How peer agent benchmarks handle the four axes. (Sources at the end.)

| Benchmark | Agent interface | Oracle | Grader | Tools / MCP? |
|---|---|---|---|---|
| **Loghub-SRE (this)** | **Shell** in container; rem family adds **in-container CLI tools** | Gold **answer JSON** (+ rem: gold final **state**) via hidden `expected.json`; `solution/` reconstructs from visible data | **pytest on final answer/state** → fractional reward | **CLI binaries** in `rem-*`; no MCP |
| **Terminal-Bench / Harbor** | Shell (tmux); **per-task MCP opt-in** via `[[environment.mcp_servers]]` | Gold `solution.sh` (solvability oracle) | pytest on final container state | Shell default; **MCP available, opt-in** |
| **SWE-bench / Verified** | Repo + shell | Gold **patch** (PR diff) | **Test execution** (FAIL_TO_PASS + PASS_TO_PASS) | None |
| **τ-bench / τ²-bench** (Sierra) | **Function-calling domain tools** + LLM user simulator | Gold final **DB state** (+ golden tool-call sequence) | **Env-state comparison** (DB hash) | **Yes — tools are the point** |
| **MCP-Bench / MCP-Universe / MCPEval** | **Real running MCP servers** | Rubric / ground-truth per task | LLM-judge (MCP-Bench) → ground-truth verifiers (MCP-Universe) | **Yes — MCP is the point** |
| **GAIA** | Open-ended tool use | Single target **answer string** | Exact-match string | Tools used, not graded |
| **WebArena** | Browser / web env | Per-task target | Hybrid: answer-match + **final site state** | Env as the "tools" |

**The spectrum.** Benchmarks ship structured tools/MCP when the domain is
**interactive and stateful** and correctness is defined by the resulting
world/DB state (τ-bench transactions, MCP orchestration, WebArena). They give
**shell/repo-only** when the task is **self-contained investigation or coding**,
where correctness is captured by deterministic tests or a target answer/state
inside the container (SWE-bench, Terminal-Bench, GAIA).

**Loghub-SRE sits squarely on the shell/investigation side** — same family as
SWE-bench and Terminal-Bench. Its grader (pytest → fractional reward) and oracle
(reconstruct-from-visible) are textbook for that family.

---

## 4. Do we need tools / an MCP server?

**Recommendation: No MCP server. Keep shell-only for v1/v2; keep the
in-container CLI-tool pattern for the outcome family (after hardening).**

**Why not MCP / function-calling tools:**

1. **It would change (and narrow) the skill under test.** The skill here is
   *investigating raw logs with general-purpose tools* (`grep`/`awk`/`python`).
   Handing the agent a bespoke "log-query" tool replaces "can you find the
   anomaly in 5 partitioned files" with "can you call our API" — and *removes the
   retrieval difficulty that is the point*. The DeepSeek failure modes that make
   this benchmark interesting — `test_no_cross_file_line_confusion` (47 tasks),
   over-citation outside ground truth (47 tasks) — are exactly the
   raw-retrieval skills a structured tool would paper over.
2. **Tools ≠ MCP.** A benchmark "has tools" the moment it puts an executable in
   the container. The `rem-*` family already does this correctly with
   `/app/bin/check_health` and `/app/bin/apply_mitigation` — the **same pattern
   Terminal-Bench uses**. This is deterministic, offline, language-agnostic, and
   requires zero MCP plumbing. You already have the right tool model.
3. **MCP only earns its keep for a different goal.** Harbor *does* support
   per-task MCP servers (`[[environment.mcp_servers]]`). That's worth it only if
   you specifically want to (a) **measure the agent's MCP-protocol tool-use
   ability**, or (b) expose something awkward as a CLI (a stateful session, a
   streaming/long-poll source). Neither is true for log investigation. Adding MCP
   would buy operational complexity (a server process per task, protocol surface,
   nondeterminism risk) for no measurement gain.
4. **Peer benchmarks in our family don't.** SWE-bench and Terminal-Bench — the
   two most-cited benchmarks of the "agent investigates a self-contained
   environment" type — are shell-only. We'd be the odd one out for no reason.

**When the answer would flip to "yes, add a richer tool surface" (still CLI, not
necessarily MCP):** if you deliberately grow the `rem-*` family into a
τ-bench-style "operate a live system" benchmark — where the agent runs a sequence
of *consequential, action-specific* operations and the reward is the genuine
end-state. That direction is good and the T6 family already points at it. But the
right move is to **make the existing CLI tools real** (see §5), not to bolt on
MCP. Only introduce MCP if you want the tool-orchestration *protocol* itself to
be the measured skill.

---

## 5. Prioritized recommendations

**P0 — Fix or quarantine the `rem-*` family before documenting it.**
   - Stop leaking the answer: remove `root_component` from the agent-visible
     `topology.json` (keep it only in hidden `expected.json` / `oracle_hints`);
     let the agent derive root from `depends_on` topology + log evidence.
   - Make the action matter: vary the gold mitigation across tasks (the enum
     already exists), and give `apply_mitigation` **action-specific physics** so
     the *wrong* allowed action leaves the cluster degraded — then the
     highest-weighted post-state check actually discriminates, matching the
     τ-bench state-reward model.
   - Make config drift real for `rollback_config` tasks (`config != known_good`),
     or drop the config scaffolding.
   - Tighten `test_causal_chain_evidence_real` to require the GT line (as the v1
     `test_no_cross_file_line_confusion` already does).
   - Add a "partial-credit floor" probe to the gate: a cheap heuristic agent
     (read sidecars only, no logs) should score **low**, not ~0.9. The
     oracle/nop endpoints don't catch mid-range gameability.

**P1 — Reconcile the count.** Update `README.md` and `docs/repo-map.md` to 180
   (or explicitly scope the public set to 160 and move `rem-*` to an
   experimental tier). Document T6 in the README task table.

**P2 — Run a real agent on `rem-*`** and add it to the leaderboard. Expect the
   first run to confirm G1; use it to drive the P0 fixes.

**P3 — Close the methodology gaps** (report Gaps 2–5): n ≥ 3 for SEM, a second
   model and/or harness, captured token/cost. Wire `tools/analysis/quality_report.py`
   into a published leaderboard path.

**P4 — Tools/MCP:** no MCP. Keep shell-only for investigation families; invest in
   making the `rem-*` CLI tools genuinely consequential (P0) rather than adding a
   protocol layer.

---

## Sources (cross-benchmark)

- Terminal-Bench: tbench.ai/docs/task-overview · github.com/laude-institute/terminal-bench
- Harbor per-task MCP (`[[environment.mcp_servers]]`): deepwiki.com/laude-institute/harbor (MCP server integration)
- SWE-bench Verified: huggingface.co/datasets/SWE-bench/SWE-bench_Verified · openai.com/index/introducing-swe-bench-verified
- τ-bench / τ²-bench: arXiv:2406.12045 · github.com/sierra-research/tau2-bench
- MCP benchmarks: MCP-Bench arXiv:2508.20453 (github.com/Accenture/mcp-bench); MCP-Universe / MCPEval (survey arXiv:2602.00933)
- GAIA / WebArena / AgentBench: benchmarkingagents.com/agent-benchmarks · evidentlyai.com/blog/ai-agent-benchmarks
