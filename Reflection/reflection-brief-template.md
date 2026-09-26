# Reflection Brief — Harness Engineering Capstone

**Name:** Suman M.
**Date:** 2026-09-26

Every answer below cites an artifact from my own capstone run, including run IDs, file paths, token counts, claim outcomes, or test counts.

## Environment

- Model(s): Claude via the Anthropic API
- OS / Python: Linux environment / Python 3.11.8 for the capstone environment
- Approx. API spend: The System 1 and System 2 runs were completed using the provided Anthropic API environment; the available run artifacts record token usage and the hosted environment displayed approximately $26.2232 remaining/available at the time of the run.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In `System-1-Agentic-Loop/20260926_155401/traces/claim_03_water_damage.jsonl`, the five recorded turns have the sequence `tool_use → tool_use → tool_use → tool_use → tool_use`; the final tool call routes the claim to an adjuster. The loop is implemented in the Claims Intake agent's stop-reason-driven loop and dispatches on the Anthropic `stop_reason`: `tool_use` continues the loop by executing the requested tools, while `end_turn` terminates normally and unexpected values are treated as errors. The trace demonstrates that the model can dynamically request policy lookup, fact recording, clarification, classification, severity assessment, and routing instead of following a fixed number of turns. This behavior is also consistent with the course repository description of the loop as dispatching on `tool_use`, `end_turn`, and unexpected stop reasons.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → One important anti-pattern is treating the model response as a single fixed action instead of dispatching from `stop_reason`. If the loop assumed one response was enough, the water-damage claim in `System-1-Agentic-Loop/20260926_155401/traces/claim_03_water_damage.jsonl` would stop after the first `lookup_policy` call and would never record the claim facts, request the missing cause clarification, classify the claim, assess severity, or route it. The actual trace requires five tool-use turns to reach the routing decision. Therefore a fixed one-shot or fixed-step approach would lose the dynamic behavior demonstrated by the run.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `record_claim_fact` and `request_clarification` both operate on information needed to complete a claim, but their descriptions distinguish recording known information from asking the policyholder for information that is still ambiguous. In the water-damage trace, the agent used `record_claim_fact` for concrete facts such as `incident_location`, `incident_description`, and damage fields, while it used `request_clarification` for the unresolved cause of the basement flood. Structured tool errors are also useful because they preserve machine-readable failure information that the loop can inspect and recover from, instead of forcing the model to interpret an unstructured error string. The distinction is visible in `System-1-Agentic-Loop/20260926_155401/traces/claim_03_water_damage.jsonl`, where different tools are selected for different information states.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → The water-damage trace in `System-1-Agentic-Loop/20260926_155401/traces/claim_03_water_damage.jsonl` contains five turns and reaches a final `route_to_adjuster` action. The trace records increasing input-token counts of 2,776, 2,960, 3,584, 3,784, and 4,358 across the five turns, showing that the conversation grows as tool results and facts are added. My run therefore differs from a fixed README example because the actual number of turns depends on the claim's information requirements; this claim required an additional clarification step before classification and routing. The recorded trace is the authoritative artifact for my run-specific turn count and token usage.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → `System-2-Context-Strategy/runs/20260926-160130/budget.json` reports a baseline of **38,708 tokens**, an assembled context of **16,760 tokens**, and a **56.7% reduction**. The `active` section dominates the assembled context at **15,789 tokens**, compared with 204 tokens for `case_facts`, 396 for `resolved_refund`, and 389 for `resolved_subscription`. The active conversation is kept because it contains the current working context needed to continue the interaction, while older resolved material can be compressed. The large reduction demonstrates that context management can substantially lower the working prompt while preserving the information needed for the current task.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The context strategy summarizes older resolved conversation sections when their detailed wording is no longer required, while preserving the durable case-facts block exactly because those facts are authoritative state. In my `budget.json`, `case_facts` is only **204 tokens**, while `resolved_refund` is **396 tokens** and `resolved_subscription` is **389 tokens**, showing that the resolved sections can be represented compactly. The compression calls themselves processed large inputs of **12,334** and **11,475** tokens and produced outputs of only **383** and **376** tokens respectively. This makes the rule practical: preserve authoritative facts, compress historical narrative, and keep the active working context available.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The regression is **Q6**, which asks for the exact structured status token for the payment-method update issue. In `System-2-Context-Strategy/runs/20260926-160130/eval.jsonl`, Q6 passed with the exact answer **`in_progress`**, while `eval_control.jsonl` returned `unknown` and failed because the structured status was absent. The other shown control question, Q1, still passed with the refund amount of **$22.14**. This proves that the durable facts block is not merely redundant context: removing structured case facts can cause exact-state questions to become unanswerable even when the broader conversation remains available.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The System 3 configuration uses path-scoped rule files with glob frontmatter so that rules apply only to files matching their intended surface. The validator artifact `System-3-Claude-Code-Config/test-output.txt` confirms this behavior with `test_ac_02_01_react_rule_has_component_and_page_globs`, `test_ac_02_02_api_rule_has_api_glob`, and `test_ac_02_03_tests_rule_has_test_globs`. This is better than placing every convention in a directory-level `CLAUDE.md` because the rules can cross directory boundaries while still applying only to the relevant file types. The validator also confirms that React, API, and test files match the intended combinations of rules.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → The deploy-check skill is configured with `context: fork` and a read-oriented `allowed-tools` list, as verified by `System-3-Claude-Code-Config/test-output.txt`, including `test_ac_04_02_name_description_context_fork` and `test_ac_04_03_allowed_tools_read_only`. Running it in a fork isolates the deployment inspection from the main conversation state, while the read-only allowlist prevents the check from modifying the repository. Without these constraints, a deployment check could accidentally push, deploy, write files, or otherwise change project state while it is supposed to inspect it. The skill documentation also explicitly states that it does not push, deploy, run migrations, run the test suite, or check secrets because those responsibilities belong to other gates.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The System 3 validator confirms that the `/review` command is project-scoped through `test_ac_03_06_is_project_scoped`, while user-level configuration is intentionally not versioned, as checked by `test_ac_01_05_documents_scope_and_user_level_not_versioned`. A project-level example is the repository's `/review` command because it is part of the team's shared configuration and is validated with the project. A user-level example is personal Claude Code configuration that remains outside the repository rather than being committed as team policy. This separation keeps shared engineering rules reproducible without forcing personal settings onto every contributor.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The warm-tier fixture contains **40 defects**, confirmed by loading `System-4-Orchestrator/fixtures/defects.json`. The pipeline calls `gather_new_defects()` in `System-4-Orchestrator/shift_monitor/pipeline.py`, which delegates directly to `warm.defects_since(since_ts, limit=50)` instead of loading and filtering the full table in Python. The test suite explicitly verifies this with `test_defects_since_uses_index_and_does_not_load_full_table` and `test_gather_new_defects_has_no_python_side_filtering` in `System-4-Orchestrator/test-output.txt`. This means the model receives only the relevant new-defect subset rather than the entire 40-record history, reducing prompt size and keeping historical storage outside the model context.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → The recovery policy uses a **30-minute staleness threshold**, verified by `test_threshold_constant_is_30_minutes` in `System-4-Orchestrator/test-output.txt`. The truth-table tests show that a recent incomplete state can be resumed, while a state older than 30 minutes is treated as fresh; for example, `31-False-fresh` passes while `29-False-resume` passes. A fresh run can be more reliable when an old state may represent an interrupted or inconsistent invocation, because the system can rebuild the current prompt from durable state and inject the previous summary rather than trusting stale intermediate execution state. The tests also verify that resumed prompts contain both required sections, providing a controlled recovery path rather than blindly replaying an old invocation.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
   → My `System-4-Orchestrator/data/hot_state.json` is **643 bytes**, verified from the final submission ZIP with `unzip -l Harness-Engineering-Capstone-Submission.zip`. The System 4 implementation enforces `HOT_STATE_BYTE_BUDGET` in `System-4-Orchestrator/shift_monitor/pipeline.py`, where `_trim_to_budget()` removes older alerts until the serialized state is within the configured limit. The test suite also verifies this behavior through `test_hotstate_roundtrip_and_size_budget` and `test_hotstate_rejects_more_than_20_hashes`, with **33 System 4 tests passing**. This budget matters because the system runs once per shift indefinitely: bounded hot state prevents the state file and future prompts from growing without limit.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `System-1-Agentic-Loop/20260926_155401/traces/claim_03_water_damage.jsonl` shows Claude selecting tools and controlling the agentic loop through `stop_reason`.
    → Harness: `System-3-Claude-Code-Config/test-output.txt` records **35 passing tests** that enforce configuration, scoped rules, review behavior, and the forked deploy-check skill.
    → Orchestration: `System-4-Orchestrator/shift_monitor/pipeline.py` and `System-4-Orchestrator/test-output.txt` show the outer system controlling SQL filtering, prompt construction, one Claude invocation, state persistence, recovery, and scratchpads.
    These artifacts demonstrate three different responsibilities: the model makes task-level decisions, the harness constrains how the agent operates, and orchestration controls durable execution across shifts.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic behavior is the System 4 atomic state update: `System-4-Orchestrator/shift_monitor/pipeline.py` calls `updated.write_atomic(hot_state_path)`, and `test_hotstate_atomic_write` verifies it. A prompt-guided behavior is the model deciding which claim tool to call next, demonstrated by the different tool sequences in `System-1-Agentic-Loop/20260926_155401/traces/claim_03_water_damage.jsonl`. Code enforcement is appropriate for safety, state integrity, permissions, and resource limits; prompts are appropriate for flexible reasoning where the exact next action depends on the current evidence.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 manages context inside a long conversation and reduced **38,708 baseline tokens to 16,760 assembled tokens**, a **56.7% reduction**, according to `System-2-Context-Strategy/runs/20260926-160130/budget.json`. System 4 manages context across shifts by keeping bounded hot state, querying only new defects from the warm tier, and writing durable summaries/scratchpads; its fixture contains **40 total defects**, while the pipeline requests only defects since a timestamp. Both systems apply the same principle: do not repeatedly expose the model to unnecessary history. System 2 achieves this with compression and preservation rules, while System 4 achieves it with tiered storage, SQL filtering, bounded state, and recovery.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → System 4's crash-recovery tests guarantee behavior that a normal successful shift would not expose. `System-4-Orchestrator/test-output.txt` verifies both `29-False-resume` and `31-False-fresh`, as well as the **30-minute threshold**, incomplete-state loading, and atomic writes. A successful run alone would only show that the happy path works; it would not demonstrate what happens after a crash or stale state. This matters before shipping because long-running systems must remain correct when failures occur between writes and invocations.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → For System 3, the blast radius of a misconfigured deploy-check skill could affect deployment decisions or repository safety, but the configuration intentionally limits it. `System-3-Claude-Code-Config/test-output.txt` verifies that the skill has a forked context and read-only allowed tools, while the documented behavior explicitly says it does not push, deploy, or run migrations. The `/review` command provides another enforcement point for reviewing configuration changes, including changes to the deploy-check allowlist. The practical kill switch is therefore the read-only tool boundary and the project-level review gate before configuration changes are accepted.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → One environment issue was that `nano` was not installed: running `nano "Reflection/reflection-brief-template.md"` returned `bash: nano: command not found`. I switched to editing the file directly through the VS Code file explorer/editor instead, which was available in the lab environment. I then verified the result from the terminal with `grep -cE '^[[:space:]]*[0-9]+\.' "Reflection/reflection-brief-template.md"`, which returned **20**, confirming that all reflection questions were present. I also verified the System 3 and System 4 artifacts remained intact with **35** and **33** passing tests respectively.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
   → I would make the orchestration state inspection more explicit by retaining a small representative `hot_state.json` artifact after a successful System 4 run and documenting its observed size. The current implementation in `System-4-Orchestrator/shift_monitor/pipeline.py` already enforces the hot-state byte budget and atomically writes the state, and the **33 passing System 4 tests** verify the behavior. My final submission contains `System-4-Orchestrator/data/hot_state.json`, which is **643 bytes**, so the budget behavior can also be inspected directly from the submitted artifact. Retaining and explicitly documenting this representative state would make the evidence easier to inspect for future debugging without changing the core architecture.