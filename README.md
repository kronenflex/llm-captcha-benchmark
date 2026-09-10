# LLM CAPTCHA Benchmark

**How well does a local multimodal LLM solve interactive CAPTCHAs when it has to drive a real
browser — and how robotic does its mouse look while doing it?**

A self-contained Jupyter notebook that benchmarks vision-language models on **28 interactive
CAPTCHA families**, all rendered and graded locally. The model sees a screenshot, decides on one
action, performs it in a real Chromium browser through Playwright, reads the result, and tries
again. An embedded Flask server grades every submission by **replaying the puzzle's seeded PRNG**,
so a pass never depends on the client's own claim.

Two things are measured, and they are independent:

1. **Can it solve the puzzle?** Pass rate per CAPTCHA family, per model, on identical seeds.
2. **Does its motion look human?** Every drag is scored on six motor-control metrics derived from
   [Cerno](https://github.com/plawio/cerno) — an open-source CAPTCHA that detects agents by *how*
   the pointer moves rather than *what* it clicks.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/)
[![Jupyter](https://img.shields.io/badge/Jupyter-notebook-orange.svg)](https://jupyter.org/)

> **Scope.** Every CAPTCHA in this benchmark is a **local synthetic fixture** served from
> `127.0.0.1` by the notebook's own Flask server. The notebook does not contact, solve, or
> interact with any live CAPTCHA provider, and it ships no bypass for a deployed system. It is a
> capability measurement in the tradition of MCA-Bench and Open CaptchaWorld. See
> [Scope and responsible use](#scope-and-responsible-use).

---

## Table of contents

- [What this measures, and why it is hard to measure](#what-this-measures-and-why-it-is-hard-to-measure)
- [The 28 CAPTCHA families](#the-28-captcha-families)
- [Architecture](#architecture)
- [Motor-control metrics](#motor-control-metrics)
- [The agent](#the-agent)
- [Benchmark modes](#benchmark-modes)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Comparing several models](#comparing-several-models)
- [What a run produces](#what-a-run-produces)
- [The routing policy](#the-routing-policy)
- [Notebook layout](#notebook-layout)
- [Scope and responsible use](#scope-and-responsible-use)
- [References](#references)
- [Contributing](#contributing)
- [License](#license)
- [Citation](#citation)

---

## What this measures, and why it is hard to measure

Most CAPTCHA benchmarks for vision models are **one-shot and static**: here is an image, what is
the answer. That measures perception. It does not measure the thing that matters for an agent,
which is a loop — look, act, observe the consequence, correct.

This benchmark closes that loop, and three design choices make the number honest:

**Grading is server-side.** Each fixture seeds a `mulberry32` PRNG from `?pu=<seed>`. The server
replays the same PRNG to recompute the answer, so the client's "solved" claim is never trusted.
The answer-determining draws happen *first*, before any decorative rendering, so the server only
replays a short, exact prefix.

**The fixtures leak nothing.** Every fixture shows only the instruction and the picture. None
exposes the answer, the geometry, or the formula to the client — exactly as real GeeTest, Arkose,
hCaptcha and NetEase YiDun challenges behave. A model cannot read the answer out of the DOM.

**`reveal_cookbook` is off by default.** An earlier version shipped a per-type "cookbook" that
handed the model the puzzle constants. That measured the cookbook, not the model. With it off, the
model reasons from a generic skills library, so the benchmark measures **its own** ability.

The interesting question is where tools plus a code sandbox take a vision model on tasks that
one-shot VLMs are bad at — sliders, rotation, fine discrimination, multi-step selection — and
which models separate from which once the answer is not spoon-fed.

## The 28 CAPTCHA families

**14 MCA-Bench-style base families** ([arXiv:2506.05982](https://arxiv.org/abs/2506.05982)):

| Family | What it tests |
|---|---|
| `checkbox` | the trivial control case |
| `image_grid` | select all tiles matching a category |
| `slider_puzzle` | drag a piece into a gap — sub-pixel alignment |
| `text_captcha` | distorted character recognition |
| `rotation` | rotate an image upright |
| `sequential_click` | click targets in a required order |
| `inverted_letter` | find the inverted glyph — VLMs are weakest here |
| `hollow_shape` | outline vs filled discrimination |
| `geometric_shape` | shape classification under noise |
| `brightness` | rank by luminance — VLMs are weakest here too |
| `arithmetic` | read and evaluate an expression |
| `bingo_swap` | swap two cells to complete a pattern |
| `threed_shape` | 3D orientation reasoning |
| `dice_count` | counting pips under occlusion |

**8 real-world families** (AWS WAF / hCaptcha / GeeTest / Cloudflare style):

`semantic_grid` · `odd_one_out` · `drag_to_complete` · `image_slider` · `cloudflare_turnstile` ·
`pattern_grid` · `reference_match` · `tower_blocks`

**5 coverage-gap families** added from the literature comparison:

`maze_path` (path finding) · `hold_button` (press-and-hold) · `gomoku` (tic-tac-toe /
five-in-a-row) · `fine_click` (small-target precision) · `illusion_text` (text hidden in an
optical illusion)

**1 invisible family:**

`turnstile_invisible` — the other half of Cloudflare Turnstile. `cloudflare_turnstile` reproduces
what the interstitial *looks* like: a checkbox that passes when clicked. Managed Turnstile also
scores the **client environment and the pointer behaviour**, with no puzzle to solve at all. This
fixture is scored that way — on behaviour, not on an answer — which is the case where solving the
visible puzzle perfectly still fails.

Where MCA-Bench reports human and VLM pass rates for a family, those are recorded in the registry
alongside the fixture so your numbers sit next to the published ones.

## Architecture

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │  LM Studio            OpenAI-compatible server on localhost:1234    │
 │                       any loaded vision model; MODEL_HINT="auto"    │
 └───────────────────────────────┬─────────────────────────────────────┘
                                 │  screenshot in, ONE tool call out
                                 v
 ┌─────────────────────────────────────────────────────────────────────┐
 │  Agent loop          max_steps tool calls per attempt               │
 │                      max_attempts attempts per trial (memory grows) │
 │    tools:  click · drag · type · press_and_hold · run_js ·          │
 │            run_python · run_image_code · submit · reflect           │
 │    every call validated against a JSON schema                       │
 │    coordinates bounded to the viewport                              │
 └───────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 v
 ┌─────────────────────────────────────────────────────────────────────┐
 │  Screenshot annotator   coordinate grid + Set-of-Mark labels        │
 │                         (arXiv:2310.11441) + failure-trace arrows   │
 └───────────────────────────────┬─────────────────────────────────────┘
                                 │
                                 v
 ┌─────────────────────────────────────────────────────────────────────┐
 │  Playwright → headless Chromium, viewport 600x700                   │
 │  on ONE dedicated worker thread (the sync API is not thread-safe    │
 │  and cannot share the kernel's asyncio loop)                        │
 └───────────────────────────────┬─────────────────────────────────────┘
                                 │  HTTP
                                 v
 ┌─────────────────────────────────────────────────────────────────────┐
 │  Embedded Flask server on 127.0.0.1                                 │
 │    serves 28 self-contained HTML fixtures                           │
 │    grades by REPLAYING each fixture's seeded mulberry32 PRNG        │
 │    records raw pointer events for motor analysis                    │
 └─────────────────────────────────────────────────────────────────────┘
```

While the server is running you can open any fixture in your own browser, try it yourself, and
watch the server grade your submission — the notebook prints the links, plus a gallery index at
the server root. `preview_captcha(...)` renders one inline in the notebook.

## Motor-control metrics

Based on [Cerno](https://github.com/plawio/cerno). The insight is that LLM- and script-driven
drags are *geometrically clean* — constant speed, straight path, no natural pauses — while human
motion is variable, inefficient and punctuated by micro-pauses. A real motor-control CAPTCHA would
**reject** motion that scores as robotic even when the geometry is correct.

Each drag the agent performs is scored on six cues, aggregated per episode:

| # | Metric | Robotic when |
|---|---|---|
| 1 | Tangential velocity SD (coefficient of variation) | low — near-constant speed |
| 2 | Path efficiency (straight-line / actual path length) | > 0.9 — near-perfectly straight |
| 3 | Pause intervals (near-zero-velocity dwell points) | zero |
| 4 | Jerk (variability of the third derivative) | negligible |
| 5 | Angular-velocity entropy (Shannon, binned direction changes) | low |
| 6 | Inter-event interval variation | metronome-regular |

These combine into a single **robotic score** that is logged but never used to grade the puzzle.
It is deliberately orthogonal to the pass rate: a model can solve every slider and still be
trivially detectable, which is the point.

## The agent

**Tools.** `click`, `drag`, `type`, `press_and_hold`, `submit`, `reflect`, plus three sandboxes:

- `run_js` — JavaScript in the page, for reading layout the screenshot cannot show
- `run_python` — computation
- `run_image_code` — numpy/Pillow over the **live screenshot**

Every tool call is validated against a JSON schema before it runs, and coordinates are clamped to
the viewport.

**Image-analysis skills.** Rather than have the model write fragile array code every turn,
`run_image_code`'s namespace is preloaded with tested measurement helpers — `count_blobs(img)`,
`find_gap_x(img, y0, y1)` and others. Each takes the live screenshot and returns a number read
from the pixels. None of them returns an answer or a puzzle constant.

**Experience RAG.** A corpus of worked examples and lessons; the most relevant few are retrieved
into the prompt each step (sentence-transformers when available, word overlap otherwise).

**Memory and reflection.** Attempts within a trial share memory, so attempt 3 knows what attempts
1 and 2 tried and why they failed. With `enable_reflection` on, the model writes an explicit
reflection between attempts. This is the single largest source of improvement across attempts, and
it is inspectable — Section 21 shows everything a model accumulated and reflected on for a given
type.

**Self-repair.** Agent v3 adds an extended toolbox, experience retrieval and self-repair on top of
v1's loop, additively — v1 behaviour is still reachable.

**Annotated vision.** The screenshot the model sees carries a coordinate grid, Set-of-Mark labels
([Yang et al., ICLR 2024](https://arxiv.org/abs/2310.11441)) and arrows tracing where previous
attempts went wrong.

## Benchmark modes

`CONFIG["bench_mode"]` sets how much help the model gets — the three points define a spectrum:

| Mode | The model gets |
|---|---|
| `vision_only` | the screenshot. **The default, and the honest number.** |
| `dom_assisted` | the screenshot plus page structure |
| `agentic_oracle` | an upper bound for the harness, to separate harness failure from model failure |

Key knobs:

```python
CONFIG["trials_per_captcha"]  = 3      # distinct seeds per family
CONFIG["max_attempts"]        = 3      # attempts per trial; memory accumulates
CONFIG["max_steps"]           = 12     # tool calls per attempt
CONFIG["reveal_cookbook"]     = False  # True = spoon-fed mode; keep it False
CONFIG["pin_seeds"]           = False  # True = identical puzzles across models
CONFIG["max_tokens_per_step"] = None   # cap how long the model may think
CONFIG["compact_prompt"]      = False  # True for small-context (~8k) models
```

**On speed.** The dominant cost is LLM generation. At ~45 tok/s a reply with 4000 reasoning tokens
takes ~90 s, and an episode can be ten or more replies. `max_tokens_per_step` trades depth on hard
puzzles for much faster runs: `32768` is full depth, `8192` is a good balance, `4096` is fast.

## Repository contents

| File | Status |
|---|---|
| [`captcha_benchmark_paper_v2.ipynb`](captcha_benchmark_paper_v2.ipynb) | **current.** All 28 families, agent v3, invisible Turnstile, routing policy. Use this one. |
| [`captcha_benchmark_paper_v1.ipynb`](captcha_benchmark_paper_v1.ipynb) | kept for reference — the earlier 18-family version, before the coverage-gap families and agent v3. Not maintained. |

## Installation

### 1. Python environment

Developed against the `ai-research` conda environment (Python 3.11):

```bash
conda activate ai-research
```

From scratch:

```bash
conda create -n ai-research python=3.11 -y
conda activate ai-research
pip install -r requirements.txt
playwright install chromium
```

Section 1 of the notebook also installs everything with a single `%pip install`. Versions are
pinned because an old numpy (missing `sliding_window_view`) silently breaks the model's generated
image code; keep `numpy >= 1.24`.

### 2. A vision model in LM Studio

Install [LM Studio](https://lmstudio.ai/), start the local server on port `1234`, and load a
**multimodal** model — it has to see images. Good choices: Qwen-VL, Gemma 3, MiniCPM-V, InternVL.

```python
LM_STUDIO_URL = "http://localhost:1234/v1"
MODEL_HINT    = "auto"   # or the exact id LM Studio serves
```

The health check in Section 17.1 prints every loaded model and drives the browser end to end, so
you know the harness works before spending an hour on a benchmark.

## Quick start

```bash
conda activate ai-research
jupyter lab captcha_benchmark_paper_v2.ipynb
```

Run the cells in order. For a fast smoke test first:

```python
CONFIG["captchas"] = ["checkbox", "slider_puzzle", "image_grid"]
CONFIG["trials_per_captcha"] = 1
CONFIG["max_tokens_per_step"] = 4096
```

Then run Section 18. Logs are never cleared: every step prints the model's reasoning, the tool it
chose, the validation result and the grade.

## Comparing several models

Load several models in LM Studio, list their ids, and pin the seeds so every model sees **identical
puzzles**:

```python
MODELS_TO_BENCHMARK = ["qwen/qwen3-vl-30b", "google/gemma-3-27b"]
CONFIG["pin_seeds"]  = True
CONFIG["seed_master"] = 20250831
```

One pass loops over them. Without `pin_seeds`, per-model differences are partly puzzle luck.

## What a run produces

All artefacts live beside the notebook in `WORK_DIR`:

```
captchas/                the rendered fixture HTML
logs/                    per-episode logs: reasoning, tool calls, validation, grades
results/                 per-episode JSON, the summary tables, the charts
memory/                  accumulated memory and reflections, per model and type
trajectories/            schema-v2 trajectories for the routing-policy trainer
execution_errors.jsonl   every sandbox error, with the code that caused it
*.gif                    one animated GIF per episode, step by step
```

The GIFs are separate from the text logs and are the fastest way to see *why* a model failed — a
slider released three pixels early looks like nothing in a log and is obvious in an animation.

The summary tables report, per model and per family: pass rate, mean steps to solve, mean attempts,
token usage, truncation rate, and the mean robotic score.

## The routing policy

Section 16/20 reads the schema-v2 trajectories and fits, **per CAPTCHA family**, a
gradient-boosted model over which tool sequence tends to succeed. It needs no GPU and runs
automatically after the benchmark. The result is `learned_policy.json`, loaded back in Section 8 to
provide tool-ordering hints and a motor summary in the prompt. On the first run there is no policy
and the notebook says so rather than pretending.

`MIN_EPISODES_TO_LEARN` guards it: with 3 trials × 3 attempts a family yields ~9 episodes, which
is over the threshold. Below it you get the frequency baseline and a printed warning, not a
confident model.

Section 22 is an **optional** LoRA fine-tune on the collected trajectories, off by default and the
only part that wants a GPU. The routing policy is the mandatory learner; the fine-tune is not.

## Notebook layout

| Section | Contents |
|---|---|
| 1–2 | Dependencies, `CONFIG`, directories, LM Studio endpoint |
| 3 | Motor-control metrics (Cerno) |
| 4 | The 28 fixtures — base, real-world, coverage-gap, invisible Turnstile |
| 5 | Registry: fixture ↔ goal-only instruction ↔ published human/VLM pass rates |
| 6 | The embedded verification server, and manual fixture inspection |
| 7 | Retrieval corpus and retriever |
| 8 | Learned-policy loader |
| 8a | Image-analysis skill functions |
| 9 | Tool schemas, JSON-schema validation, `run_js` / `run_python` sandboxes |
| 10 | LM Studio client — native tool calling plus a prompt-based fallback |
| 11 | Browser agent on its dedicated worker thread |
| 12 | Screenshot annotator — grid, Set-of-Mark, failure traces |
| 16 | Routing-policy training pipeline |
| 16a | Agent v3 — extended toolbox, experience RAG, self-repair |
| 17 | Instantiate the agent; system health check |
| 18 | **Run the benchmark** |
| 18.1 | Watch the episode GIFs |
| 19 | Results analysis |
| 20 | Train the routing policy on all collected data |
| 21 | Inspect memory and reflections |
| 22 | Optional: LoRA fine-tune on the trajectories |
| 23 | Cleanup |

## Scope and responsible use

This is **defensive and measurement research**. It exists to answer a question CAPTCHA designers
need answered: which challenge families still separate humans from current multimodal agents, and
which do not.

What the notebook does:

- renders its own fixtures locally and serves them from `127.0.0.1`
- grades them with its own embedded server
- records how robotic the agent's motion is, which is **the signal a defender would use**

What the notebook does **not** do, and will not accept contributions to do:

- contact, solve, or interact with any live CAPTCHA provider or deployed site
- ship a bypass for a production anti-bot system
- integrate a paid CAPTCHA-solving service

The `turnstile_invisible` family and the whole motor-control section exist precisely because
solving the *visible* puzzle is the part that has stopped being a good defence. The finding this
benchmark supports is that behavioural signals survive where visual puzzles do not — which is
useful to defenders.

If you are a CAPTCHA vendor and want a family added so you can measure it, open an issue.

## References

- **MCA-Bench** — [arXiv:2506.05982](https://arxiv.org/abs/2506.05982). The 14 base families and
  the published human/VLM pass rates the registry records.
- **Open CaptchaWorld** — [arXiv:2405.07496](https://arxiv.org/abs/2405.07496). Interactive
  CAPTCHA evaluation for multimodal agents.
- **Set-of-Mark prompting** — Yang et al., ICLR 2024,
  [arXiv:2310.11441](https://arxiv.org/abs/2310.11441). The labelling scheme used by the annotator.
- **Cerno** (PlawIO) — the motor-control CAPTCHA the six motion metrics are derived from.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The most valuable additions are new CAPTCHA families — a
fixture, a registry entry, and a server-side grader that replays the seed — and better motor
metrics.

## Related projects

- [oncology-paper-finder](https://github.com/kronenflex/oncology-paper-finder) — an autonomous
  literature-search agent with learned tool selection
- [autonomous-systematic-review](https://github.com/kronenflex/autonomous-systematic-review) — a
  full PRISMA 2020 systematic review that runs unattended

## License

[MIT](LICENSE) © 2026 Diego Pavez

## Citation

If this benchmark contributes to a publication, please cite it — see [CITATION.cff](CITATION.cff).
