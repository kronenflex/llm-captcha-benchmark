# Contributing

Thanks for considering a contribution. Please read the scope rules first — this project measures
CAPTCHA-solving capability and deliberately refuses to become a bypass tool.

## Scope rules (not negotiable)

Pull requests that do any of the following will be closed:

- contacting, solving, or interacting with a **live** CAPTCHA provider or a deployed site
- shipping a bypass, token harvester, or replay attack against a production anti-bot system
- integrating a paid CAPTCHA-solving service
- removing or weakening the server-side grading so the client's claim is trusted

Everything in this benchmark is a **local synthetic fixture** served from `127.0.0.1` and graded by
the notebook's own server. That boundary is the reason the project can exist in the open.

## Adding a CAPTCHA family

This is the highest-value contribution. Four things are required:

1. **The fixture** — a self-contained HTML string in Section 4. It must
   - seed a `mulberry32` PRNG from `?pu=<seed>`, and make **the answer-determining draws first**,
     before any decorative rendering, so the server can replay a short exact prefix;
   - expose **only** the instruction and the picture. No answer, no geometry, no formula reachable
     from the DOM or from JavaScript;
   - record raw pointer events, so motor analysis works on it.
2. **A registry entry** in Section 5 — fixture, a goal-only instruction, and, if the family appears
   in MCA-Bench or another published benchmark, the reported human and VLM pass rates.
3. **A server-side grader** in Section 6 that replays the seed and recomputes the answer. If
   grading needs the client to tell the server anything beyond its submitted action, the fixture is
   wrong.
4. **Proof it is solvable by a human.** Open it yourself via the links Section 6.1 prints and pass
   it. A family nobody can solve measures nothing.

## Other valuable work

- **Motor metrics.** Section 3 implements six cues from Cerno. More cues, or a better-calibrated
  combination, directly improve the defensive value of the benchmark. Bring numbers on synthetic
  human-like vs script-like paths.
- **Image-analysis skills.** Section 8a. A helper must read a number *from the pixels* of the live
  screenshot and must never return an answer or a puzzle constant.
- **Harness honesty.** If you can show a failure is the harness's fault rather than the model's,
  that is a bug worth reporting. `bench_mode="agentic_oracle"` exists to separate the two.
- **Small-model support.** `compact_prompt` and `vision_max_width` make the benchmark usable on
  ~8k-context models. Improvements there widen who can run it.

## Reporting results

If you benchmark a model and want to share numbers, please include:

- the exact model id and quantisation
- `bench_mode`, `trials_per_captcha`, `max_attempts`, `max_steps`, `max_tokens_per_step`
- `reveal_cookbook` (it must be `False` for a comparable number)
- `pin_seeds` and `seed_master` — without pinned seeds, cross-model differences are partly luck
- the mean robotic score alongside the pass rate

A pass rate reported without `reveal_cookbook=False` and pinned seeds is not comparable to anyone
else's.

## Working with the notebook in git

Notebook diffs are noisy. Before committing:

```bash
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --inplace captcha_benchmark_paper_v2.ipynb
```

Keep outputs only where they are the evidence of a run the README refers to, and never commit
outputs containing paths from your machine or anything identifying. `nbdime` makes review sane:

```bash
pip install nbdime
nbdime config-git --enable --global
```

## Reporting a bug

Include your OS, the output of the Section 17.1 health check, the CAPTCHA family, and the relevant
block from `logs/`. For a sandbox error, `execution_errors.jsonl` already contains the code that
failed — attach the entry.

## Reporting a security issue

Please do not open a public issue. Open a
[private security advisory](../../security/advisories/new) instead.
