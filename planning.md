# TakeMeter — Planning

**Project:** A text classifier that reads a r/wallstreetbets post and decides whether it carries a real, reasoned *take* — a "Signal" — or whether it is ambient "Noise" (hype, questions, PSAs, social chatter).

**One-line goal:** Build a "take meter" that a community tool could use to surface the small fraction of WSB posts that actually contain analysis, while filtering out the firehose of noise.

> Status note: the community, label scheme, and dataset below are not hypothetical — they reflect the data already in this repo (`reddit_wsb.csv`, `wsb_to_label.csv`). 213 posts are labeled so far (148 N / 63 S). The plan is written to match that reality and is the design I'm committing to; if I change the scheme later I'll update this file before annotating more.

---

## 1. Community

**Chosen community: r/wallstreetbets (WSB), GameStop-era posts (Jan–Mar 2021).**

I chose WSB because it is a single community that produces wildly heterogeneous discourse around the *same* subject matter. During the GME short squeeze, the subreddit simultaneously hosted:

- genuine, numerate due-diligence ("DD") posts with options math, short-interest analysis, and explicit theses;
- pure hype and rally cries ("GME TO THE MOON 🚀💎🙌");
- beginner questions about mechanics ("what time do shorts have to cover?");
- public-service announcements (broker outages, transfer warnings);
- meta/news posts and social/celebration posts.

That mix is exactly what makes it a good classification target. The *topic* is held nearly constant (GME, AMC, options, brokers), so the classifier cannot cheat by keying on subject keywords — it has to learn the harder distinction between **reasoned analysis** and **everything else**. The discourse is varied along the axis I actually care about (analytical content vs. not), which is the axis that makes the task interesting rather than trivially keyword-separable.

It is also a community where this classifier would be *useful*: WSB's signal-to-noise ratio is notoriously low, so an automatic "TakeMeter" that flags the posts worth reading has obvious real-world value for a reader, a moderator filter, or a research pipeline.

---

## 2. Labels

This is a **binary** scheme (2 labels). I considered a 3-way split (Signal / Hype / Admin) but collapsed it: in practice the boundary that matters for a "is this worth reading" tool is reasoned-analysis vs. not, and a binary scheme keeps annotation consistent and the minority class large enough to learn.

### Label `S` — **Signal**
> A post is **Signal** if it presents an original, reasoned position about a security, market, or strategy — i.e., it advances a specific claim and supports it with evidence or analysis (data, options/short math, a comparison, a mechanism, or a worked thesis), such that a reader could agree or disagree with the *argument* rather than just the vibe.

Example posts:
1. **"GME DD: Analysis of options expiry on 2/26 and potential for gamma and short squeezes"** (`id=lspbb5`) — opens with a TL;DR, computes ~$306m / 7m shares in-the-money under a stated price assumption, and reasons about gamma exposure. A textbook reasoned take.
2. **"Prisoner's Dilemma — 💎🖐️ $GME 🚀"** (`id=l9cnx6`) — despite the emoji title, the body lays out a game-theoretic argument for why holders' incentives interact, i.e., a structured thesis you can argue with.

### Label `N` — **Noise**
> A post is **Noise** if it does *not* advance a supported analytical claim: it is a question, a rally cry / hype, a celebration or social shout-out, a meta/news/media post, or a PSA. The post may be useful or popular, but it contains no original analysis a reader could evaluate as an argument.

Example posts:
1. **"What time of day Fri the 29th do the GME shorts have to be reimbursed by buying stocks?"** (`id=l71mc4`) — a pure mechanics question; asks for information, advances no thesis.
2. **"I just want to give a big s/o to each of you!"** (`id=l71u5q`) — social/celebration; no analytical content.

---

## 3. Hard Edge Cases

The genuinely ambiguous posts sit on the **S/N boundary** in three recurring ways:

1. **Thesis-flavored hype.** A rally post that *links to or gestures at* analysis but whose body is mostly cheerleading. Real example: **"Let's revive the buried WSB culture! GME to THOUSANDS…"** (`id=nc6qi3`, score 16488) embeds a VW/TSLA comparison chart but reads as a hype/morale post. → **Rule: label by the body of the post itself, not by what it links to.** If the reasoning lives in an external image/link and the post text is hype, it is **N**.

2. **Confident prediction with no support.** A post that makes a strong directional call in an analytical *register* but offers no evidence or mechanism. Real example: **"Hard to swallow pill: Robinhood is going to skyrocket at IPO."** (`id=nwsnib`) — reads serious, but is assertion, not argument. → **Rule: a claim without supporting reasoning is N.** Signal requires the *support*, not just the confidence.

3. **Authoritative-sounding meta.** Posts like **"The GME shorts' endgame. INFORMATION PURPOSES ONLY."** (`id=l7z0u9`) that adopt DD framing but are summary/narration rather than original analysis. → **Rule: restating known events ≠ analysis; if there's no original claim+support, it's N.**

**Handling during annotation:** when a post is genuinely 50/50, I apply the tie-breaker **"Could a reader *disagree with the argument*, or only with the *mood*?"** If there's an argument to disagree with → S; if only a mood → N. I will keep a running `edge_cases.md` log of every post I had to think about for more than ~15 seconds, with the post id, the call I made, and the deciding rule — so the boundary stays consistent across the full 200+ and so the rules above can be refined from real cases rather than guesses.

---

## 4. Data Collection Plan

**Source:** the public Kaggle "Reddit WallStreetBets Posts" dataset (`reddit_wsb.csv`, ~349k rows, Jan–Mar 2021). A working subset has been sampled into `wsb_to_label.csv` (id, title, text, score, label).

**Target:** ≥200 labeled posts, and I will push to **~250–300** so that after holding out a test set I still have enough of the minority class. Current state: **213 labeled (148 N / 63 S)** ≈ a 70/30 split — i.e., **Signal is the minority class.**

**If a label is underrepresented after 200 examples** (Signal almost certainly will be, since most of WSB is noise), I will *not* fabricate or duplicate examples. Instead:
- **Targeted sampling**, not random: draw additional candidates from `reddit_wsb.csv` that are *more likely* to be Signal — e.g., posts with "DD" / "thesis" / "analysis" in the title, longer bodies (analysis posts are long), and higher comment counts — then label them honestly (some will still turn out to be N).
- Keep collecting Signal candidates until I reach **at least 80–100 confirmed S** so the minority class is learnable and the test set has enough positives to score.
- **Report the real base rate** in the writeup. The natural class imbalance is itself a finding about WSB, and it directly drives the metric choices below.
- For training, address imbalance at the *model* level (class weights / threshold tuning) rather than by distorting the labeled distribution.

---

## 5. Evaluation Metrics

**Why accuracy alone is misleading here:** with a ~70/30 split, a model that predicts **N for everything scores ~70% accuracy** while finding *zero* Signal — i.e., it is useless for the actual goal. Accuracy rewards the majority class and hides total failure on the class I care about most.

Metrics I will report, and why:

- **Per-class precision, recall, and F1 — reported for the `S` (Signal) class specifically.** This is the heart of the evaluation, because the tool's job is to surface Signal:
  - **Recall (S)** = of all real Signal posts, how many did we catch? Low recall = the meter misses the good takes (the main failure I want to avoid).
  - **Precision (S)** = of posts we flagged as Signal, how many really were? Low precision = the meter cries wolf and the reader stops trusting it.
- **Macro-F1** (average of S-F1 and N-F1) as the single headline number, because it weights the minority class equally instead of letting the majority dominate.
- **Confusion matrix** (`outputs/confusion_matrix.png`) to see *which direction* errors go — N→S (false alarms) vs S→N (missed signal) — since those have different costs.
- **Majority-class baseline (always-predict-N)** reported alongside, so every number is judged against the ~70% "do nothing" floor rather than against 0%.

The full numeric results will be saved to `outputs/evaluation_results.json`.

---

## 6. Definition of Success

I want criteria specific enough that, at the end, I can mechanically check pass/fail.

**"Good enough" (the bar I'm setting):**
- **Macro-F1 ≥ 0.75**, and
- **Signal recall ≥ 0.70** (catch at least ~7 of every 10 real takes), and
- **Signal precision ≥ 0.65** (a flagged post is a real take more often than not by a clear margin), and
- beats the always-N baseline on macro-F1 by a wide margin (baseline macro-F1 ≈ 0.41).

**"Genuinely useful for deployment" in a real community tool:**
- **Signal precision ≥ 0.80** — when the TakeMeter says "this is a real take," a reader/mod can trust it ~4 of 5 times. For a *filtering* tool, precision is the trust-critical metric: a noisy filter is worse than no filter.
- Signal recall ≥ 0.70 maintained, so it isn't achieving precision by flagging almost nothing.
- Errors concentrated in the *defensible* edge cases from §3, not on clear-cut posts.

**Self-review — are these objective?** Yes: every criterion is a number computed from the held-out test set (precision/recall/F1 per class + the baseline), all emitted to `evaluation_results.json`. At the end I can read those numbers and state unambiguously whether each threshold was hit. The one soft criterion ("errors concentrated in edge cases") is made checkable by the §3 edge-case log: I can verify the misclassified posts against that list.

---

## 7. AI Tool Plan

This is an annotation-and-evaluation project, not an implementation project — there's little code to generate. AI tools help in three specific places:

### 7a. Label stress-testing (before annotating 200)
Before committing to 200+ annotations, I will give an LLM (Claude) my §2 label definitions and §3 edge-case description and ask it to **generate 8–10 posts that deliberately sit on the S/N boundary** (e.g., confident-but-unsupported calls, hype that name-drops a thesis, restated-news-as-analysis). If I can't classify its outputs cleanly with my current rules, that's evidence my definitions are too loose — I will **tighten the definitions and the §3 tie-breaker rules now, before annotating**, and note what changed. (This step runs first precisely so it can fix the labels cheaply.)

### 7b. Annotation assistance
I **will** use an LLM to **pre-label a batch** before reviewing each one myself, to speed up annotation — but every pre-label is reviewed and corrected by me; the LLM never has the final say. Tracking for disclosure:
- Tool: Claude.
- I will add a `prelabeled` column (true/false) to the labeling CSV so it's recorded which rows were AI-pre-labeled vs. labeled cold, and I can **report the human-override rate** (how often I disagreed with the AI) as a check on annotation quality.
- This is disclosed in the AI Usage section of the README.

### 7c. Failure analysis
After evaluation, I will export the list of **wrong predictions** (post text, true label, predicted label) and ask an LLM to **cluster them and propose patterns** ("misses are mostly long DD with heavy emoji," "false alarms are mostly PSAs in analytical tone," etc.). Then I will **verify each proposed pattern by hand** against the actual misclassified posts before putting it in the writeup — the AI proposes hypotheses, I confirm or reject them against the data. The verified patterns feed the error discussion and any §3 rule refinements.

---

## 8. Repository Layout

```
ai201-project3-takemeter/
├── planning.md               # this file
├── README.md                 # project overview, results, AI usage disclosure
├── reddit_wsb.csv            # raw source data (Kaggle) — gitignored if too large
├── wsb_to_label.csv          # working labeled dataset (id,title,text,score,label[,prelabeled])
├── edge_cases.md             # running log of boundary calls + deciding rules
└── outputs/                  # downloaded from Colab
    ├── evaluation_results.json
    └── confusion_matrix.png
```

The model/notebook itself lives in Colab; this repo holds everything *outside* the notebook.
