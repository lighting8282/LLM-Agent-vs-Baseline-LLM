# LLM Agents vs. a Baseline LLM

Does giving a language model tools, a planning loop, and error recovery actually
make it better at multi-step tasks — and what does it cost in latency?

This project builds two systems on the **same underlying model**
(`llama-3.3-70b-versatile`, served via Groq) and benchmarks them on an identical
task set, so the only variable is agentic capability.

> **Research question:** How well can an LLM agent perform multi-step tasks,
> self-recovery, API access, and replanning, compared to a baseline LLM that does none of these?

## The two systems

**Baseline** — a single prompt in, a single response out. No tools, no planning,
no retries. A standard chat assistant.

**LLM Agent** — wraps the same model in a full agentic loop:

1. **Plan** — decompose the query into tool-call steps (`plan_steps`)
2. **Execute** — run each step, timing every call (`agent_loop`, up to 3 rounds)
3. **Self-assess** — decide whether the task is actually finished (`agent_thinks_done`)
4. **Recover** — on failure, replan and retry with different tools (`replan_after_failure`)
5. **Synthesize** — fold tool outputs into a final answer (`final_answer`)

## Tools

| Tool | Source | Assigned difficulty |
|---|---|---|
| `calculator` | Local, AST-parsed arithmetic | 1 |
| `search` | Tavily web search API | 2 |
| `weather` | OpenWeather API | 3 |

The `api_difficulty` ranking is used as a regression predictor — the idea being
that a local computation, a search index, and a live weather lookup impose
increasing latency costs.

## What gets measured

Every tool call is logged to JSONL with `execution_time`, `success`,
`error_flag`, `error_type`, and `recovery_action`, alongside per-task
`task_complexity`, `api_difficulty`, and `tool_count`.

Three regressions (`LLM_Agent_pipeline.run_regression_comparison`) test what
drives total latency, fitting agent and baseline side by side:

1. `task_complexity` → `total_latency` — does a task needing more tool calls take longer?
2. `tool_count` → `total_latency` — does the number of calls the agent *actually made* predict latency?
3. `api_difficulty` → `total_latency` — do slower APIs dominate the cost?

## Findings

The agent handles work the baseline structurally cannot: it recovers from failed
tool calls, retrieves live data through APIs, and chains multiple steps toward a
single answer. The baseline, restricted to one shot with no tools, has no
recovery path at all.

That capability is not free — the agent's latency scales with how many tools a
task requires, which is exactly what the regressions quantify. The tradeoff is
capability against speed and cost, and which side wins depends on whether your
tasks actually need multi-step reasoning.

## Repository layout

```
LLM-Agents.qmd           Quarto source for the full report
LLM-Agents.html/.pdf     Rendered report (HTML needs LLM-Agents_files/)
Agents_Multi_step.py     Agent loop, baseline, tools, logging, benchmark driver
LLM_Agent_pipeline.py    Regression analysis and comparison plots
agent_runs.jsonl         122 tool-call records across 33 agent queries
baseline_runs.jsonl      36 baseline query records
agentic_results.csv      Per-query agent summary (33 queries)
baseline_results.csv     Per-query baseline summary (36 queries)
LLM-Agentsref.bib        Bibliography
apa.csl                  APA citation style
```

## Running it

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

Copy `.env.example` to `.env` and add your keys:

```
GROQ_API_KEY=
TAVILY_API_KEY=
OPENWEATHER_API_KEY=
```

Then render the report:

```bash
quarto render LLM-Agents.qmd
```

Rendering re-runs the full benchmark against all three live APIs, which takes
time and consumes quota. The committed `.csv` and `.jsonl` files are the outputs
of a previous run if you would rather analyze those directly.

## Known limitations

- The agent arm covers 33 queries and the baseline 36, so the two arms are not
  strictly matched — a handful of agent runs did not produce result rows.
- `task_type` is recorded as `unknown` for most logged calls, so any analysis
  sliced by task type is limited.
- The baseline's "tool usage" columns come from keyword matching in
  `benchmark_non_agentic`, not from the baseline actually calling tools. They
  exist to make the two CSVs structurally comparable, not to imply the baseline
  had tool access.

## Disclaimer

Use AI tools responsibly. AI is a powerful assistant but easy to over-rely on —
don't blindly trust the output.

## Author

Jonathan Trnka
