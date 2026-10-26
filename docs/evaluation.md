# Run and evaluate CLE Lite

[Dataset homepage](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite) · [Project README](../README.md#quick-start) · [Custom task lists](#custom-task-lists) · [Scoring](#understand-the-scores)

Run a single task or an ordered task sequence, then inspect the agent's submissions and scores. The examples below use task **008: Operations Research Analyst** and pin the runtime and task data to specific revisions.

## Choose an evaluation mode

CLE Lite supports **two evaluation modes: Direct and CLE**. Both use the same tasks and grading rubrics. The difference is when submissions are graded and whether the agent must pass a level before continuing.

- **Direct — attempt all five levels, then grade.** The agent works through all five levels without receiving grading feedback during execution. The runtime saves each level's submission and grades those saved versions afterward. Tasks are evaluated independently. Choose this mode to examine performance across every level.
- **CLE — pass each level before continuing.** Each submission is graded immediately. The agent advances only after passing every rubric check for the current level. If it fails after using its allowed submissions, the task ends and unreached levels count as failed. Choose this mode to follow CLE's level-by-level evaluation protocol.

CLE mode provides two submission settings:

| Setting | Submissions and feedback |
| :--- | :--- |
| **Trial 1** | One submission per level, with no diagnostic feedback. |
| **Trial 2** | After a failed first submission, the agent receives feedback identifying unmet rubric categories and may revise and resubmit once. Both submissions share the level's execution time budget. |

Start with **`run`** to evaluate a single task. Use **`stream --setting cle`** to evaluate an ordered sequence of tasks while retaining the agent's conversation and configured memory between tasks. Steps 1–3 cover a single task; step 4 shows the optional task stream.

<a name="quick-start"></a>

## Run the example

### Before you start

The **solver** is the agent that performs the task. The **judge** evaluates its submissions against the grading rubrics. The packaged runtime currently runs the solver through Codex; the examples use GPT-6.1-Sol. You select the judge separately in step 2.

You will need:

- **Environment:** Linux x86-64, Bash, Docker with Compose, `curl`, Python 3.10+, `uv`, and Node.js 22+.
- **Access:** approved access to [CLE Lite Full](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite-Full) and a Codex login with access to the solver model. The OpenRouter judge option also requires an OpenRouter API key.
- **Resources:** at least **30 GB of disk space for a single example task**, with additional space for task streams. Evaluations using the full execution budget can take hours. Both the solver and the model-based judge consume inference usage.

Run the commands in **Bash, in the same shell and working directory**, so the paths and variables defined during setup remain available.

### 1. Download and prepare a task

```bash
uv tool install huggingface_hub
hf auth login
hf download Continual-Learning-Exam/CLE-Lite-Full \
    --repo-type dataset --revision 26ab77e5085476d59dc8c243b928e5b8ac490ef4 \
    --include 'runtime/*' --local-dir .

TASK=008-operations-research-analyst
DATA_REVISION=803301ec01b26ab594a5ab3cdcd855a14dcc26e1
BUNDLE="./CLE-Lite/$TASK/harbor"
RUNTIME_ENV="./.cle-runtime-$TASK"

uv run --no-project --with huggingface_hub python runtime/cle_lite.py download \
    --task "$TASK" --directory ./CLE-Lite --revision "$DATA_REVISION"
python3 runtime/cle_lite.py setup --bundle "$BUNDLE" --env "$RUNTIME_ENV"

npm install --global --prefix ./.cle-tools @openai/codex@0.160.0
export PATH="$(pwd)/.cle-tools/bin:$PATH"
codex login --device-auth

docker build -t cle-lite-codex-worker:0.160.0-cle2 \
    -f runtime/codex-worker.Dockerfile runtime
export CLE_CODEX_DOCKER_IMAGE=cle-lite-codex-worker:0.160.0-cle2
export CLE_CODEX_AUTH_FILE="$HOME/.codex/auth.json"
export CLE_CODEX_TRANSPORT=https
```

The first download retrieves the runtime scripts; the `download` command then retrieves the selected task, verifies its recorded file hashes, and restores required permissions and timestamps. `setup` installs the task's recorded Python and package versions with Harbor 0.22.0. The runtime and task data are pinned to separate revisions; record both when reporting results.

If already logged in to Codex, skip `codex login` and point `CLE_CODEX_AUTH_FILE` at the existing login file.

### 2. Configure the judge

**Choose one judge configuration.** This choice is independent of Direct or CLE mode.

**DeepSeek V4 Flash via OpenRouter**

Use an OpenRouter API key to access [DeepSeek V4 Flash](https://openrouter.ai/deepseek/deepseek-v4-flash):

```bash
read -r -s -p 'OpenRouter API key: ' OPENROUTER_API_KEY
export OPENROUTER_API_KEY
JUDGE_BACKEND=openrouter
JUDGE_MODEL=deepseek/deepseek-v4-flash
```

**GPT-6.1-Sol via Codex**

This option uses the Codex login configured in step 1:

```bash
JUDGE_BACKEND=codex
JUDGE_MODEL=gpt-6.1-sol
export CLE_CODEX_REASONING_EFFORT=low
```

The `low` effort setting applies to both the solver and the Codex judge. To use the runtime defaults instead, run `unset CLE_CODEX_REASONING_EFFORT` before evaluation.

### 3. Run one task

**Choose one command below.** Each command starts a separate evaluation of task 008. Use a new `--results` directory every time you run an evaluation.

**Direct**

```bash
python3 runtime/cle_lite.py run \
    --setting direct --bundle "$BUNDLE" --env "$RUNTIME_ENV" \
    --agent codex --model openai/gpt-6.1-sol \
    --judge-backend "$JUDGE_BACKEND" --judge-model "$JUDGE_MODEL" \
    --codex-auth "$CLE_CODEX_AUTH_FILE" --results ./results/direct-008-01
```

**CLE · Trial 1**

```bash
python3 runtime/cle_lite.py run \
    --setting cle --trial 1 --bundle "$BUNDLE" --env "$RUNTIME_ENV" \
    --agent codex --model openai/gpt-6.1-sol \
    --judge-backend "$JUDGE_BACKEND" --judge-model "$JUDGE_MODEL" \
    --codex-auth "$CLE_CODEX_AUTH_FILE" --results ./results/cle-trial1-008-01
```

**CLE · Trial 2**

```bash
python3 runtime/cle_lite.py run \
    --setting cle --trial 2 --bundle "$BUNDLE" --env "$RUNTIME_ENV" \
    --agent codex --model openai/gpt-6.1-sol \
    --judge-backend "$JUDGE_BACKEND" --judge-model "$JUDGE_MODEL" \
    --codex-auth "$CLE_CODEX_AUTH_FILE" --results ./results/cle-trial2-008-01
```

Inspect these files in the selected results directory:

| File | Contents |
| :--- | :--- |
| `run.json` | Submissions, grading decisions, feedback delivered to the solver, and execution events. |
| `summary.json` | Completion and cumulative level pass rates. Rates are fractions from 0 to 1; multiply by 100 for percentages. |

The examples use each task's packaged execution budget. Set `--agent-timeout-sec SECONDS` to change the per-level budget; the value is saved in the results.

### 4. Optional: evaluate an ordered task sequence

Use a CLE task stream to evaluate how the agent carries experience from one task to the next. The example task list runs **008 → 019**. The agent retains its conversation and configured memory, while each task starts with its own files and application services.

<details>
<summary><strong>🔁 Show task preparation and stream commands</strong></summary>

After completing steps 1 and 2, prepare both tasks listed in `runtime/lite-example-tasks.json`:

```bash
for TASK in 008-operations-research-analyst 019-computer-research-scientist; do
    uv run --no-project --with huggingface_hub python runtime/cle_lite.py download \
        --task "$TASK" --directory ./CLE-Lite --revision "$DATA_REVISION"
    python3 runtime/cle_lite.py setup \
        --bundle "./CLE-Lite/$TASK/harbor" --env "./.cle-runtime-$TASK"
done
```

Choose one Trial. Each Trial starts a separate stream with its own initial agent state and results directory.

**CLE stream · Trial 1**

```bash
python3 runtime/cle_lite.py stream \
    --setting cle --trial 1 --task-list runtime/lite-example-tasks.json \
    --agent codex --model openai/gpt-6.1-sol \
    --judge-backend "$JUDGE_BACKEND" --judge-model "$JUDGE_MODEL" \
    --codex-auth "$CLE_CODEX_AUTH_FILE" --results ./results/cle-stream-trial1-01
```

**CLE stream · Trial 2**

```bash
python3 runtime/cle_lite.py stream \
    --setting cle --trial 2 --task-list runtime/lite-example-tasks.json \
    --agent codex --model openai/gpt-6.1-sol \
    --judge-backend "$JUDGE_BACKEND" --judge-model "$JUDGE_MODEL" \
    --codex-auth "$CLE_CODEX_AUTH_FILE" --results ./results/cle-stream-trial2-01
```

Tasks run in the order specified by the task-list file. Its paths are relative to that file. See [Custom task lists](#custom-task-lists) to choose your own sequence.

</details>

### Custom task lists

Create a JSON file with one `tasks` entry for each task you want to evaluate. Each entry names the task's `harbor/` bundle and the environment created by `setup`. For example, save this as `runtime/my-tasks.json`:

```json
{
  "tasks": [
    {
      "bundle": "../CLE-Lite/008-operations-research-analyst/harbor",
      "env": "../.cle-runtime-008-operations-research-analyst"
    },
    {
      "bundle": "../CLE-Lite/019-computer-research-scientist/harbor",
      "env": "../.cle-runtime-019-computer-research-scientist"
    }
  ]
}
```

Download and prepare every listed task, then replace `--task-list runtime/lite-example-tasks.json` in a stream command with `--task-list runtime/my-tasks.json`. Paths are resolved relative to the JSON file. Reorder the entries to change the evaluation order.

In a CLE stream, the agent retains its Codex conversation and the declared `/app/cle-memory/` directory between tasks. Each new task receives fresh task files and application services. Each Trial starts with its own initial state; `run.json` records the state transfers.

## Understand the scores

The [Quick start](#quick-start) explains how to select Direct or CLE mode. Both use the released tasks and their original grading checks.

- **Progression:** in CLE mode, the task ends when the agent exhausts its allowed submissions without passing, or runs out of execution time. Unreached levels count as failed.
- **Time budget:** each level uses the execution budget recorded in its task package. Trial 2 submissions share that budget. Grading time is excluded.
- **Feedback:** Trial 2 identifies unmet rubric categories, the section titles of the level's rubric (for example `Fiscal quarters and trailing year`); see [Feedback categories](#feedback-categories) for how this maps to the task checklist. Private rubrics, reference answers, and judge logs stay outside the solver workspace.
- **Errors:** runs interrupted by infrastructure or judge errors are marked invalid.

For a CLE Lite evaluation, **completion = cleared levels / (5 × number of selected tasks)**. For example, clearing seven levels across two tasks gives 7/10, or **70% completion**. L1–L5 use the same cumulative definition as the table below, with the selected tasks as the denominator. Values in `summary.json` are fractions from 0 to 1.

Direct mode also reports the fraction of rubric checks passed and the number of consecutive levels passed starting at `t1`.

| Metric | Meaning |
| :--- | :--- |
| Completion | Cleared levels divided by five times the number of selected tasks. |
| L1–L5 | For each level k, the fraction of selected tasks whose first k levels are all cleared. L5 is full-task completion. |
| Gain | Trial 2 completion minus Trial 1 completion. Multiply the difference by 100 to report percentage points. |

Record the runtime and data revisions, task order, solver model, judge model and backend, reasoning effort, execution budget, and state retention settings with each result.

### Feedback categories

Trial 2 returns the names of rubric categories containing failed checks after the first unsuccessful submission. The categories are the section titles of each level's rubric page in CLE-Lite-Rubrics (for example `Fiscal quarters and trailing year`); the runtime's `rubric-categories.json` maps every check to its section, and another mapping can be supplied with `--rubric-categories`. The agent receives the category names and can revise its submission within the remaining level budget.

### Relationship to the main experiment

The [CLE main experiment](https://haoyu-zhao.github.io/CLE/assets/continual-learning-exam.pdf) evaluates **120 tasks and 600 levels** in a fixed serial order, retaining the agent's harness state across tasks. It uses DeepSeek V4 Flash for model-based grading.

CLE mode follows the same pass-to-advance and submission rules: Trial 1 allows one submission per level; Trial 2 adds category feedback and one revision after a failed first submission. Use `stream --setting cle` to retain conversation and configured memory across the selected tasks. Each task starts with its own files and application services.

CLE Lite provides **20 tasks and 100 levels**. Report the selected tasks and use their level count as the denominator; the full CLE leaderboard uses all 600 levels.

## Grade saved Direct submissions

`run` already performs scoring. To grade a saved direct-mode composite again with the configured judge, use a fresh results directory:

```bash
python3 runtime/cle_lite.py grade \
    --bundle "$BUNDLE" --env "$RUNTIME_ENV" \
    --output ./results/direct-008-01/composite-output \
    --judge-backend "$JUDGE_BACKEND" --judge-model "$JUDGE_MODEL" \
    --results ./results/direct-008-regrade-01
```

For a Codex judge, export `CLE_CODEX_AUTH_FILE` as in setup. For OpenRouter, export `OPENROUTER_API_KEY`. Changing the judge for a **CLE** evaluation requires a new run or stream: its pass gates, permitted retries and later submissions depend on the judge's earlier decisions. Regrading old snapshots cannot reconstruct that interaction.

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
