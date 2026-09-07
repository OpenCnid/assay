# assay

Agent skills claim to make models better at checking their own work. Almost none
of them come with a number. Assay is the harness that produces one.

It measures a single skill at a time by ablation: the same artifacts, the same
model, the same response protocol, run with and without the skill loaded. The
gap between the two runs is the claim. Everything else here exists to stop that
gap from being an artifact of something other than the skill.

The first subject is [Dovetail](https://github.com/OpenCnid/dovetail), which
ships eight skills for Claude Code and Codex and has never published a delta for
any of them. Assay measures two: `judge-composition`, whether the skill helps an
agent identify a planted defect, and `self-play`, whether it helps surface a
latent flaw in a design.

## What it is careful about

**Controls, not just defects.** A judge that shouts DEFECT at everything scores
100% on a defect-only set. One in four artifacts is genuinely clean, and false
positive rate is reported next to recall. A skill that raises both equally has
not improved judgement, it has made the model more suspicious.

**A placebo arm.** The skill arm's prompt is longer than baseline, so any
improvement could be prompt length rather than skill content. A third arm
carries length-matched generic review advice with none of the skill's structure.
The headline number is skill over placebo, never skill over baseline alone.

**Deterministic scoring.** No model grades the model, at any milestone. A
defect sample counts only when the model names the planted defect, not merely
some defect. A grader would be the largest uncontrolled variable in an
experiment whose whole output is a small delta.

**The skill is never copied in.** Skill text is read at run time from a Dovetail
checkout, and the resolved git revision is recorded in every log. A result that
cannot name the revision it measured is not a result. Loading fails loudly
rather than silently producing an empty preamble, which would make both arms
identical and report a delta of zero for entirely the wrong reason.

**A standing critique.** Every published number in `docs/RESULTS.md` has a
matching entry in `docs/CRITIQUE.md` recording what it does not establish.
Known and only partly controlled: the same author writes the artifacts, the
matching rules, and reads the skill under test.

## Status

Specification. No implementation yet. `SPEC.md` is the contract,
`IMPLEMENTATION_PROMPT.md` is the build order, and `docs/PROGRESS.md` carries
the running record once work starts.

## Running it

Requires Python 3.11+, `uv`, and a Dovetail checkout.

```
uv run inspect eval src/assay/tasks.py \
  -T suite=judge -T arm=skill -T dovetail=/path/to/dovetail \
  --model <provider>/<model>
```

Then the same command with `-T arm=baseline` and `-T arm=placebo`. Compare with
`uv run python -m assay.report`.

A run against `mockllm/model` should score 0.0 with every sample recorded
unparseable. That is the scorer confirming it gives away nothing, and any other
result under mockllm is a bug.

## What a result here does not mean

A delta on two skills is not a claim about Dovetail. A delta on one model is a
claim about that model. A delta below the noise floor of the dataset is not a
finding, and the floor is computed before the deltas are read. If the skill
turns out not to help, that is the number, and it gets published the same way a
positive one would.

Built on [Inspect AI](https://inspect.aisi.org.uk/), the open-source evaluation
framework from the UK AI Security Institute.

MIT.
