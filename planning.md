# TakeMeter — planning.md

## Community

<!-- What community did you choose and why? Why is this community a good fit for a classification task — what makes the discourse varied enough to be interesting? -->

I chose **r/houseplants** — a community focused on the discussion, care, and well-being of houseplants — because I love houseplants and have a ton of them. It's a good fit for classification because the discourse naturally splits into distinct *purposes* even though every post shares the same subject matter (plants): some posts exist to ask for opinions/advice, some exist to show off a purchase, some exist to show growth over time, and some exist purely for entertainment. That mix of intents on a single topic gives a classifier something real to learn, rather than just topic-matching keywords like "monstera" or "pothos."

## Labels

<!-- What are your 2–4 labels? Define each in a complete sentence. Include 2 example posts per label. -->

### Discussion

A Discussion post is one where the user is asking a question, seeking opinions, or inviting debate about a plant-related topic (e.g., care advice, diagnosing problems, product recommendations).

- <https://www.reddit.com/r/houseplants/comments/1wj7uj7/hey_guys_im_looking_to_start_this_monstera/>
- <https://www.reddit.com/r/houseplants/comments/1wj49g3/i_got_tired_of_typical_grow_lights_not_matching/>

### Haul

A Haul post is one where the user is showing off plants, pots, or supplies they recently purchased or acquired, with the purchase itself as the focus of the post.

- <https://www.reddit.com/r/houseplants/comments/1wiricz/plant_shelves/>
- <https://www.reddit.com/r/houseplants/comments/1wj58wx/second_day_of_the_succulent_sale/>

### Before/After - Progress Pics

A Progress Pics post is one where the user shares photos documenting how a plant (or plant setup) has changed or grown over a defined span of time.

- <https://www.reddit.com/r/houseplants/comments/1w1kasy/update_pothos_setup/>
- <https://www.reddit.com/r/houseplants/comments/1wiby27/37_years_ago_today_my_parents_were_gifted_this/>

### Humor/Fluff

A Humor/Fluff post is one whose primary purpose is to entertain or amuse — a funny story, joke, or lighthearted moment involving a plant — rather than to inform, sell, or track progress.

- <https://www.reddit.com/r/houseplants/comments/1wgemdo/i_need_a_new_house/>
- <https://www.reddit.com/r/houseplants/comments/1wiymeh/my_green_space/>

## Hard edge cases

<!-- What type of post will be genuinely ambiguous between two labels? How will you handle it when you encounter it during annotation? -->

The most common ambiguity is **Progress Pics vs. Humor/Fluff**: a post can show a plant that has clearly grown/changed (Progress Pics signal) while the caption is a joke (Humor signal) — e.g., "he ate that" over a photo of an overgrown vine. A second recurring ambiguity is **Haul vs. Discussion**: someone posts a purchase but also asks "was this a good price?" or "is this healthy?", which pulls toward Discussion even though the post frames itself around an acquisition.

My rule for resolving these during annotation:

1. **Title/caption intent wins over image content.** If the title or top-level text is asking a question or inviting opinions, it's Discussion, even if the photo alone would suggest another label.
2. **Primary purpose over secondary purpose.** If a Haul post's main framing is "look what I got" and the price question is incidental, it stays Haul; if the question is the point and the purchase is incidental context, it's Discussion.
3. When I genuinely can't decide after applying rules 1–2, I'll log the post in a running "ambiguous cases" list with my reasoning and default to the label whose definition it satisfies more completely, so I can revisit the set later if a pattern emerges that suggests a definition needs tightening (rather than adding a new label mid-annotation).

## Data collection plan

<!-- Where will you collect examples? How many per label? What will you do if a label is underrepresented after 200 examples? -->

I'll collect examples primarily from r/houseplants on Reddit, pulling from both "New" and "Top" sorts (and different time windows) so I don't only capture whatever is currently trending. If Reddit alone doesn't give me enough variety, I'll supplement with houseplant Facebook groups, since the same four post types (advice, purchases, progress, humor) show up there too.

Target: ~50 examples per label, for a total of ~200 examples across 4 labels.

If a label is underrepresented after collecting 200 examples total (e.g., Humor/Fluff only has 15 examples instead of 50):

- First, I'll widen the search — different subreddits with similar audiences, older posts, different keywords — before assuming the label is just rare.
- If it's still underrepresented after that, I will **not** silently fold it into another label, since that would erase a distinct discourse type. Instead I'll either (a) accept the smaller count and note the class imbalance explicitly in my evaluation write-up, using per-class metrics that won't be misleadingly inflated by imbalance, or (b) if the count is too small to be usable at all (under ~15–20 examples), merge that label into its closest neighbor and document why, rather than adding a brand-new label this late.

## Evaluation metrics

<!-- Which metrics will you use to evaluate your model and why are those the right ones for this specific task? (Accuracy alone is not enough — explain what else you need and why.) -->

Accuracy alone is misleading here because my labels won't be perfectly balanced (Humor/Fluff is likely to be the rarest), so a model could score deceptively well by defaulting to the majority label. I'll use:

- **Per-class precision and recall**, not just an overall average — I need to know, for example, whether the model is bad specifically at recognizing Humor/Fluff (low recall for that class) even if it's great at Discussion, since that's exactly the kind of failure overall accuracy would hide.
- **Macro-averaged F1** as my headline number, because it weights all four labels equally regardless of how many examples each has, which matters given the expected class imbalance.
- **A confusion matrix**, so I can see *which* labels get confused with which — this directly ties back to the hard edge cases (Progress Pics vs. Humor, Haul vs. Discussion) and tells me whether errors cluster where I already expect ambiguity or show up somewhere unexpected.

Together these tell me not just "how often is it right" but "is it right for the right reasons, on every label, in the ways I predicted vs. didn't."

## Definition of success

<!-- What performance would make this classifier genuinely useful? What would you accept as "good enough" for deployment in a real community tool? -->

**Genuinely useful** target: macro-F1 ≥ 0.75, with no single class's F1 below 0.65. This means the classifier is reliably distinguishing all four post types, not just acing the easy majority label while quietly failing on a rarer one.

**"Good enough" for a real deployment** (e.g., auto-suggesting a flair to a moderator, not auto-applying it): macro-F1 ≥ 0.65, with no class below 0.5, AND the confusion matrix shows that most of the remaining errors fall into the edge cases I already identified (Progress Pics ↔ Humor, Haul ↔ Discussion) rather than being spread randomly across unrelated label pairs. Confusion concentrated in known-ambiguous pairs means the tool is failing in an *explainable, human-reviewable* way — a moderator glancing at a suggested flair can sanity-check it — whereas scattered, unexplainable errors would mean the model isn't actually learning the label distinctions and shouldn't be trusted even as a suggestion tool.

These thresholds are specific numbers I can check against my actual test-set results at the end, so I can objectively say whether I hit "genuinely useful," "good enough for deployment as a suggestion," or neither.

## AI Tool Plan

### Label stress-testing

Before annotating my 200 examples, I'll give an AI tool my four label definitions plus my hard-edge-case description above, and ask it to generate 5–10 synthetic r/houseplants-style posts specifically written to sit at the boundary between two labels (e.g., a purchase post that's secretly a question, or a progress-pic post with a joke caption). If I can't cleanly assign a label to any of these generated posts using my current definitions, I'll rewrite the relevant label definition(s) and/or add a tiebreaker rule to the Hard Edge Cases section — and I'll do this stress test *before* starting the 200-example annotation pass, not partway through.

### Annotation assistance

I will use an LLM to pre-label a batch of collected posts before I review them myself, since 200 examples is a lot to label cold. I'll use Claude (via this workflow) for pre-labeling, and I'll track which examples were pre-labeled vs. hand-labeled by adding a `pre_labeled_by_ai` boolean column in my annotation spreadsheet/CSV, so this is disclosable in my AI usage write-up. I will personally review and confirm/correct every pre-labeled example — the AI label is a starting suggestion, not a final answer.

### Failure analysis

After training and testing my model, I'll compile the list of misclassified examples (with true label, predicted label, and post text) and give it to an AI tool to identify patterns — e.g., "are most errors concentrated in one label pair," "do errors correlate with post length, missing text, or image-only posts," etc. I'll verify any pattern it identifies by manually re-reading a sample of the flagged misclassifications myself before including the claim in my evaluation write-up — I won't take the AI's pattern summary at face value without checking it against the actual posts.
