# TakeMeter — r/nba Discourse Classifier

A fine-tuned text classifier that categorizes r/nba posts and comments into three discourse types: `analysis`, `hot_take`, and `reaction`. Built for AI 201 Project 3.

---

## Community Choice

I chose **r/nba** (the NBA subreddit) because it is one of the most active sports communities on Reddit, with discourse that varies enormously in quality. Some posts make careful statistical arguments, others are pure emotional reactions to games, and many are bold opinions stated as facts with no evidence. This variation makes it an ideal classification task — the distinctions are real, they matter to participants, and they appear consistently enough to build a 200-example dataset from.

---

## Label Taxonomy

| Label | Definition |
|-------|-----------|
| `analysis` | A post that makes a structured argument supported by specific, verifiable evidence — statistics, historical comparisons, game film observations, or tactical breakdowns. |
| `hot_take` | A bold, confident opinion stated without meaningful supporting evidence. The post asserts a claim rather than arguing for it. |
| `reaction` | An immediate emotional response to a specific recent game or event. The post expresses a feeling in the moment with little to no argument. |

**Examples per label:**

`analysis`:
- "Over the last 3 seasons, Jokic leads all centers in assists per game (9.1), PER (31.4), and True Shooting (.638). There's no statistical case for anyone else as the best big man."
- "Hart has 18 fouls in 108 game minutes. Castle has 21 in 185 game minutes. They have been VERY lenient on Castle."

`hot_take`:
- "Steph Curry would not survive in the 90s. He is a product of the modern game and soft defenders."
- "The Warriors dynasty is the most overrated run in NBA history."

`reaction`:
- "BRO WHAT WAS THAT SHOT FROM TATUM I'M LOSING MY MIND"
- "Can't believe we just watched that. This team is cooked."

---

## Data Collection

**Source:** r/nba — post titles, post bodies, and comment threads collected manually by browsing the subreddit feed and clicking into comment sections.

**Labeling process:** Each example was read individually and assigned one of three labels using the definitions above. An AI tool (Claude) was used to pre-label batches of 30–40 posts at a time; every pre-assigned label was reviewed and corrected manually before inclusion.

**Label distribution:**

| Label | Count | Percentage |
|-------|-------|-----------|
| analysis | 43 | 21.3% |
| hot_take | 77 | 38.1% |
| reaction | 82 | 40.6% |
| **Total** | **202** | |

**Three difficult-to-label examples:**

1. *"LaMelo Ball is the perfect complement to Edwards — a superb passer and dynamic shooter who will open the floor."* — This reads like a hot_take (bold claim) but contains specific reasoning about fit. Labeled `analysis` because it argues a position rather than just asserting it.

2. *"Two of the three pick swaps in the LaMelo deal are basically worthless."* — Borderline between hot_take and analysis. Labeled `analysis` because the post goes on to explain specifically which picks and why, with verifiable context.

3. *"Does he know we can see him?"* — Very short, no obvious game context. Labeled `reaction` because it is a sarcastic emotional response to watching a player foul repeatedly.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` from HuggingFace

**Training setup:**
- Framework: HuggingFace `transformers` + `Trainer`
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16 (train), 32 (eval)
- Weight decay: 0.01
- Warmup steps: 50
- Train/Val/Test split: 70% / 15% / 15% (stratified)
- GPU: Google Colab T4

**Key hyperparameter decision:** I kept the default 3 epochs rather than increasing to 5, because with only ~140 training examples, more epochs risk overfitting. The model already showed signs of struggling to learn the `analysis` vs `hot_take` boundary, and more epochs would likely amplify that confusion rather than resolve it.

---

## Baseline Description

**Model:** `llama-3.3-70b-versatile` via Groq API (zero-shot)

**Prompt used:**
```
You are classifying posts and comments from the r/nba subreddit on Reddit.
Assign each post to exactly one of the following categories.

analysis: The post makes a structured argument supported by specific, verifiable evidence such as statistics, historical comparisons, or tactical observations.
Example: "Over the last 3 seasons Jokic leads all centers in assists per game PER and True Shooting percentage."

hot_take: A bold confident opinion stated without meaningful supporting evidence. The post asserts a claim rather than arguing for it.
Example: "Steph Curry would not survive in the 90s. He is a product of the modern game and soft defenders."

reaction: An immediate emotional response to a specific recent game or event. The post expresses a feeling in the moment with little to no argument.
Example: "BRO WHAT WAS THAT SHOT I AM LOSING MY MIND"

Respond with ONLY the label name. Do not explain your reasoning.
```

All 31 test examples received parseable responses.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|-------|---------|
| Zero-shot baseline (Groq llama-3.3-70b) | **0.677** |
| Fine-tuned DistilBERT | **0.516** |

The fine-tuned model performed **worse** than the zero-shot baseline by 16.1 percentage points.

### Per-Class Metrics — Baseline (Groq)

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|-----|---------|
| analysis | 0.57 | 0.67 | 0.62 | 6 |
| hot_take | 0.62 | 0.67 | 0.64 | 12 |
| reaction | 0.82 | 0.69 | 0.75 | 13 |
| **accuracy** | | | **0.68** | 31 |

### Per-Class Metrics — Fine-Tuned DistilBERT

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|-----|---------|
| analysis | low | low | low | 6 |
| hot_take | moderate | high | moderate | 12 |
| reaction | moderate | low | low | 13 |
| **accuracy** | | | **0.52** | 31 |

### Confusion Matrix (Fine-Tuned Model)

|  | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|--|---|---|---|
| **True: analysis** | low | high | low |
| **True: hot_take** | low | moderate | low |
| **True: reaction** | low | high | low |

See `confusion_matrix.png` for the full visualization.

### Wrong Predictions — Analysis of 3 Failures

**Failure #1:**
- Text: *"Clippers Draft Pick Narcisse Ngoy Decides to Return to Auburn Instead of Playing for LAC"*
- True: `reaction` | Predicted: `hot_take`
- Why it failed: This is a short news headline with no emotional language. The model had no clear signal for `reaction` and defaulted to `hot_take`. The model learned to associate short assertive statements with `hot_take`, which is wrong here — this is just a news fact.

**Failure #2:**
- Text: *"Among star players his on/off profile is an anomaly. Cannot name another max guy whose team has consistently performed better when he is off the court"*
- True: `analysis` | Predicted: `hot_take`
- Why it failed: This is the hardest boundary in the dataset. The post makes a specific analytical observation but uses assertive language ("cannot name another"). The model picked up on the assertive tone and called it `hot_take`, missing the underlying statistical reasoning. This is exactly the edge case I identified in my planning document.

**Failure #3:**
- Text: *"Brunson made 8 baskets from within 10 feet tonight which ties the most by any player against the Spurs in the playoffs. He was not blocked by Wemby a single time"*
- True: `analysis` | Predicted: `hot_take`
- Why it failed: This post cites specific game statistics but is written in a game-thread context, which the model may have associated with emotional `reaction` or assertive `hot_take` posts. With only 43 analysis examples in the dataset, the model did not learn this pattern well enough.

### Sample Classifications

| Post | Predicted Label | Confidence | Notes |
|------|----------------|------------|-------|
| "Jalen Brunson Tonight: 45 PTs, Rest of Team: 49 PTs" | reaction | 0.71 | Correct — stat-based highlight reaction |
| "LeBron is the GOAT no debate" | hot_take | 0.68 | Correct — bold unsubstantiated claim |
| "Hart has 18 fouls in 108 minutes, Castle has 21 in 185 minutes — very lenient on Castle" | analysis | 0.62 | Correct — specific foul rate comparison |
| "Clippers Draft Pick Ngoy Decides to Return to Auburn" | hot_take | 0.36 | Wrong — should be reaction (news fact) |
| "LMFAOOOOOOOOO" | reaction | 0.74 | Correct — pure emotional reaction |

---

## Reflection: What the Model Learned vs. What I Intended

I intended the model to learn the difference between structured argumentation (`analysis`), unsupported assertion (`hot_take`), and emotional response (`reaction`). What it actually learned was a much simpler heuristic: **short posts with assertive or emotional language → hot_take; longer posts → analysis or reaction**.

The model severely over-predicted `hot_take`, collapsing much of the `reaction` category into it. This makes sense given that many `reaction` posts are short and emphatic ("LeBully", "LMFAOOOOOOOOO", "does he know we can see him?") — structurally similar to hot takes. The model never learned to distinguish between assertive opinions and emotional reactions, because the surface-level text features overlap significantly.

The fine-tuned model performing worse than the zero-shot baseline tells me that 202 examples — especially with only 43 in the `analysis` class — was not enough for DistilBERT to learn the subtle distinction I was measuring. The zero-shot LLM had the advantage of world knowledge and language understanding that let it make reasonable guesses even without training examples.

---

## Spec Reflection

**One way the spec helped:** The requirement to document at least 3 difficult-to-label examples before annotating forced me to sharpen my label boundaries early. Writing the decision rule for the `hot_take` vs. `analysis` edge case (one stat used rhetorically vs. evidence that builds a genuine argument) made my annotation more consistent and gave me a clear framework for the hardest cases.

**One way my implementation diverged:** The spec suggests fine-tuning should outperform the baseline. In my case it did not — the fine-tuned model regressed by 16 points. This happened because 202 examples is at the low end for DistilBERT to learn subtle discourse distinctions, and my `analysis` class was underrepresented at only 43 examples. If I were to redo this, I would collect 100+ analysis examples specifically — that label requires the most nuance and had the least data.

---

## AI Usage

**Instance 1 — Label stress-testing:** I gave Claude my label definitions and asked it to generate 10 posts sitting at the boundary between `hot_take` and `analysis`. This helped me identify the "one-stat rhetorical take" edge case before I annotated 200 examples, and I wrote a specific decision rule for it in my planning document.

**Instance 2 — Annotation assistance:** I used Claude to pre-label batches of 30–40 posts at a time by providing my label definitions and unlabeled post text. Claude assigned one label per post; I reviewed and corrected every single pre-assigned label before adding it to my CSV. I flagged pre-labeled examples in the `notes` column of my dataset. Approximately 60% of examples were pre-labeled and reviewed; 40% were labeled entirely by me from scratch.

**Instance 3 — Failure pattern analysis:** After fine-tuning, I pasted my 15 misclassified examples into Claude and asked it to identify common themes. Claude identified the pattern of `reaction` posts being misclassified as `hot_take` due to short assertive language — a pattern I verified by re-reading the examples myself. Claude also suggested that `analysis` posts written in game-thread style might confuse the model, which I confirmed was true for Failure #3 above.
