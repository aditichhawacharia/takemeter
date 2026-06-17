# TakeMeter — Planning

## Overview

I'm building a classifier for r/TaylorSwift that sorts posts into four categories: Art, News/Factual Update, Substantive Analysis, and Open-Ended Question Discussion. These distinctions matter to the community because each type asks for a different kind of engagement — a News post wants an upvote and maybe a confirmation, an Art post wants appreciation, a Substantive Analysis post wants critical engagement with a specific claim, and an Open-Ended Discussion post wants people to share their own stories and opinions — so collapsing them together (or mislabeling one as another) misreads what the poster is actually asking the community to do.

## Community

I'm classifying posts from r/TaylorSwift. I chose this community because it has unusually wide variance in post *intent* within a single fandom space: the same album drop generates pure fact-reporting ("certified Platinum"), structured argumentative essays with cited evidence, and loosely-argued vibe checks that are dressed up as analysis but are really asking for emotional validation. That last category — analysis-shaped posts that are functionally discussion-bait — is what makes this discourse genuinely hard to classify rather than trivially sortable by length or vocabulary, which is exactly the kind of ambiguity this project needs.

## Labels

**Art** — shares a fan-made creative work (visual art, edits, design) inspired by Taylor Swift, where the post's primary content is the artifact itself.
- "made a folklore wallpaper (points for spotting the pride flag)"
- "Taylor's Albums in the Showgirl font!"

**News/Factual Update** — reports something that happened (release, chart, tour, quote) with little to no added interpretation.
- "Taylor Swift's new single debuts at #1 on Billboard Hot 100."
- "Taylor Swift's Courtside Seat from Knicks-Cavaliers Playoff Game Sells at Auction for $7,000"

**Substantive Analysis** — makes a specific, evidence-backed claim or interpretation (lyric connections, songwriting craft, career patterns), regardless of whether it's phrased as a statement or a question.
- "The Theory of Everything: Taylor Swift, TLOAS Edition" — a long-form post mapping a three-character framework (Poet/Brand/Showgirl) onto specific tracks, with cited interpolation sources and a follow-up fact-check edit.
- "Taylor Swift Isn't Complaining, She's Confessing" — an essay arguing TLOAS and TTPD share a thesis about fame and material success, built around close reading of specific lines and a sustained analogy.

**Open-Ended Question Discussion** — broad, speculative, or highly subjective prompts designed for lengthy, story-driven comments from the community.
- "What's your favorite Taylor soundtrack song? Least favorite?"
- "What songs would be best performed on a Broadway stage?"

## Hard edge cases

The hardest recurring case is the **analysis-shaped vibe check**: a post that opens with a discussion-style hook ("does anyone else think..."), but then builds real argumentative scaffolding — comparative claims, a semi-quantified observation, a structural theory — before closing with an explicit ask for other people's personal reactions rather than a rebuttal of the claim.

Concrete example: "does anyone else think the album is just... fine? like sonically it's nice but I don't feel anything." The post argues that TLOAS has lower lyrical density than recent albums, supports this with a specific comparison (counting internal callbacks/motifs across TTPD vs. Showgirl) and a claim about why "joy" is harder to write specifically than "devastation" — that's evidence-backed interpretation, which is what Substantive Analysis is defined to catch. But the framing device on both ends ("does anyone else," "I want to understand what I'm missing because everyone seems more moved than I am") is doing real work: it's soliciting validation and personal reactions, not inviting engagement with the argument itself.

**How I'll handle it:** I'm adopting a load-bearing-claim rule, not a punctuation rule. If the post contains a specific, falsifiable claim supported by evidence or close reading — even if it's wrapped in casual or question-shaped framing — it's Substantive Analysis. If the post's only content is a prompt for others to share preferences/feelings/stories with no claim of its own to evaluate, it's Open-Ended Question Discussion, even if it gestures at a reason. I will not use "has a question mark" or "uses casual language" as a deciding signal, since both labels contain examples of each. When a post is genuinely 50/50 under this rule (evidence present but thin, framing strongly validation-seeking), I'll log it in a difficult_cases.md file with my reasoning rather than silently picking one, so I can audit consistency once I've labeled ~30-40 posts and revisit the rule if I'm flip-flopping.

A secondary edge case worth flagging during annotation: art posts whose entire content is an illustration of a fan theory (e.g., "recreated the Showgirl cover but made every sequin a different fan theory"). I'll classify by primary artifact type — if the post is fundamentally an image/visual object, it's Art, even if the caption or top comments unpack an analytical idea — since the model is learning from the post body, not from how deep the comment section goes.

## Data collection plan

I'll pull posts from r/TaylorSwift using Reddit's API (PRAW) across hot, top (week/month/year), and new, to avoid overrepresenting whatever's algorithmically trending right now — a pure "hot" pull would oversample News/Factual Update around release weeks and starve the other labels. Target is roughly 50 examples per label (200 total), pulling extra during initial collection (~250-300 raw posts) since some will be unclassifiable noise (giveaways, meta posts about the sub itself, removed/deleted content) that I'll discard rather than force into a label.

If a label is underrepresented after the initial pull — Art is the most likely candidate, since image-only posts are less searchable by keyword than text posts — I'll do a targeted second pass filtering for flair (r/TaylorSwift uses post flairs like "Fan Art," "News," "Discussion") and sort by top within that flair across a longer time window (year) rather than lowering my bar for what counts as belonging to the underrepresented label.

## Evaluation metrics

Accuracy alone will hide a meaningful failure mode here: a model that's bad specifically at the Substantive Analysis vs. Open-Ended Discussion boundary (the hardest one) could still post a respectable overall accuracy if Art and News are easy and well-represented. So I'll report:

- **Per-class precision, recall, and F1**, not just macro accuracy, so a model that's quietly bad at one label can't hide behind being good at the other three.
- **A full confusion matrix**, since the actual value of this project is seeing *which* labels get confused with which — I expect most of the matrix's off-diagonal mass to sit between Substantive Analysis and Open-Ended Discussion specifically, and the matrix is the only metric that shows that directly.
- **Qualitative error review** of at least 3 wrong predictions per error type, since for a subjective discourse-quality task, knowing *that* the model is wrong matters less than knowing *why* — e.g., did it key on surface features like question marks instead of the actual content of the claim.

## Definition of success

A genuinely useful version of this classifier would hit roughly 75-80% accuracy overall, with no single class falling below 65% F1 (so it's not just acing the easy labels and ignoring a hard one), and confusion concentrated almost entirely on the Substantive-Analysis/Open-Ended-Discussion boundary rather than scattered across all label pairs — scattered errors would mean the model hasn't learned the taxonomy at all, while concentrated errors on one known-hard boundary would mean it's learned the easy distinctions and is struggling exactly where I predicted a human would also struggle.

"Good enough for deployment" in a real community tool (e.g., auto-flair suggestion) means: overall accuracy at or above the zero-shot Groq baseline by a meaningful margin (not just statistically distinguishable — I'd want at least a 10-point accuracy gap to justify the fine-tuning effort over just prompting), and per-class recall high enough on News/Factual Update and Art specifically that those two labels could be auto-applied with minimal human review, while Substantive Analysis and Open-Ended Discussion would still get flagged for a human glance given their known overlap. These thresholds are specific enough that I can check them directly against my evaluation report's confusion matrix and per-class table at the end, rather than eyeballing whether it "feels" good.

## AI Tool Plan

**Label stress-testing:** Before annotating, I gave my four label definitions and the analysis-vs-discussion edge case to Claude and asked it to generate boundary-straddling posts for each label pair. It produced the "does anyone else think the album is just... fine" example above, which is what surfaced the need for the load-bearing-claim rule instead of a punctuation-based rule. I'll repeat this for the Art/Substantive-Analysis boundary (theory-illustrating fan art) before I start the real 200-post annotation pass, and will tighten definitions again if the generated examples don't sort cleanly under my current rules.

**Annotation assistance:** I will use an LLM (Groq's llama-3.3-70b-versatile, the same model used for the baseline, to keep my pre-labeling and baseline comparison architecturally separate from anything used in training) to pre-label the full raw pull before I review it myself. I'll add a `pre_labeled_by_llm` boolean column to my dataset alongside my own final `human_label` column, so every example's provenance is traceable, and I will disclose in my README that initial labels were LLM-suggested and human-confirmed/corrected, not human-originated from scratch. I will not treat LLM pre-labels as ground truth on the hard edge cases — those get manually re-read in full regardless of what the pre-label says.

**Failure analysis:** After generating my fine-tuned model's wrong predictions on the test set, I'll give the full list (post text, true label, predicted label) to an AI tool and ask it to propose systematic patterns rather than treat each error as independent — e.g., "is the model defaulting to Open-Ended Discussion whenever a post contains a question mark, regardless of content." I will verify any proposed pattern myself by pulling the actual subset of errors it's describing and re-reading them, rather than reporting an AI-suggested pattern in my evaluation writeup without checking it against the real examples first.
