# README figures

These figures accompany the project README and the public [CLE Lite dataset page](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite). PNG files are used for consistent rendering on GitHub and Hugging Face; matching SVG sources are included for editing and reuse. The artwork comes from the CLE Lite homepage preview supplied by the project maintainer.

| Figure | Scope |
| :--- | :--- |
| `cle-banner` | Shared CLE wordmark. |
| `cle-lite-glance` | The 20-task CLE Lite release: 100 levels, 16 occupations, 59,261 workspace files and 3,309 rubric criteria. |
| `cle-lite-levels` | The five assignments in task `016-data-scientist`, with newly supplied files and rubric criteria for each level. |
| `cle-lite-rubrics` | Rubric counts across the 20 Lite tasks, grouped by level. |
| `cle-leaderboard` | Main-experiment results for the full CLE benchmark: 120 tasks and 600 levels. These are not Lite subset scores. |

## Data and counting method

The figures were checked against the released data on 2026-10-22. Exact dataset revisions and aggregate counts are recorded in [statistics.json](statistics.json).

- Task, level and occupation counts come from the public task cards. Occupations count distinct `role` values.
- Workspace counts include file paths under each task's `workspace/` directory in [CLE Lite Workspaces](https://huggingface.co/datasets/Continual-Learning-Exam/CLE-Lite-Workspaces/tree/d8159f8d242140a478458732ea9f3816b7155a86). They exclude 112 empty `.gitkeep` placeholders. Source-code trees are included; identical content at different paths is counted separately. The archive contains 59,373 file entries before that exclusion and 59,261 material files afterward, totalling 29,084,108,048 bytes (29.1 decimal GB).
- The PJM figure uses the same counting rule separately for `workspace/t1/` through `workspace/t5/`: 274, 23, 27, 11 and 25 files.
- Rubric counts are checklist items in each released `eval/t1.md` through `eval/t5.md`, summed across the 20 tasks: 477, 556, 647, 749 and 880. Only aggregate counts are shown; rubric text and reference answers are not included in these assets.
- The leaderboard follows the [CLE main results](https://haoyu-zhao.github.io/CLE/leaderboard/) and rounds displayed percentages to one decimal place. The README tables retain two decimal places.

The Lite workspace inventory and the paper's full-benchmark input-file inventory are separate reported statistics. Do not combine their totals. The Lite figures describe release contents; they do not establish equivalence between the Lite runtime protocol and the main experiment.
