Paste into Codex at the repository root.

---

You are implementing Milestone 1 of `assay`, an ablation benchmark that measures
whether Dovetail's `judge-composition` skill improves an agent's ability to
identify a planted defect. Read `SPEC.md` first and treat it as binding. Where
this prompt and the spec disagree, the spec wins; flag the conflict rather than
resolving it silently.

Stack: Python 3.11+, Inspect AI (`inspect-ai`, developed against 0.3.263),
`uv` for dependency management. No other runtime dependencies without asking.

Build, in this order:

1. `src/assay/arms.py`. Load `judge-composition` from a Dovetail checkout path
   supplied as a task parameter. Resolve `SKILL.md`, then `skill.md`, then
   `README.md`. Raise on a missing directory, a missing skill file, or an empty
   one, with an error naming the path tried and the clone command. Capture the
   checkout's git revision and return it alongside the text so it reaches the
   eval log. Assemble the two system prompts, with the response protocol block
   byte-identical between arms.

2. `artifacts/`. One file per artifact, the six defect classes and two controls
   named in the spec. Write them yourself rather than lifting public code, and
   keep each one short enough to review by eye. Each defect artifact carries
   exactly one planted defect. If you find yourself planting a second, split it
   into two artifacts.

3. `src/assay/dataset.py`. Load artifacts from disk into Inspect `Sample`
   objects carrying `kind`, `planted`, and `requires` per the dataset contract.
   For each defect, supply the alternative surface forms a correct reviewer might
   plausibly use. Be generous here: a false INCORRECT is a worse failure than a
   permissive match, because it silently deflates the arm being measured.

4. `src/assay/scoring.py`. The deterministic scorer and the four metrics, exactly
   as specified. Distinguish the four failure modes in score metadata:
   unparseable, miss, false positive, wrong defect. Do not blend them into one
   accuracy number in the returned metrics.

5. `src/assay/tasks.py`. One parameterized Inspect task taking `arm`
   ("baseline" or "skill") and the Dovetail path. Reject any other arm value.
   Name the task per arm so logs are distinguishable.

6. Tests. The scorer is the part that must not be wrong: cover a correct hit, a
   wrong-defect hit, a miss, a false positive, and an unparseable response, plus
   the three skill-loading failures. Verify both arms end with an identical
   protocol block.

Verification before you report done:

```
uv run pytest
uv run inspect eval src/assay/tasks.py -T arm=baseline --model mockllm/model
```

The mock model produces no parseable verdicts, so an accuracy of 0.0 with every
sample recorded as unparseable is the correct outcome and confirms the scorer is
not giving away credit. Any other result under mockllm is a bug.

Constraints:

- No model-graded scoring anywhere, at any milestone.
- Do not reproduce the skill text in this repository. It is read at run time.
- Do not publish, commit, or write into `docs/RESULTS.md` any delta from a
  mockllm run or from a single-epoch pilot.
- Do not widen a `requires` group in response to seeing a model fail on it
  during a run. Record the case, finish the run, widen between runs.

Stop and ask rather than guessing on any item under "Open decisions" in the
spec. Milestone 1 is a pilot: the deliverable is a harness whose failures are
legible, not a number.
