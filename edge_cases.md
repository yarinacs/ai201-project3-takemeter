# Edge Case Log — TakeMeter (signal/noise annotation)

Running log of posts that took more than ~15 seconds to label. Each entry records the post, the call, and the deciding rule, so the signal/noise boundary stays consistent across all 200+ examples. The three rules below are derived here and explained in full in [planning.md §3](planning.md).

| # | id | title (short) | score | label | why it was hard | deciding rule |
|---|----|----|------|------|-----------------|---------------|
| 1 | `nc6qi3` | Let's revive WSB culture! GME to THOUSANDS… (VW/TSLA TA) | 16488 | **noise** | claims a "GME thesis" + TA comparison, but the analysis is in a linked chart; body is hype | Label by the post **text itself**, not by what it links to. |
| 2 | `nwsnib` | Hard to swallow pill: Robinhood will skyrocket at IPO | 20 | **noise** | offers a real causal mechanism + a falsifiable prediction — closest call | A mechanism alone isn't enough; Signal needs **verifiable support** (data/math/sources), not a confident narrative. |
| 3 | `l7z0u9` | The GME shorts' endgame. INFORMATION PURPOSES ONLY | 1792 | **noise** | full DD costume (TL;DR, "educational"), high score | Restated/secondhand content + DD **framing** ≠ Signal; real message was "DIAMOND HAND" (a rally cry). |

## Tie-breaker
When still 50/50 after the rules above: **"Could a reader disagree with the *argument*, or only with the *mood*?"**
Argument to disagree with → `signal`. Only a mood → `noise`.

## Patterns surfaced during error analysis (planning §7c)

Reviewing the fine-tuned model's 7 test errors exposed boundary cases worth codifying:

| pattern | example | call | rule |
|---|---|---|---|
| **Data dump / list** | short-interest table (GME 41%, SKT 40%…); earnings-move compilation ($FB/$AMZN/…) | **noise** | Reposting numbers/tables with no original argument is noise, however "quantitative" it looks. |
| **Argument lives in an external video/link** | "watch this video and decide… for experienced traders" | **noise** | Reaffirms Case 1 — judge the post's own text; an external-media pointer is not in-text analysis. (This post was originally mislabeled `signal` — a label-noise example found via §7c.) |
| **Meta-rant *about* analysis** | "this sub is full of stupid DD now…" | **noise** | Mentioning "DD"/analysis vocabulary ≠ doing analysis. |
| **Tentative transcription** | "someone mentioned a special dividend… Investopedia says…" | borderline → lean **noise** | Restating external info + asking is closer to a question than an original supported claim. |

---

_Add new rows below as annotation continues._
