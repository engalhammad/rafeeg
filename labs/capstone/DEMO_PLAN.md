# 8-minute talk and live demo plan (Track 1: Rafeeq)

Rubric weights: demo 25% · context/harness decisions 25% · evals 20% · security 20% · presentation 10%.

| Time | What you do | Rubric |
|---|---|---|
| 0:00–0:45 | The problem: permit questions are slow and risky (data leaks, wrong changes). Rafeeq = bilingual assistant with a human gate on changes | Presentation |
| 0:45–2:30 | **Live demo 1:** `ask` a fee question in English, then in Arabic. Show the tool call and the answer with the policy cited | Demo 25% |
| 2:30–3:30 | **Live demo 2:** a write (book inspection). Show the `🛡️ Allow ...? [y/N]` prompt, press `y`, show the result. Then show `audit.jsonl` (timestamps, approval, masked ID) | Demo + Harness |
| 3:30–5:00 | Design decisions: AGENTS.md (context file), retrieval over policies, compaction (24 messages), result cap (3000 chars), budgets (8 steps, $0.05), allow/ask permissions. For each: what and why | Context/harness 25% |
| 5:00–6:15 | Evals: 15 tasks, 2 configs (B adds the Facts rule). Show the pass-rate table and 2 failures with the root cause and what you tried | Evals 20% |
| 6:15–7:30 | Security: the trifecta (private data yes, untrusted content yes, egress no), the injected-refund attack, the defence (my code guard that denies writes to permits the person never named, human approval in code, model rule as a third layer); show a `guard_denied` line in `audit.jsonl`, residual risk, PDPL points | Security 20% |
| 7:30–8:00 | Limitations, next steps, repo link and how to reproduce (`uv run ...`) | Reproducibility 10% |

## Before the talk
- [ ] Run each demo command once so the cache makes it fast
- [ ] Record a backup video of both demos
- [ ] Have `eval.csv` open for the Q&A
- [ ] Know the answer to: why this one change between A and B? why is `send_email` not wired? what happens at 8 steps?

## Likely Q&A
- **Why human approval on writes?** Writes change real records, and a model can be tricked by text in a message. The gate is code, not a prompt.
- **What does the model do if it hits the budget?** The run stops (`max_steps` or `max_cost`) and returns no answer; the harness tells you why.
- **How do you know it improved from A to B?** The pass-rate table and the tasks that flipped.
