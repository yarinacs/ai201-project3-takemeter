# Edge Case Log — TakeMeter (S/N annotation)

Running log of posts that took more than ~15 seconds to label. Each entry records the post, the call, and the deciding rule, so the S/N boundary stays consistent across all 200+ examples. The three rules below are derived here and explained in full in [planning.md §3](planning.md).

| # | id | title (short) | score | label | why it was hard | deciding rule |
|---|----|----|------|------|-----------------|---------------|
| 1 | `nc6qi3` | Let's revive WSB culture! GME to THOUSANDS… (VW/TSLA TA) | 16488 | **N** | claims a "GME thesis" + TA comparison, but the analysis is in a linked chart; body is hype | Label by the post **text itself**, not by what it links to. |
| 2 | `nwsnib` | Hard to swallow pill: Robinhood will skyrocket at IPO | 20 | **N** | offers a real causal mechanism + a falsifiable prediction — closest call | A mechanism alone isn't enough; Signal needs **verifiable support** (data/math/sources), not a confident narrative. |
| 3 | `l7z0u9` | The GME shorts' endgame. INFORMATION PURPOSES ONLY | 1792 | **N** | full DD costume (TL;DR, "educational"), high score | Restated/secondhand content + DD **framing** ≠ Signal; real message was "DIAMOND HAND" (a rally cry). |

## Tie-breaker
When still 50/50 after the rules above: **"Could a reader disagree with the *argument*, or only with the *mood*?"**
Argument to disagree with → `S`. Only a mood → `N`.

---

_Add new rows below as annotation continues._
