# Reflection Brief — Harness Engineering Capstone

**Name:** Marcin Bien
**Date:** 2026-11-17

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): `claude-haiku-4-5-20251001` for Systems 1 and 2 (recorded in `runs/20261117_085556/summary.md` and `case_facts_call.json`). System 4 ran offline from `fixtures/recorded_responses/shift_C_2026-04-30.json`. System 3 makes no API calls.
- OS / Python: macOS 26.7.2 (Darwin 25.6.0) / Python 3.13.18, one `.venv` per system.
- Approx. API spend: about $0.35. System 1 is $0.1217 per `summary.md`. System 2 is roughly $0.2, estimated from ~200k Haiku input tokens across the case-facts call (39,209), two compression calls (12,334 + 11,475), six evals (~16.9k each) and the control run. Systems 3 and 4 cost $0.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → `runs/20261117_085556/traces/claim_05_auto_collision.jsonl` shows `tool_use, tool_use, tool_use, end_turn`. Turn 1 calls `lookup_policy` plus six `record_claim_fact`. Turn 2 calls `classify_claim` and `assess_severity`. Turn 3 calls `route_to_adjuster`. Turn 4 is `end_turn`. `claims_intake/loop.py::run` decides, and the only signal is `response.stop_reason`. `"end_turn"` returns a `FinalState`, `"tool_use"` executes the tools and continues, and any other value raises `UnexpectedStopReason`. The loop is `while True` with no turn cap. `Budget.check()` is the safety net.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_no_string_membership_against_text_in_loop` uses an AST walk to reject checks like `"done" in text` in `loop.py`. Text-based termination would have broken my run. Five of the 8 claims (01, 02, 03, 06, 07) ended with `end_turn` and no terminal tool call. The final turn is only a short message, for example 3,719 in / 54 out on claim_03 turn 5. A text-sniffing loop could have read that prose as "done" or "routed". With `stop_reason` as the signal, the loop ended cleanly and `run.py` caught the missing terminal action. The sibling test `test_no_integer_literal_iteration_cap_in_loop` rejects a `while turn < N` cap, which would cut off claim_03's 5-turn clarification flow if N were small.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `route_to_adjuster` and `escalate_to_human` both take a summary of the same claim. Each description starts with "TERMINAL TOOL" and gives a numeric boundary. Route is for "confidence at least 0.6", escalate is for "below 0.6 even after clarification". `classify_claim` and `request_clarification` also overlap, and clarification is limited to "genuinely ambiguous between two or more types". In claim_03 this produced one `request_clarification` ("What caused…") at turn 3, followed by a `record_claim_fact` for `cause` at turn 4. Tool failures use `_err()` in `tools.py`, which returns `{is_error, error_category: transient|permanent, is_retryable, message}`. That lets the agent retry transient failures and stop retrying permanent ones. None of my 8 traces contained a tool error, so I know this shape from the code and tests, not from my run.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → claim_05_auto_collision took 4 turns, 14,983 input / 975 output tokens, $0.0199 and 16.4 s. The whole run of 8 claims cost $0.1217, with 2 to 5 turns per claim. The `solution/` folder has no README sample and the capstone README gives no per-claim numbers, so I have nothing to diff against and won't invent a comparison. Within my run, cost tracks turns. Input tokens grow each turn because the full history is resent, so the 5-turn claims (03, 04) cost about $0.020–0.022 and the 2-turn claims (01, 02) cost $0.0083.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → `runs/20261117-085928/budget.json` has baseline 38,708 tokens, assembled 16,871, and reduction 56.41%, which clears the ≥50% bar. My baseline is lower than the README's ~47k, probably because `count_tokens` is model-authoritative. The `active` section dominates at 15,789 tokens, about 94% of the assembled context. It is the payment-method update still in progress, and the next turn depends on exact details such as the AVS mismatch and the last-4 values. A summary can drop or paraphrase those, so `compressor.py` never summarizes it and keeps it byte-exact.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Rule: resolved segments get compressed, and the active segment and the structured case-facts block are never touched. `summarize_segment` refuses any segment whose status is not `resolved`. Refund went from 12,334 input tokens to a 409-token section, with a 396-token model output. Subscription went from 11,475 to 487, with a 474-token output. Those two cut about 96–97% each. Active stays at 15,789 and `case_facts` is 204. Resolved issues are history, so a lossy summary is acceptable. The active issue is live state, so the cost of loss is higher than the cost of the tokens.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → `eval.jsonl` passed 6 of 6. The control (`eval_control.jsonl`) reran only Q1 and Q6, the two flagged `required_in_control_fail`, with case facts stripped. Q1 still passed ($22.14) because the refund summary kept that figure. Q6 regressed: the model said "there is no structured status token recorded" instead of `in_progress`. With the facts block in place it answered "`payment_update_status`: in_progress". So the structured block carries exact tokens that prose summaries and even the verbatim transcript do not reliably surface. The control isolates the facts block as the cause, because the only change was removing it.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → `.claude/rules/tests.md` has `paths:` set to `"**/*.test.tsx"` and `"**/*.test.ts"`. The other two rules use `src/api/**/*` (`api.md`) and `src/components/**/*` plus `src/pages/**/*` (`react.md`). Test files are co-located across the tree, so no single directory owns them. A directory CLAUDE.md can't cover files scattered across directories, and it would load whenever you work in that directory. A glob rule loads only when matching files are touched, which keeps unrelated context out. The validator checked all three rules and printed `OK`, and `pytest` gave 35 passed.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → `.claude/skills/deploy-check/SKILL.md` has `context: fork` and an allowlist of `Read`, `Grep`, `Glob`, `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(git rev-parse:*)` and `Bash(git ls-files:*)`. Forking runs the validation in a sub-agent, so the file reads and diffs stay out of the main session and only one pass/fail summary comes back. The allowlist means a pre-deploy check cannot edit files or run arbitrary commands. Without the fork, the main context fills with check output. Without the allowlist, a "check" could mutate the tree it is checking.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator printed only `OK`. It requires the CLAUDE.md scope block to mention both levels, say user scope is not version-controlled, and include a user-scope example. `CLAUDE.md` line 11 gives project-level as `./CLAUDE.md`, `.claude/standards/` and `.claude/rules/`, which are in git and shared with the team. Line 12 gives user-level as `~/.claude/CLAUDE.md`, `~/.claude/commands/` and `~/.claude/skills/`, which stay on the laptop. Project example: `.claude/rules/api.md`. User example: the personal `/morning` summary command the file itself names.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The warm tier in `data/warm.sqlite` holds 40 defects. The shift run's `defects_since` returned 17, matching `shift_scratchpad.jsonl`: "17 new defects analyzed since 2026-04-01T00:00:00Z". The query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` in `warm.py`, backed by `idx_defects_ts` (and `idx_defects_shift_ts`). Filtering and limiting happen in SQL, with no Python-side filtering. Only those rows go into the prompt. `test_defects_since_uses_index_and_does_not_load_full_table` pins this behavior. The model sees a bounded slice, so prompt size doesn't grow as history accumulates.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `recovery.decide()` returns `"fresh"` for an empty or complete manifest. It returns `"resume"` only if the last step is within `STALE_RESUME_THRESHOLD_MINUTES = 30`, which is about 1/16 of an 8-hour shift. Older partials are restarted from scratch with the manifest's findings injected as a "Prior summary" (`invocation.py`). After 30 minutes the working set may be stale, and resumed context can carry half-finished reasoning forward. A fresh start with a short summary gives a clean context plus only the verified facts. Tests covering this are `test_recovery_decide_truth_table` and `test_threshold_constant_is_30_minutes`.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → `data/hot_state.json` is 965 bytes against `HOT_STATE_BYTE_BUDGET = 5_120`, about 19% of the budget. It holds 17 defect hashes (capped at 20), the shift summary, `active_alerts` and `threshold_statuses`. The file is loaded into every invocation's prompt, forever. Without a hard cap, small per-shift growth compounds until it crowds out working context, and no single run would show it. `write_atomic` raises if the payload exceeds the budget. It writes to a temp file, calls fsync, then uses `os.replace`, so a crash never leaves a torn file.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `claude-haiku-4-5-20251001`, called through `claims_intake/client.py` and `retail_context/client.py`. It produces the reasoning, such as choosing `route_to_adjuster` for claim_05 or compressing the refund segment.
    → Harness: `claims_intake/loop.py` and `tools.py`. They wrap the model with the `stop_reason` loop, tool schemas, executor and Budget. `.claude/rules/` and `.claude/skills/deploy-check/SKILL.md` do the same for Claude Code.
    → Orchestration: `shift_monitor/pipeline.py`, `recovery.py` and `manifest.py`. They decide when a run starts, whether to resume or start fresh, and what survives between runs in `hot_state.json`. These decisions sit above any single model conversation.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → In code: the `run.py` backstop. It fired for 5 of 8 claims in `runs/20261117_085556` with `"no terminal tool called before end_turn"`, and wrote `escalations.jsonl`. The 5,120-byte budget in `state.py` is another. In prompt: the instruction "call `classify_claim` exactly once… If confidence is below 0.6, prefer escalate_to_human", and the "clarify once" guidance. Code is right when a violation is costly or irreversible, such as a claim lost without routing or corrupted state. Prompts are right for judgment that varies by case, such as whether a claim is ambiguous. The 5-of-8 backstop rate shows that prompt guidance alone is not enough on the paths that matter.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → The principle is to give the model the smallest context that preserves what the next decision needs. System 2 compresses within one conversation: 38,708 down to 16,871 tokens (56.41%). It uses model summaries for resolved segments (12,334 → 409 and 11,475 → 487), verbatim storage for the active issue (15,789) and a 204-token facts block. System 4 manages context across runs. SQL returns 17 of 40 rows, `hot_state.json` stays at 965 of 5,120 bytes, and the manifest supplies a summary on recovery. System 2 uses lossy summarization because the history is already in the prompt. System 4 uses retrieval and a hard cap because the history lives outside it. Both keep exact facts (case facts, hashes) separate from compressible narrative.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `test_recovery_decide_truth_table` in `tests/test_us03_crash_recovery.py` covers every resume/fresh case. My System 4 run was a clean offline shift, so it never exercised a crash or a stale manifest. If recovery picked wrongly, I would only find out in production after a mid-shift failure. `test_hotstate_rejects_more_than_20_hashes` is similar: my 17 hashes sit under the cap, so the run says nothing about what happens at 21. These failure paths need a test because they may not occur until well after launch.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 1, the claims agent. Its tools write facts and queue entries, and the terminal tools put a claim in `queues/*.jsonl` or `escalations.jsonl`. The worst case is a claim misrouted to a wrong queue, for example a high-severity liability claim sent to property_damage, which delays an injury claim. A runaway loop is bounded by `Budget.check()` at the top of each turn, and a model that stops without acting is caught by the backstop and escalated to a human. Wrong output stays inside files that adjusters review, because the agent has no payment or policy-mutation tool. The kill switch is the budget (or ending the process, since `--all` exits 0 only when every claim terminates). There is no per-claim rollback of a queue write.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The main surprise was in System 1. Tests passed, 30 vs the 29 the README expects, but `runs/20261117_085556` escalated 5 of 8 claims through the backstop (01, 02, 03, 06, 07). For example, claim_01 ended at turn 2 with `end_turn` and no terminal tool, and claim_02 (stolen bike) was escalated with empty `candidate_claim_types`. I did not tune the prompt and left the run as evidence. The backstop worked as designed, and the loop was not at fault. What I checked: all four suites pass (30, 30, 35, 33) and the System 3 validator prints `OK`. Environment drift: `git status` shows `uv` added `[dependency-groups] dev = pytest>=9.2.0` to all four `pyproject.toml` files, and the run used Python 3.13 rather than the 3.10 the repo targets.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → In System 1 I'd make the terminal action part of the loop's contract rather than a prompt request followed by a backstop. Five of 8 claims reached `end_turn` without a terminal tool. The fix would be a harness-level nudge: when `end_turn` arrives with no terminal call, append a tool result or user turn asking the model to route or escalate, allow one retry, and escalate only if that fails. The 5 escalations in `runs/20261117_085556/escalations.jsonl` carried facts and a reason but lost the classification work the model was ready to do. Claims 01, 02 and 07 were stopped at 2–3 turns with obviously classifiable content (kitchen fire, stolen bike, tree on car). One retry turn costs roughly $0.005, while a needless human escalation costs more.
