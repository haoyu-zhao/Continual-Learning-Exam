# Run and evaluate CLE Lite

[Back to the dataset README](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite) · [Runtime script](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite/blob/main/runtime/cle_lite.py) · [Verification record](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite/blob/main/runtime/verification.json)

This guide runs the Conversational Agent Evaluation task through all five levels and scores the submitted work. It preserves the setup and protocol from the maintainers' verified Linux run. In this guide, **solver** means the agent being evaluated, and **judge** means the grader that assesses its deliverables.

The runtime script is pinned to revision `026143c454983014232b83318a7c2db1578b51e0`; the task data is pinned separately below. Record both revisions when reporting an experiment.

## Run the example

The [`runtime/cle_lite.py`](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite/blob/main/runtime/cle_lite.py) script downloads a task, prepares its environment, runs the agent, and scores its submissions. It uses the grader packaged with the task and checks that the task files and grader match the recorded release. Run evaluations through this script. The bundled `tests/test.sh` only signals to the host grader; running it directly does not score a submission.

### Prerequisites

- Approved access to this dataset and a Hugging Face login for that account.
- A Linux host with Docker, the Docker Compose plugin, `curl`, and [uv](https://docs.astral.sh/uv/getting-started/installation/). The commands below were tested on Linux x86-64.
- Node.js 22 or later and a Codex login with access to the selected model. Both the evaluated agent and the model used for grading consume inference usage.
- Enough disk space for the inputs, downloaded-content cache, solver variant, Docker images and result artifacts. Reserve at least **30 GB for the example task**; other tasks can require more. A complete five-level run and grading can take several hours.

### 1. Authenticate and download one task

```bash
uv tool install huggingface_hub
hf auth login

# Fetch the runtime adapter and its Linux judge-container recipe.
hf download Continual-Learning-Exam/CLE-Lite \
    --repo-type dataset \
    --revision 026143c454983014232b83318a7c2db1578b51e0 \
    --include 'runtime/*' --local-dir .

TASK=019-computer-research-scientist
DATA_REVISION=04014384175aeb218c4ba7188612b1c76e87d3fc
BUNDLE="./CLE-Lite/$TASK/harbor"
RUNTIME_ENV="./.cle-runtime-$TASK"

uv run --no-project --with huggingface_hub python runtime/cle_lite.py download \
    --task "$TASK" --directory ./CLE-Lite --revision "$DATA_REVISION"
```

This fetches only the selected task's `harbor/task/`, `harbor/private/` and freeze manifest. It verifies every content hash and restores the file permissions and timestamps required by the frozen manifest. Repeating the download reuses verified files. An ordinary `hf download` remains useful for inspecting the archive, but it does not restore these metadata for an integrity-checked evaluation.

The example is **Conversational Agent Evaluation**, with levels `t1`–`t5`. Its input workspace is approximately 1.4 GB. Keep `DATA_REVISION` in experiment records; a later README or adapter revision does not change this pinned task snapshot.

### 2. Prepare the host and grading container

```bash
uv run --no-project python runtime/cle_lite.py setup \
    --bundle "$BUNDLE" --env "$RUNTIME_ENV"

npm install --global --prefix ./.cle-tools @openai/codex@0.160.0
export PATH="$(pwd)/.cle-tools/bin:$PATH"
codex login --device-auth

docker build -t cle-lite-codex-worker:0.160.0 \
    -f runtime/codex-worker.Dockerfile runtime

export RSI_CODEX_DOCKER_IMAGE=cle-lite-codex-worker:0.160.0
export RSI_CODEX_AUTH_FILE="$HOME/.codex/auth.json"
export RSI_CODEX_TRANSPORT=https
export RSI_CODEX_REASONING_EFFORT=high
```

`setup` reads `private/grader/identity.json` and installs the recorded Python patch version and host packages alongside **Harbor 0.22.0** in a dedicated environment. Tasks in this release record either Python 3.12.13 or 3.13.15, so keep a separate environment for each task. For this task, the host interpreter is Python 3.12.13. Linux model-based judge workers run in the Docker image built above; its interpreter is separate from the verified host runtime.

For an existing Codex login, skip `codex login` and point `RSI_CODEX_AUTH_FILE` at that login's `auth.json`. Keep the file private and do not commit it. The solver receives the login through Harbor's credential upload; the judge receives a read-only mount in its worker container.

### 3. Run all five levels and score the submission

```bash
uv run --no-project python runtime/cle_lite.py run \
    --bundle "$BUNDLE" --env "$RUNTIME_ENV" \
    --agent codex --model openai/gpt-6.1-sol \
    --judge-backend codex --judge-model gpt-6.1-sol \
    --codex-auth "$RSI_CODEX_AUTH_FILE" \
    --results ./results/019-gpt-6.1-sol-01
```

Use a **new results directory** for each invocation. The script retains the original grading code, checks, references, and file inventory. When the bundle was built for Claude Code, it creates a separately recorded Codex solver variant that changes the solver installation and inference-host allowlist. The original frozen bundle remains the source of grading identity.

The run starts three trials concurrently:

| Trial | Purpose |
| :--- | :--- |
| `oracle` | Score the reference submission to check that the grader works. |
| `nop` | Submit no work to check that an empty submission fails. |
| Solver | Run the agent being evaluated and score its deliverables. |

The packaged grader accepts the oracle only if every failed check, if any, is marked unstable. The nop must pass no checks. These rules do not change the raw rewards. If either control fails its validity checks, the run is invalid. Report the solver trial as the model result; the two controls are diagnostics.

Harbor continues the solver's trajectory across `t1`–`t5`, collects each level's output, and invokes the host judge after the final level. The configured per-level solver budget is 60 minutes. The packaged network guard restricts solver networking to the inference proxy; web search, apps, browser and computer-use tools are disabled.

### 4. Inspect the result

```bash
python3 - <<'PY'
import json
from pathlib import Path
run = json.loads(Path('results/019-gpt-6.1-sol-01/run.json').read_text())
print('Run state:', run['state'])
for name, control in run['controls'].items():
    print('Control:', name, control['status'], control.get('score'))
for trial in run['trials']:
    print('Solver:', trial['status'], trial['passed'], '/', trial['total'])
    print('All-checks reward:', trial['score'])
    print('Check fraction:', trial['passed'] / trial['total'] if trial['total'] else 0)
    print('Levels:', json.dumps(trial.get('ladder'), indent=2))
    print('Detailed judge result:', trial['judge_result'])
PY
```

`run.json` contains execution validity, control records, check scores, per-level results and the detailed judge-result path. Each trial directory also retains its Harbor result, collected deliverables, solver trajectory and verification logs. A `complete` run means that evaluation completed; it does not mean that the model passed every check.

### Score an existing submission

You can score an existing submission without rerunning the agent. Use the environment and judge settings from the setup above. The example below reads the recorded task variant and submission path from the completed run, then scores the same files again:

```bash
EVALUATED_BUNDLE=$(python3 - <<'PYTHON'
import json
from pathlib import Path
run = json.loads(Path('results/019-gpt-6.1-sol-01/run.json').read_text())
print(run['bundle'])
PYTHON
)
SUBMISSION=$(python3 - <<'PYTHON'
import json
from pathlib import Path
run = json.loads(Path('results/019-gpt-6.1-sol-01/run.json').read_text())
print(Path(run['trials'][0]['trial_dir']) / 'artifacts/app/output')
PYTHON
)
uv run --no-project python runtime/cle_lite.py grade \
    --bundle "$EVALUATED_BUNDLE" --env "$RUNTIME_ENV" \
    --output "$SUBMISSION" \
    --judge-backend codex --judge-model gpt-6.1-sol \
    --results ./results/019-regrade-01
```

The command writes the original detailed result to `judge-result.json` and a compact result, including per-level checks and the post-hoc gated count, to `summary.json`. The submission must contain each level's deliverables under `t1/` through `t5/`, as specified by that task. This command evaluates the supplied files; it does not establish that they came from an isolated benchmark run. The recorded solver trial's `judge_result` remains the result of the complete run above.

## Understand the scores

The adapter calls the scoring implementation archived under `harbor/private/grader/src/`. Its identity and dependency versions are checked against `private/grader/identity.json`; task and grading files are checked against `freeze.json`. Checks use the packaged rubric, ground truth and reference material, combining file presence, numeric comparisons, code reruns, figure checks and model-based judgments as configured by the task.

Report the following separately:

- **Task reward**: `score` and the Harbor reward are binary: one only when every check passes, otherwise zero.
- **Check fraction**: `passed / total`, also recorded as the verifier reward's `fraction`. This is a diagnostic measure over the task's checks.
- **Level completion**: a level's `whole` value is true only when all of its checks pass.
- **Gated level count**: `ladder.gated` counts consecutive fully passed levels starting at `t1`, out of `ladder.of` (five).

This Harbor adapter executes all five levels and calculates gated completion **afterward**. It does not stop execution immediately after a failed level. Describe this protocol when reporting results; do not equate its task reward, check fraction or post-hoc gated count with the full CLE leaderboard's stop-on-failure experiment.

The oracle must satisfy the sealed scorer's control rules, and the nop must pass no checks. Model-based judgments can vary between runs; record the judge model and reasoning effort along with the solver model, task revision, grader identity and runtime versions. Keep rubrics, ground truth, references and judge-only setup outside the solver's accessible workspace throughout evaluation.

### Verified end-to-end example

On 2026-10-21, the commands above were validated on Linux x86-64 with task `019-computer-research-scientist` at revision `04014384175aeb218c4ba7188612b1c76e87d3fc`. All five solver stages ran and the original host grader evaluated all 159 checks. Both controls satisfied the original control rules: oracle **159/159** (raw reward 1) and nop **0/159** (reward 0).

The real `gpt-6.1-sol` solver passed **158/159** checks, with task reward **0** and post-hoc gated completion **3/5**. The final stage (`t5`) reached its configured 60-minute agent budget; the original runner recorded the trial as valid with `execution = "budget_exhausted"` and graded the collected deliverables. Total agent time across the five stages was approximately 185 minutes. These are the observed results of a runtime validation run. See [`runtime/verification.json`](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite/blob/main/runtime/verification.json) for the frozen grader identity, runtime versions and aggregate evidence; benchmark answers and private grading logs are not republished there.

## Troubleshooting

| Problem | What to check |
| :--- | :--- |
| Download returns `401` or `403` | Confirm that dataset access was approved. Run `hf auth login` with the approved account and a token that can read this dataset. |
| File hashes, permissions, or timestamps do not match | Download the bundle with `runtime/cle_lite.py download`. A plain `hf download` does not restore all metadata required by the task's integrity checks. |
| The runtime or dependency check fails | Run `setup` for that task in its own environment. Tasks can require different Python versions. |
| The verifier says the host judge is required | Run through `runtime/cle_lite.py run` after `setup`; do not invoke `tests/test.sh` directly. |
| A results directory already exists | Use a new `--results` directory for each run or regrade. |
| The reward is zero despite many passing checks | Task reward is binary. Inspect the check fraction and per-level results to see which checks failed. |

For help, [open a GitHub issue](https://github.com/haoyu-zhao/Continual-Learning-Exam/issues) with the task, data and runtime revisions, model names, and error message. Keep credentials, reference answers, and private grading logs out of public reports.
