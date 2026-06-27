# TakeMeter — Planning Document

## Community

I chose **r/nba**, the NBA subreddit, as my community. It's one of the most active sports communities on Reddit, with thousands of posts and comments daily spanning game reactions, player debates, historical comparisons, and bold predictions. The discourse quality varies enormously — some posts make careful statistical arguments, others are pure emotional venting, and many are confident opinions stated as facts with no evidence. This variation makes it an ideal fit for a classification task: the distinctions I care about are real, they matter to participants, and they show up consistently enough to build a 200-example dataset from.

---

## Label Taxonomy

I'm using three labels:

**`analysis`**
A post that makes a structured argument supported by specific, verifiable evidence — statistics, historical comparisons, game film observations, or tactical breakdowns. The evidence would still support the claim even if you removed the opinion framing.

- Example 1: *"Over the last 3 seasons, Jokic leads all centers in assists per game (9.1), PER (31.4), and True Shooting % (.638). There's no statistical case for anyone else as the best big man in the league."*
- Example 2: *"In Game 7s since 2010, LeBron's PER is 28.3 vs. a regular season average of 27.9 — the 'LeBron disappears in big moments' narrative doesn't match the data."*

**`hot_take`**
A bold, confident opinion stated without meaningful supporting evidence. The post asserts a claim rather than arguing for it. The framing is often provocative or exaggerated.

- Example 1: *"Steph Curry would not survive in the 90s. He's a product of the modern game and soft defenders."*
- Example 2: *"The Warriors dynasty is the most overrated run in NBA history. They had to add KD because they couldn't win without a superteam."*

**`reaction`**
An immediate emotional response to a specific, recent game or event. The post is expressing a feeling in the moment — excitement, frustration, disbelief — with little to no argument or evidence.

- Example 1: *"BRO WHAT WAS THAT SHOT FROM TATUM I'M LOSING MY MIND 🤯"*
- Example 2: *"Can't believe we just watched that. This team is cooked. I'm done for the season."*

---

## Hard Edge Cases

**The hardest case: a post that uses one stat to support a hot take.**

Example: *"LeBron is overrated — his playoff win rate against top-seeded opponents is below .500."*

This looks like `analysis` because it cites a stat, but the stat is cherry-picked and decorative — it's not part of a genuine argument, just a number dropped to make the take sound credible.

**Decision rule:** If the post provides specific, verifiable evidence that would support the claim even after removing the opinion framing, label it `analysis`. If the evidence is vague, cherry-picked, or exists only to lend credibility to an assertion rather than actually reason through it, label it `hot_take`. One isolated stat selected for rhetorical effect → `hot_take`. Multiple pieces of evidence building a case → `analysis`.

**Second hard case: a reaction post that includes a mini-opinion.**

Example: *"That loss hurt. Refs were terrible all game, this league is rigged."*

This is primarily emotional (`reaction`), but it includes a claim ("refs were terrible," "this league is rigged"). Decision rule: if the dominant register is emotional and event-specific, label it `reaction`. The claim has to be the focus of the post, not a throwaway comment, to qualify as `hot_take`.

---

## Data Collection Plan

**Source:** r/nba — post titles and comment threads from the past 6 months. I'll collect manually by browsing top posts and sorting by "Hot" and "Top" across different time windows to get variety.

**Target distribution:** ~70 examples per label (analysis, hot_take, reaction) for a total of 210 examples. This gives each label at least 33% of the dataset, avoiding majority-class dominance.

**If a label is underrepresented:** `analysis` posts are rarer than hot takes and reactions on Reddit, so if I'm below 60 after 200 examples, I'll specifically browse threads tagged as "Film Study," "Deep Dive," or long-form posts to find more analytical content. I'll also search for posts from accounts like stat-heavy commenters or journalists.

**CSV format:** Two required columns (`text`, `label`) plus a third `notes` column for flagging difficult cases during annotation.

---

## Evaluation Metrics

I'll use the following metrics:

- **Overall accuracy** — the baseline number, useful for high-level comparison between the zero-shot and fine-tuned models.
- **Per-class F1** — the most important metric for this task. Because my labels aren't perfectly balanced and each represents a meaningfully different failure mode, accuracy alone hides whether the model is just predicting the majority class. F1 per label tells me if the model actually learned each distinction.
- **Confusion matrix** — to identify which label pairs are being confused and in which direction. I expect the hardest boundary to be `hot_take` vs. `analysis` (a stat-backed take vs. a genuine argument), and the confusion matrix will confirm or disprove that.
- **Precision and recall per class** — to understand whether the model is over-predicting or under-predicting each label.

Accuracy alone is not enough because a model that predicts `hot_take` 70% of the time would still get ~50% accuracy if that's the most common label — but it would fail completely on `analysis` and `reaction`.

---

## Definition of Success

I'll consider this classifier genuinely useful if:

- Fine-tuned model overall accuracy **≥ 70%** on the test set
- Per-class F1 **≥ 0.60** for all three labels (no label is completely missed)
- Fine-tuned model beats the zero-shot baseline by **at least 10 percentage points** in overall accuracy

If the fine-tuned model's F1 for any single label is below 0.40, I'll treat that as a failure and investigate whether it's a data distribution problem or a label boundary problem before concluding.

---

## AI Tool Plan

**Label stress-testing:** Before annotating, I'll give Claude my three label definitions and the edge case description and ask it to generate 10 posts that sit at the boundary between `hot_take` and `analysis`. If I can't cleanly label those generated examples using my decision rule, I'll tighten the definitions before annotating 200 real posts.

**Annotation assistance:** I will use Claude to pre-label batches of 30–40 posts at a time by providing my label definitions and the unlabeled text. I'll review and correct every pre-assigned label myself — I won't accept pre-labels without reading each post. I'll mark pre-labeled examples with a flag in the `notes` column of my CSV and disclose this in the AI usage section of my README.

**Failure analysis:** After fine-tuning, I'll paste my misclassified test examples into Claude and ask it to identify common patterns — post length, sarcasm, label pairs, vague language. I'll then verify those patterns by re-reading the examples myself and only include patterns I can confirm. I'll note any patterns Claude suggested that I had to correct or discard.
