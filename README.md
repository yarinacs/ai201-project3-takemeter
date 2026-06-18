# TakeMeter 🚀📊

A binary text classifier that reads a **r/wallstreetbets** post and decides whether it carries a real, reasoned *take* — **Signal** — or whether it's ambient **Noise** (hype, questions, PSAs, social chatter).

WSB during the GameStop squeeze had a notoriously low signal-to-noise ratio. TakeMeter is a "take meter" that surfaces the small fraction of posts actually worth reading and filters out the firehose.

> This repository holds everything **outside** the Colab notebook: planning, the labeled dataset, and the evaluation outputs downloaded from Colab. The model training/evaluation itself lives in the Colab notebook.

## Labels

| Label | Name | Definition |
|-------|------|------------|
| `S` | **Signal** | The post advances an original, reasoned position about a security/market/strategy, supported by evidence or analysis (data, options/short math, a comparison, a mechanism, a worked thesis) — something a reader could argue *with*. |
| `N` | **Noise** | The post advances no supported analytical claim: a question, a rally cry / hype, a celebration, a meta/news post, or a PSA. |

Full definitions, example posts, and the S/N edge-case rules are in **[planning.md](planning.md)**.

## Dataset

- **Source:** Kaggle [Reddit WallStreetBets Posts](https://www.kaggle.com/datasets/gpreda/reddit-wallstreetsbets-posts) (`reddit_wsb.csv`, ~349k rows, Jan–Mar 2021).
- **Labeled working set:** [`wsb_to_label.csv`](wsb_to_label.csv) — columns `id, title, text, score, label`.
- **Class balance:** Signal is the minority class (~30%), reflecting WSB's real base rate. See planning.md §4–§5 for how this drives the metric choices.

> `reddit_wsb.csv` (~39 MB) is **gitignored**. Download it from the Kaggle link above and place it in the repo root to reproduce sampling.

## Evaluation

Results produced in Colab and saved here:

- [`outputs/evaluation_results.json`](outputs/evaluation_results.json) — per-class precision/recall/F1, macro-F1, and the always-N baseline.
- [`outputs/confusion_matrix.png`](outputs/confusion_matrix.png) — error directions (missed Signal vs. false alarms).

**Why not accuracy?** With a ~70/30 split, always predicting Noise scores ~70% accuracy while catching zero Signal. The headline metric is **macro-F1**, with **Signal precision/recall** as the deployment-critical numbers. Success targets are defined in planning.md §6.

> _Results pending — populated once the Colab outputs are downloaded into `outputs/`._

## AI Usage Disclosure

AI tools (Claude) were used in three scoped ways, per planning.md §7:

1. **Label stress-testing** — generating boundary-case posts to pressure-test and tighten the label definitions before annotation.
2. **Annotation assistance** — pre-labeling a batch for human review. Pre-labeled rows are flagged with a `prelabeled` column; the human-override rate is reported as a quality check. Every label was reviewed by a human.
3. **Failure analysis** — clustering wrong predictions to propose error patterns, each verified by hand before being reported.

AI tools were also used to help draft this repository's planning and documentation.

## Repository Layout

```
ai201-project3-takemeter/
├── planning.md               # design: community, labels, edge cases, metrics, AI plan
├── README.md                 # this file
├── wsb_to_label.csv          # labeled working dataset
├── reddit_wsb.csv            # raw source (gitignored — get from Kaggle)
├── edge_cases.md             # running log of S/N boundary calls (created during annotation)
└── outputs/                  # downloaded from Colab
    ├── evaluation_results.json
    └── confusion_matrix.png
```
