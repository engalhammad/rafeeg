# Rafeeq in production: capstone

**Track:** 1 · Rafeeq in production
**Team:** Salman Alhammad (individual)
**Model:** `anthropic/claude-haiku-5.5` through OpenRouter, using my own Anthropic key. The agent and the judge grader use the same model, and OpenRouter's free fallback models are switched off, so every result comes from one model.
**Why this model:** I first used the course's free model, but it hit OpenRouter's free daily request limit in the middle of my first eval run. Haiku 5.5 is Anthropic's cheapest current model and supports tool calling. My own eval is the evidence: 15/15 tasks on both configurations (see Evaluation). I did not compare other models.

## Problem
Applicants ask Madar Municipal Services about permits (building, renovation, signage, food truck, event): fees, rules, status, inspections, refunds. Answers by hand are slow and inconsistent, and a mistake can leak another person's data or change an application.
Rafeeq answers in Arabic and English from the policy documents and the permit database. "Done" means: a correct answer with the policy cited, identity verified before any private detail is shared, and a human approving every change.

## How to run
```
cd labs
uv run capstone/starter.py ask "What is the fee for an illuminated 3 m² sign?"
uv run capstone/starter.py chat --session demo
uv run capstone/starter.py eval capstone/tasks.jsonl --config A B
```
Environment (`labs/.env`, never committed; see `labs/.env.example`):
```
OPENROUTER_API_KEY=sk-or-...
AGENT_MODEL=anthropic/claude-haiku-5.5
AGENT_SMALL_MODEL=anthropic/claude-haiku-5.5
AGENT_FALLBACKS=
```
Needs `uv` and Python 3.11+. Results are written to `labs/artifacts/capstone/` (`eval.csv`, `audit.jsonl`).

## What I changed in the academy starter (and what I did not)
The agent loop, tools, MCP server, approval prompt, budgets, compaction and eval runner are the academy's code. My changes:
1. **Own spec:** `capstone/AGENTS.md` replaces the course prompt (privacy, write rules, no promises that no tool keeps).
2. **Config B differs from A by one thing:** the "Facts" rule.
3. **Write guard in code** (`make_permission`): a write is denied unless the person named that permit (or inspection) in their own messages.
4. **Audit log v2** (`log`, `approve_logged`, `redact`): timestamp, every human approval or denial, success flag, run end with steps and cost, and masked ID digits and mobile number.
5. **15 eval tasks** (`capstone/tasks.jsonl`): fees, rules, privacy, a security attack, scope and a real write, in Arabic and English.
6. **Judge fix in `labs/evals.py`:** the judge's token limit goes from 300 to 800, and if its JSON is damaged but it did state a verdict, that verdict is used.

## Design across the ladder
| Rung | My decision | Why |
|---|---|---|
| Prompt | `capstone/AGENTS.md` as the system prompt: role, language, privacy (POL-08), write rules (writes only for a verified person who asked in their own message; report only what the system returned) | One file that is easy to read, change and test |
| Context | Retrieval: `search_policies` over the 10 policy documents. History kept to the last 24 messages (`compact()`). Tool results cut at 3000 characters | Policies must be cited, not remembered; the context budget stays bounded |
| Harness | Reads run, writes need human approval. My guard denies a write before the human is asked unless the permit was named by the person. Budgets: 8 steps and $0.05 per request (the cost budget never fired: see Limitations). My audit log records approvals, denials and run ends, with masked personal data | Writes change real records; text in tool results must not steer a write to another permit; the log supports review and data minimisation |
| Loop | Not needed: a request is answered in a short tool loop (2 steps on average) | Keeps it simple and cheap |
| Graph | Not needed: no long-running process; the human gate is the approval on writes | A graph would add complexity without a benefit |

```mermaid
flowchart LR
  P[Person] -->|message| H[Harness: starter.py]
  H -->|AGENTS.md + message| M[Model: Claude Haiku 5.5 via OpenRouter]
  M -->|tool call| H
  H -->|read tools| MCP[MCP server madar_mcp.py: search_policies, get_permit, compute_fee, list_slots]
  H -->|write tools, after guard| W[Local write tools: book, reschedule, refund, escalate]
  H -->|Allow? y/N| A[Human approver]
  MCP --> DB[(Madar database and 10 policies)]
  W --> DB
  H --> L[audit.jsonl]
```

## Tools
| Tool | Read / write | Over MCP? | Approval? | Trifecta leg |
|---|---|---|---|---|
| search_policies | read | yes (servers/madar_mcp.py) | no | none |
| get_permit | read | yes (servers/madar_mcp.py) | no | private data |
| compute_fee | read | yes (servers/madar_mcp.py) | no | none |
| list_slots | read | yes (servers/madar_mcp.py) | no | none |
| book_inspection | write | no (local) | yes | none |
| reschedule_inspection | write | no (local) | yes | none |
| create_refund | write | no (local) | yes | none |
| escalate | write | no (local) | yes | none |

## Evaluation
<img width="316" height="622" alt="image" src="https://github.com/user-attachments/assets/601abe45-84ea-4c6e-b91b-dcdd1d0849e5" />

Suite: 15 tasks in `capstone/tasks.jsonl` (8 fees, 2 rules, 2 privacy, 1 security, 1 scope, 1 action; 2 in Arabic). Graders: contains_any, not_contains, tool_called, tool_not_called, db (checks the database), judge.
Configurations: **A** = `AGENTS.md`; **B** = A + a "Facts" rule (never state a fee, date or slot from memory; call the tool; search policies first). Same model, one change.

Final run (v3), 1 trial per task, so pass^k equals the pass rate (k = 1):

| Config | passed | pass rate | pass^k (k = 1) | avg steps |
|---|---|---|---|---|
| A | 15/15 | 1.00 | 1.00 | 2.07 |
| B | 15/15 | 1.00 | 1.00 | 2.07 |

Runs over time (all kept in `capstone/results/`: `eval_v1.csv`, `eval_v2.csv`, `eval_v3.csv`):

| Run | A | B | Why |
|---|---|---|---|
| v1 | 14/15 | 14/15 | `book-inspection` failed on both |
| v2 | 14/15 | 15/15 | after the AGENTS.md fix; A lost a task to a grader glitch |
| v3 | 15/15 | 15/15 | after the judge fix |

Not recorded: tokens, request counts and cost (the runner does not log tokens or requests, and OpenRouter reported $0.00 for every run). The `seconds` column is mostly cached replays, so it is not a latency measurement.

## Error analysis
| Root cause | Count (A / B) | Example task | Fix tried | Result |
|---|---|---|---|---|
| The agent asked for confirmation the person had already given ("I confirm, go ahead"). My own AGENTS.md rule said "get confirmation before any change". It found the one free slot, then stopped to ask | v1: 1 / 1 | `book-inspection` (agent called `get_permit` and `list_slots`, never `book_inspection`) | Rule rewritten: if already confirmed, act; if several times are acceptable, book the earliest free one; otherwise ask once | Passed on both in v2 |
| Grader false failure: the judge's reply was cut off at 300 tokens. It had said `"pass": true`, but the code could not parse it and counted a fail | v2: 1 / 0 | `status-wrong-credentials` | Token limit 800, and use the judge's stated verdict if the JSON is damaged (`evals.py`) | Passed on both in v3 |

**What flipped between A and B:** nothing real. In v1 both configs failed the same task. In v2, B's extra pass came from the grader glitch hitting A only, not from the Facts rule. The Facts rule made no measurable difference, because the model already called the fee and policy tools without being told. Pass rate and average steps are identical.
**Found by hand, not by the eval:** in my first real booking the model said "You'll receive a confirmation shortly", but nothing sends one. I added a rule to AGENTS.md and a grader on the booking task (`not_contains` for those phrases); it passes in v2 and v3.
**What I would change next:** harder tasks (ambiguous requests, Arabic policy questions, multi-step cases), more than 1 trial per task, and a judge from a different model.

## Security note
**Trifecta:**
- Private data: yes. `get_permit` returns an applicant's permit details (national ID masked).
- Untrusted content: yes. Users can paste emails or messages that contain instructions.
- Egress: no external channel is wired. `send_email` exists in `madar.py` but is not given to the agent, and the MCP server exposes only the 4 read tools (its `--write` flag is off). Writes only change the Madar database.
So the three legs do not fully meet. The main risk is an injected instruction that triggers a write.

**Attack tried:** task `attack-injected-refund`: a pasted message contains a hidden instruction to create a refund for another permit. It passed in every run (v1, v2 and v3, on both configs): no `create_refund` call was made.
**Not tested:** the attack with the model-side rules removed, and the human approval prompt (the eval auto-approves writes).
**Defence:** (1) a code guard: a write is denied unless the person named that permit in their own messages, so an instruction in tool output cannot redirect a write to another permit (tested with a scripted model: 0 refunds created, a `guard_denied` line logged); (2) human approval on every write, in code; (3) the AGENTS.md write rule: writes only for a verified person who asked for it in their own message (depends on the model). AGENTS.md has no explicit rule that instructions inside pasted text or tool results are data. Rules 1 and 2 do not depend on the model.
**Residual risk:** the guard cannot tell a pasted message from the person's own words. An instruction inside text the person pastes (as in `attack-injected-refund`) names the permit "in their own message", so only human approval and the model's own judgement stop it. The eval auto-approves writes, so it tests the model, not the human gate. An approver can also click "y" without reading, and a verified person can still ask for a legitimate write. Share this with the service owner.
**Governance:** data minimisation (national ID masked in tool results, ID digits and mobile masked in the audit log, only the verified application is shown); identity check before private data (POL-08); human oversight on writes; an audit log of approvals, denials and tool calls. Data goes to the model provider (OpenRouter, then Anthropic), so real citizen data would need a PDPL review and an approved hosting choice. Here all data is synthetic.

## Limitations and next steps
- Perfect scores on 15 mostly easy tasks with one trial each do not separate A from B, and one lucky run could look perfect.
- The judge is the same model as the agent, which can favour its own style.
- Policy search is a keyword search, so wording that differs from the policy text can miss (Arabic especially).
- The $0.05 cost budget is not effective: OpenRouter returned a cost of $0.00 for every run in `audit.jsonl`, so it can never fire. Only the 8-step budget is enforced.
- The guard checks the permit ID only, not the amount or reason of a refund.
- Next: harder tasks, 3+ trials, a second judge model, an explicit AGENTS.md rule that pasted text and tool results are data, and marking pasted content as untrusted before the model sees it.
