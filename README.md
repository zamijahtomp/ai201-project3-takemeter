# TakeMeter — r/houseplants Post Classifier

A text classifier that sorts r/houseplants posts into four categories — **Discussion**, **Haul**, **Before/After - Progress Pics**, and **Humor/Fluff** — comparing a zero-shot LLM baseline (Groq) against a DistilBERT model fine-tuned on 200 hand-labeled posts.

Design decisions, label definitions, edge-case rules, and the data collection/AI-tool plan are documented in [planning.md](./planning.md); this README covers the final build, results, and analysis.

## What this is

r/houseplants is a single-topic community where every post is nominally "about a plant," but the actual *purpose* of each post varies widely — someone asking for care advice, someone showing off a purchase, someone documenting months of growth, or someone just cracking a joke about their overgrown pothos. TakeMeter automatically sorts posts into those four intents (Discussion, Haul, Before/After - Progress Pics, Humor/Fluff), which could help a community tool auto-suggest post flair or help moderators and readers filter the feed by what kind of post they're actually looking for, rather than relying on posters to self-tag consistently.

## Dataset

- 200 posts manually collected and labeled from r/houseplants (see [planning.md](./planning.md) for the collection process and label definitions).
- Split 70/15/15 into train (140) / validation (30) / test (30), stratified evenly across all four labels.
- Labels: Discussion, Haul, Before/After - Progress Pics, Humor/Fluff — full definitions and examples in planning.md.

## Models

- **Baseline:** zero-shot classification via Groq (`openai/gpt-oss-120b`), prompted with the four label definitions and one example per label.
- **Fine-tuned:** `distilbert-base-uncased`, fine-tuned on the 140-example training split.
  - Final hyperparameters: `num_train_epochs=15`, `learning_rate=2e-5`, `per_device_train_batch_size=16`, `warmup_steps=5`.
  - Note: the first training run used `num_train_epochs=3` with `warmup_steps=50` — with a dataset this small (140 examples, batch size 16), that produced only 27 total optimization steps, and the warmup schedule alone exceeded the entire run. The model never moved off its random initialization (loss stuck at `ln(4) ≈ 1.386`, predicting a single class 100% of the time). Reducing warmup to 5 steps and extending training to 15 epochs fixed this — see the Reflection section below.

## Evaluation Report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Baseline (zero-shot Groq) | 69.23% |
| Fine-tuned (DistilBERT) | 43.33% |

The Baseline currently outperforms the fine-tuned model due to subtle differences in authors' tones.

### Per-class metrics — fine-tuned model

*(computed from the confusion matrix below; test set n=30)*

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| Discussion | 0.44 | 0.50 | 0.47 | 8 |
| Haul | 0.20 | 0.14 | 0.17 | 7 |
| Before/After - Progress Pics | 0.54 | 0.88 | 0.67 | 8 |
| Humor/Fluff | 0.33 | 0.14 | 0.20 | 7 |

### Per-class metrics — baseline model

*(evaluated on 13/30 test examples that produced a parseable label; 17/30 baseline responses were unparseable — see note below)*

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| Discussion | 1.00 | 0.40 | 0.57 | 5 |
| Haul | 0.67 | 1.00 | 0.80 | 2 |
| Before/After - Progress Pics | 0.67 | 1.00 | 0.80 | 4 |
| Humor/Fluff | 0.50 | 0.50 | 0.50 | 2 |
| **macro avg** | **0.71** | **0.72** | **0.67** | **13** |

Note: the baseline's 69.23% accuracy is computed only over the 13 test examples where the Groq model's raw output could be parsed into one of the four valid labels; the other 17/30 produced empty or malformed responses (see Models section above for the `max_tokens`/parsing fixes made during development). This means the baseline number is optimistic relative to the fine-tuned model's accuracy, which is computed over all 30 test examples — worth flagging as a fairness caveat when comparing the two headline accuracy numbers directly.

### Confusion matrix — fine-tuned model (test set)

Rows = true label, columns = predicted label.

| True \ Predicted | Discussion | Haul | Before/After - Progress Pics | Humor/Fluff |
|---|---|---|---|---|
| **Discussion** | 4 | 2 | 2 | 0 |
| **Haul** | 2 | 1 | 3 | 1 |
| **Before/After - Progress Pics** | 0 | 0 | 7 | 1 |
| **Humor/Fluff** | 3 | 2 | 1 | 1 |

A full-resolution image version is also committed as [confusion_matrix.png](./confusion_matrix.png).

### Analysis: where the fine-tuned model fails

**Pattern 1 — Humor/Fluff is consistently read as Discussion or Before/After.** Examples:

- *"I didn't even know these could do that?!?!? I nursed this little Chinese Evergreen back to life from one sad leaf..."* — True: Humor/Fluff, Predicted: Discussion (confidence 0.65)
- *"Saving Grace. If it wasn't for lovely surprises like this I'd chuck my Pink Polka Dot in the bin because... it's a pinkpolkadot pain in the bum!"* — True: Humor/Fluff, Predicted: Discussion (confidence 0.68)
- *"Love this corner. Looking at my plant wall makes me so happy! Yes some are struggling, im working on it!"* — True: Humor/Fluff, Predicted: Discussion (confidence 0.53)

**Why this boundary is hard:** the humor in these posts is tonal — a joke embedded in an exclamation, an emoji, or self-deprecating phrasing — while the surrounding sentence structure is functionally identical to a sincere care-update or progress post. This is exactly the ambiguity flagged in advance in planning.md's Hard Edge Cases section (Progress Pics vs. Humor/Fluff): a bag-of-words-level model has no mechanism to detect sarcasm or tone, so it defaults to the literal, surface-level content, which reads as Discussion or Progress.

**Pattern 2 — Haul is frequently swallowed into Before/After - Progress Pics.** Examples:

- *"My baby jungle! 🪴 Almost all of these started from propagations but a few I have plantnapped from my parents..."* — True: Haul, Predicted: Before/After - Progress Pics (confidence 0.33)
- *"my new philodendrons, white+pink princess☺️"* — True: Haul, Predicted: Before/After - Progress Pics (confidence 0.58)
- *"Bathroom window plants. Plants names, starting from left to right: Rhaphidophora tetrasperma, Marble pothos..."* — True: Haul, Predicted: Before/After - Progress Pics (confidence 0.50)

**Why this boundary is hard:** the Haul label definition centers the *purchase* as the post's focus, but many real Haul posts caption casually ("my new X," "baby jungle") without ever using explicit acquisition language ("bought," "purchased," "picked up"). Textually, these are close to indistinguishable from someone showing off an established collection — Before/After's territory.

**Is this a labeling problem or a data problem?** These posts were labeled consistently (Haul and Humor/Fluff by their *intent*, matching planning.md's definitions), so this isn't annotation inconsistency — it's a training data distribution gap. Both labels are defined by author *intent*, which isn't always lexically marked in the text, and with only 35 training examples per label, the training set likely didn't contain enough of the "casual, no purchase-language" Haul posts or "flat-affect humor" posts for the model to learn those as valid members of the class. The model instead leans on the concrete, describable cues, which for these two labels are sparse.

**What would fix it:** more Haul training examples that lack explicit purchase language, and more Humor examples where the joke lives in tone rather than an explicit humor cue (e.g., "lol," exclamation points) — i.e., diversifying the *hard* cases within each class rather than adding more easy, obvious ones. A tighter Haul definition that explicitly calls out "casual acquisition mentions count too" would also help at the labeling stage.

### Sample Classifications

| Post (truncated) | True label | Predicted | Confidence | Correct? |
|---|---|---|---|---|
| Did my monstera serianna throw me a random unicorn?! | Discussion | Discussion | 0.63 | yes |
| Do Marantas Flower?! | Discussion | Discussion | 0.61 | yes |
| My Alocasia Silver dragon. The first pic is what my SD looked like when I first brought it... | Discussion | Discussion | 0.39 | yes |
| Variegated Chiapense never disappoints. Expresses variegation differently on every leaf | Discussion | Before/After - Progress Pics | 0.56 | no |
| Just a plant fest haul! In case you can't tell, if something is pink, purple, striped, or... | Haul | Discussion | 0.50 | no |

The "Do Marantas Flower?!" prediction is reasonable: the post is phrased as a direct question inviting a factual answer from the community, which is the clearest possible signal for the Discussion label — no purchase, no photo-progress framing, and no humor markers, so the model has an unambiguous lexical cue (a question mark plus a plant-care query) to key off of.

## Reflection: what the model captured vs. what I intended

The fine-tuned model learned a strong, lexically-grounded signal for the two labels that have concrete textual markers: Before/After - Progress Pics (highest recall, 7/8) picks up on time-and-growth language ("update," "months ago," "then vs. now"), and direct-question Discussion posts ("Do Marantas Flower?!") are easy for it because a question mark plus a plant-care noun phrase is an almost unambiguous cue. What it did not learn were the two labels I defined by author *intent* rather than surface content — Haul (a purchase-focused post) and Humor/Fluff (an entertain-first post). Both of those require reading *why* someone is posting, not just *what* they're describing, and with only 35 training examples per label, the model didn't see enough variety of "casual, no purchase-language" Haul posts or "tone-only, no explicit joke-marker" Humor posts to generalize past surface plant-photo-caption patterns. In short, my label scheme mixes two different kinds of signal — topic/structure vs. intent/tone — and a small model trained on this little data picks up the former far faster than the latter.

Separately, the training process itself surfaced an important lesson about "what the model captured": for most of my debugging session, the honest answer was *nothing*. My first training run (`num_train_epochs=3`, `warmup_steps=50`) produced a model that predicted a single class 100% of the time, with confidence scores clustered at ~0.25–0.27 — the uniform-random value for a 4-class problem — and a validation loss stuck at `ln(4) ≈ 1.386`, the theoretical loss of a completely untrained classifier. The model had 27 total optimization steps to work with (140 training examples ÷ batch size 16 × 3 epochs), and a warmup schedule longer than the entire training run meant the learning rate never reached its target value. It wasn't a hard-boundary problem; it was a model that never left initialization. Reducing warmup to 5 steps and extending training to 15 epochs let it actually learn, taking accuracy from 23% to 43%. This was a useful reminder that a flat, uniform-confidence failure mode should be diagnosed as a training bug before it's written up as a finding about task difficulty.

## Spec Reflection

The project spec's own checkpoint guidance — "if the fine-tuned model performs worse than the baseline across the board, that's a signal worth investigating before writing up your report" — directly caught a real bug in my training run. Without that prompt, I could easily have written up a 23% fine-tuned accuracy (worse than baseline, worse than random guessing) as a legitimate "the boundary is just hard" finding, when it was actually a `warmup_steps`/total-steps misconfiguration that left the model untrained.

Where I diverged from the spec: I kept a small number of non-English posts (French, Portuguese) in my dataset instead of filtering to English-only. This was a deliberate choice tied to my Community section in planning.md — I wanted the dataset to reflect the actual, realistic makeup of r/houseplants rather than an artificially cleaned English-only slice. I translated each one into English for the `text` column (rather than leaving the original-language text, which a bag-of-tokens model trained mostly on English couldn't use) and noted the original language in the `notes` column for disclosure. The tradeoff: with only ~200 total examples, a couple of translated posts don't meaningfully teach the model to handle multilingual input — that would require a much larger, deliberately multilingual dataset — so this was more a realism/documentation choice than one that meaningfully changed model behavior.

## AI Usage

I used Claude (Sonnet 5) for several parts of this project:

1. **Groq classification prompt and parsing debugging.** After building my `classify_with_groq()` function, I hit 401 errors (an incorrectly-copied API key), then 404 errors (a deprecated model name), then a parsing failure where 30/30 responses were marked "unparseable" despite the API succeeding. Working through this with Claude, I found and fixed: an API key that had been truncated when I copied it into Colab's Secrets manager, a `max_tokens=20` limit that was too low and cutting off model output entirely, a case-sensitivity bug in my label-matching code (comparing a lowercased model response against non-lowercased `LABEL_MAP` keys, which silently failed even on exact semantic matches), and a missing prefix-match case for when the model returned a truncated version of a multi-word label (e.g., `"before/after"` instead of the full `"Before/After - Progress Pics"`).

2. **Diagnosing a silent fine-tuning failure.** My first fine-tuned model scored 23.33% accuracy — worse than both the baseline and random guessing — and its confusion matrix showed it predicting a single class ("Haul") for all 30 test examples. I described this behavior to Claude, which identified that a validation loss plateaued almost exactly at `ln(4) ≈ 1.386` (the theoretical loss for a uniformly-random 4-class classifier), and confidence scores near 0.25–0.27 — both signs the model had never left random initialization. Claude helped me rule out a frozen-layers bug (I searched my notebook for `requires_grad`/`freeze` and found nothing) before we identified the actual cause: `warmup_steps=50` in my `TrainingArguments` exceeded my total training run of only 27 optimization steps, meaning the learning rate never reached its target value. I fixed this by lowering warmup to 5 steps and extending training to 15 epochs, which brought fine-tuned accuracy up to 43.33%.

3. **Failure-pattern analysis.** After retraining, I pasted my fine-tuned model's 17 misclassified test examples (text, true label, predicted label, confidence) into Claude and asked it to identify patterns. It proposed two: Humor/Fluff posts being misclassified as Discussion or Before/After because the humor is tonal (sarcasm/self-deprecation) rather than lexically marked, and Haul posts being misclassified as Before/After because many real Haul posts don't use explicit purchase language ("bought," "purchased"). I verified both patterns myself by re-reading the flagged posts and cross-checking against the confusion matrix numbers before including them in the Evaluation Report above — both held up.

**Annotation:** I hand-labeled all 200 examples myself while reading each post; I did not use LLM pre-labeling during annotation.

## Demo

[Plant Meter Agent, Intents and Accuracy Results](https://www.loom.com/share/97f5b6abebee428d8a15bf3565fd1503)
