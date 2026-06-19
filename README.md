# TakeMeter 🚀📊

A binary text classifier that reads a **r/wallstreetbets** post and decides whether it carries a real, reasoned *take* — **Signal** — or whether it's ambient **Noise** (hype, questions, PSAs, social chatter).

WSB during the GameStop squeeze had a notoriously low signal-to-noise ratio. TakeMeter is a "take meter" that surfaces the small fraction of posts actually worth reading and filters out the firehose.

> This repository holds everything **outside** the Colab notebook: planning, the labeled dataset, and the evaluation outputs downloaded from Colab. The model training/evaluation itself lives in the Colab notebook.

## Labels

| Label | Name | Definition |
|-------|------|------------|
| `signal` | **Signal** | The post advances an original, reasoned position about a security/market/strategy, supported by evidence or analysis (data, options/short math, a comparison, a mechanism, a worked thesis) — something a reader could argue *with*. |
| `noise` | **Noise** | The post advances no supported analytical claim: a question, a rally cry / hype, a celebration, a meta/news post, or a PSA. |

Full definitions, example posts, and the signal/noise edge-case rules are in **[planning.md](planning.md)**.

## Dataset

- **Source:** Kaggle [Reddit WallStreetBets Posts](https://www.kaggle.com/datasets/gpreda/reddit-wallstreetsbets-posts) (`reddit_wsb.csv`, ~349k rows, Jan–Mar 2021).
- **Labeled working set:** [`wsb_to_label.csv`](wsb_to_label.csv) — columns `id, title, text, score, label`.
- **Class balance:** Signal is the minority class (~30%), reflecting WSB's real base rate. See planning.md §4–§5 for how this drives the metric choices.

> `reddit_wsb.csv` (~39 MB) is **gitignored**. Download it from the Kaggle link above and place it in the repo root to reproduce sampling.

## Evaluation

Results produced in Colab and saved here:

- [`outputs/evaluation_results.json`](outputs/evaluation_results.json) — per-class precision/recall/F1, macro-F1, and the always-N baseline.
- [`outputs/confusion_matrix.png`](outputs/confusion_matrix.png) — error directions (missed Signal vs. false alarms).

**Why not accuracy?** With a ~64/36 split, always predicting Noise scores ~63% accuracy while catching zero Signal (macro-F1 ≈ 0.41). The headline metric is **macro-F1**, with **Signal precision/recall** as the deployment-critical numbers. Success targets are defined in planning.md §6.

### Results (test set, n=35 — 13 signal / 22 noise)

| Model | Accuracy | Macro-F1 | Signal P | Signal R | Signal F1 |
|-------|:--------:|:--------:|:--------:|:--------:|:---------:|
| Majority baseline (always-noise) | 0.63 | 0.41 | — | 0.00 | 0.00 |
| Groq `llama-3.3-70b` (zero-shot prompt) | 0.77 | 0.76 | 0.67 | 0.77 | 0.71 |
| **DistilBERT (fine-tuned)** | **0.80** | **0.79** | **0.71** | **0.77** | **0.74** |

Against the planning.md §6 "good enough" bar, the **fine-tuned model passes all three criteria**: macro-F1 0.79 ≥ 0.75, Signal recall 0.77 ≥ 0.70, Signal precision 0.71 ≥ 0.65 — and both models crush the always-noise floor (0.41). Neither reaches the higher *deployment-grade* Signal precision ≥ 0.80 yet (best is 0.71).

**Two honest findings:**

1. **Data volume was the dominant lever, not hyperparameters.** Growing the labeled set from 63 → 82 Signal examples lifted the fine-tuned model's test macro-F1 from **0.64 → 0.79**. A separate config bug (warmup steps exceeding total training steps, plus selecting the best checkpoint on accuracy instead of macro-F1) had earlier collapsed the model into predicting Noise for everything — fixing those was the precondition for the model learning at all.
2. **A zero-shot 70B LLM nearly matches a fine-tuned model here.** On 35 test examples the fine-tuned model (0.79) and the prompt-only baseline (0.76) differ by ~1 example — effectively a tie. At this data scale, prompting a large instruction model is a strong, training-free baseline.

> ⚠️ **Small-sample caveat:** the test set is only 35 examples (13 signal). A single flipped prediction moves macro-F1 by ~3–5 points, so 0.79 vs 0.76 is *not* a meaningful gap, and "passes §6" should be read as "clears the bar on a noisy estimate." The clearest next step is more labeled data (planning.md §4).

### Confusion Matrix — Fine-Tuned Model (test set, n=35)

Rows = true label, columns = predicted (same data as [`outputs/confusion_matrix.png`](outputs/confusion_matrix.png)):

|                    | Pred: `signal` | Pred: `noise` | Total |
|--------------------|:----:|:----:|:----:|
| **True: `signal`** | **10** | 3 | 13 |
| **True: `noise`**  | 4 | **18** | 22 |
| **Total**          | 14 | 21 | 35 |

Because the task is binary, **every error lies on the single `signal ↔ noise` boundary** — there is one confused pair and it accounts for 100% of the errors. The split is **4 false alarms** (noise→signal) vs **3 misses** (signal→noise): false alarms slightly dominate, while recall is balanced (0.77 in each direction). So the model's weakness is leaning *toward* calling things Signal.

### Surfacing error patterns with AI — and what I corrected

I pasted all 7 misclassified posts into Claude and asked it to find common themes (post length, sarcasm, a recurring confused direction, low-information posts, etc.). It proposed two clusters that held up on re-reading:

- **(A) "Costume of analysis" false alarms** — the model calls a post Signal when it carries the *surface markers* of analysis (tickers, numbers, the word "DD," earnings vocabulary) even with no original argument.
- **(B) Buried / tentative / external missed Signal** — it calls real analysis Noise when the reasoning is phrased conversationally, sits past the truncation window, or lives in a linked video.

**What I corrected after verifying by hand:**
- Claude initially counted all 7 as *model* errors. Re-reading showed one — the external-video post — is actually a **labeling** error (Noise under my §3 rule), so I moved it from "model failure" into the label-noise discussion below.
- I **discarded** a weaker theme it floated ("errors correlate with high post score"); the examples didn't support it (the misclassified scores ranged from 1 to thousands).

### Three analyzed failures

**Failure 1 — earnings-data compilation → false alarm (the model's *most confident* error).**
> *"Historical Post Earnings Moves MEGA Compilation (Week 2) — $FB, $AMZN, $AAPL, $AMD, $MSFT, $PINS, and More… I fucking love earnings season."*
> True: `noise` · Predicted: `signal` · **confidence 0.92**
- **Which boundary:** noise→signal. **Why it's hard:** the post is wall-to-wall tickers and earnings numbers, so every *surface* cue screams "analysis" — but structurally it is a **data compilation with no original thesis**, nothing a reader can argue *with*.
- **Labeling or data problem?** The label is consistent with §2 (a data dump is Noise), so this is a **data/boundary problem**, not annotation drift: the training set has too few "quantitative-but-Noise" posts for the model to learn that *numbers ≠ argument*.
- **What would fix it:** more training examples of data dumps, screenshotted tables, and compilations explicitly labeled Noise.

**Failure 2 — meta-rant *about* DD → false alarm.**
> *"What the fuck happened? This used to be a sub full of different opinions… people post stupid DD that is…"*
> True: `noise` · Predicted: `signal` · confidence 0.70
- **Which boundary:** noise→signal. **Why it's hard:** the post contains "DD" and "opinions," and the model keys on those tokens — but it is a **complaint *about* analysis culture**, not analysis.
- **Labeling or data problem?** Consistent label; the issue is **lexical-cue overfitting** — the model treats the token "DD" as nearly decisive.
- **What would fix it:** training examples of meta / complaint posts that *use* analysis vocabulary without doing analysis.

**Failure 3 — reposted DD whose math is buried → missed Signal.**
> *"I post a DD yesterday and gained lots of comments… So I paste that calculation again here…"*
> True: `signal` · Predicted: `noise` · confidence 0.66
- **Which boundary:** signal→noise. **Why it's hard:** the *visible* text is chatty preamble; the actual calculation sits lower in the body and is almost certainly cut off by the **256-token truncation**. Structure signals Noise; the Signal content never reaches the model.
- **Labeling or data problem?** Consistent label; this is a **representation problem** (truncation + preamble-before-substance ordering), not a label or boundary problem.
- **What would fix it:** feed `title + body`, raise `max_length`, or keep the TL;DR up front; add buried-DD examples.

**Label-quality finding (honest).** Verification of a 4th error — a post that says *"watch this video and decide"* (argument in an external video) — showed it should be **Noise** under my §3 rule (*judge the post's own text, not what it links to*) but was labeled `signal` in a not-fully-careful annotation pass. So the dataset carries some **label noise**, and the model's true accuracy is **at least** the reported 0.79. I deliberately did **not** edit the held-out test labels to match the prediction (that would tamper with the test set); I report it instead. The correct fix is a §3-driven consistency relabel over the *full* dataset, applied blind to predictions — listed as future work.

### Sample Classifications (fine-tuned model)

Each row is a real test post run through the fine-tuned model, with its predicted label and softmax confidence:

| # | Post (excerpt) | Predicted | Confidence | Correct? |
|---|----------------|:---------:|:----------:|:--------:|
| 1 | "Historical Post Earnings Moves MEGA Compilation — $FB, $AMZN, $AAPL, $AMD…" | `signal` | 0.92 | ❌ (true: noise) |
| 2 | "What the fuck happened? …people post stupid DD that is…" | `signal` | 0.70 | ❌ (true: noise) |
| 3 | "I post a DD yesterday… So I paste that calculation again here…" | `noise` | 0.66 | ❌ (true: signal) |
| 4 | "Take this with a grain of salt… **TL/DR: Funds are going to fear**…" | `signal` | 0.87 | ✅ (true: signal) |
| 5 | "I've seen a few people saying to buy RYCEY… **Is this our next target?**" | `noise` | 0.92 | ✅ (true: noise) |

**Why these correct predictions are reasonable.** Row 4 is correctly `signal`: despite the "not financial advice" hedging, it opens with a **TL;DR and advances an original thesis** (a mechanism for why funds will act a certain way) — exactly the §2 markers of a supported claim, so surface and substance agree. Row 5 is correctly `noise` with high confidence: it is a **bare question** ("Is this our next target?") with no claim a reader could argue against — the textbook Noise shape.

## Reflection: what I intended vs. what the model captured

**Intended boundary:** Signal vs. Noise was *defined* semantically — "does the post make an original claim and *support* it with reasoning you could argue against?" (planning.md §2). The intent was for the model to judge **substance**.

**Captured boundary:** the decision boundary the model actually learned is closer to **"does this *look* like analysis?"** — it leans heavily on surface markers (tickers, numbers, "DD," earnings vocabulary, length). 

- **What it overfit to:** lexical and structural cues of finance-talk. Posts wearing the "costume of analysis" (data dumps, ticker lists, the token "DD") get called Signal even with no argument — that direction produced 4 of the 7 errors.
- **What it missed:** reasoning that doesn't *look* the part — tentatively-phrased arguments ("someone mentioned… Investopedia says…"), or substance buried past the truncation window / behind a chatty preamble / in an external link.

This is precisely the §3 edge case, learned the hard way: the gap between my definition (substance) and the model's boundary (appearance) *is* the project's core finding. With only ~160 training examples, the model latched onto the cheap, high-frequency surface correlation instead of the expensive semantic one. Closing that gap needs training examples that **break** the correlation — quantitative-but-Noise and plain-spoken-but-Signal posts — far more than it needs more hyperparameter tuning.

## Spec Reflection

**One way the spec helped:** the spec's insistence on **per-class metrics and a confusion matrix rather than accuracy alone** is what made the first failure legible. The initial fine-tuned model scored ~0.69 accuracy — which *looks* passable — but the per-class breakdown exposed Signal recall = 0.00 and macro-F1 = 0.41: the model was predicting Noise for everything. Without the spec forcing per-class reporting (and the planning.md §5 metric design it required), that collapse would have hidden behind a respectable-looking accuracy number, and I'd have "passed" with a useless classifier.

**One way my implementation diverged:** the starter taxonomy assumed a **3–4 label scheme** (the illustrative r/nba example), and the notebook front-loaded a prompt-based classifier. I diverged by (a) collapsing to a **binary signal/noise** scheme — the boundary that actually matters for a "worth reading" filter is reasoned-analysis-vs-not, and a binary split kept the minority class large enough to learn — and (b) treating the **fine-tuned DistilBERT as the primary deliverable** with the Groq prompt as a comparison baseline, rather than the reverse. I also diverged from my *own* plan (planning.md §7b): I intended to LLM-pre-label a batch, but ultimately labeled everything cold and dropped the `prelabeled` workflow — disclosed below.

## AI Usage

AI tools (Claude) were used throughout. Three specific, verifiable instances:

**Instance 1 — debugging the model collapse.** *Directed:* I gave Claude the training cell and the all-Noise result (macro-F1 0.41) and asked why the model wasn't learning. *Produced:* it diagnosed three compounding causes — `warmup_steps=50` exceeding the ~30 total training steps (so the LR never ramped up), `metric_for_best_model="accuracy"` actively selecting the all-Noise checkpoint, and no class weights for the 64/36 imbalance — and supplied corrected `TrainingArguments` plus a class-weighted `Trainer`. *What I changed / verified:* I applied the fixes and re-ran; macro-F1 went 0.41 → 0.64, then 0.79 after I also added data. I kept the 8-epoch setting but relied on early-stopping (best checkpoint = epoch 6/3 depending on run) rather than its suggested epoch count blindly.

**Instance 2 — targeted candidate sampling for the minority class.** *Directed:* with Signal underrepresented, I asked Claude to surface likely-Signal posts from the ~50k raw rows. *Produced:* a 200-row ranked CSV scored by DD/analysis keywords, body length, and comment count. *What I overrode:* I labeled all 200 **by hand** and rejected the heuristic's optimism on several fronts — e.g., the top-ranked recurring "Wall Street Week Ahead" digests scored high but I labeled them **Noise** (restated market recaps, not original analysis). The tool found candidates; the labels were mine.

**Annotation disclosure.** I did **not** use LLM pre-labeling: although planning.md §7b proposed an LLM-pre-label-then-review workflow with a `prelabeled` flag, every label in `wsb_to_label.csv` was assigned cold by a human. Claude's only role in annotation was generating the *unlabeled* candidate pool (Instance 2). Claude also helped surface error patterns from the 7 misclassified posts (see "Surfacing error patterns with AI"), where I corrected one mislabeled case and discarded one unsupported theme.

## Demo

A 3–5 minute walkthrough video accompanies this submission: it shows 5 posts classified by the fine-tuned model with label + confidence, narrates one correct and one incorrect prediction, and walks through this evaluation report. _(Link: TODO — add once recorded.)_

## Repository Layout

```
ai201-project3-takemeter/
├── planning.md               # design: community, labels, edge cases, metrics, AI plan
├── README.md                 # this file
├── wsb_to_label.csv          # labeled working dataset
├── reddit_wsb.csv            # raw source (gitignored — get from Kaggle)
├── edge_cases.md             # running log of signal/noise boundary calls (created during annotation)
└── outputs/                  # downloaded from Colab
    ├── evaluation_results.json
    └── confusion_matrix.png
```
