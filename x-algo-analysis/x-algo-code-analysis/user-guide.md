# X Reach User Guide

<!-- REVIEW-BASELINE repo=https://github.com/xai-org/x-algorithm.git rev=e1a0375a903899984545c663bb0bccb3cdcdf046 short=e1a0375 date=2026-10-13T08:05:54Z prev=85ac72a1bba41f21615e3f0bca56da75970a6633 range_commits=28 -->

### The non-obvious playbook, derived from the open X algorithm code

This guide condenses the actionable findings of [`algo-analysis.md`](algo-analysis.md) and [`account-analysis.md`](account-analysis.md) — based on the open-source releases of Aug 13–14, 2026 (commits `a389166` and `c65aa17`) **through the Oct 13, 2026 state (commit `e1a0375`)**, which exposed the scoring weights, visibility rules, credibility system, and model training code.

It deliberately skips the advice everyone already knows ("post engaging content", "be consistent"). Everything here is something the code actually says that most people don't know — including several things that contradict common platform advice.

**How to read the code references:** every fact below that comes from the source is cited inline with a link to the exact file in the open-source repo ([`xai-org/x-algorithm`](https://github.com/xai-org/x-algorithm)). The technical backing for each claim lives in [`algo-analysis.md`](algo-analysis.md); the scenario detail lives in [`account-analysis.md`](account-analysis.md).

**Don't know a term?** Every technical term used below — *model, candidate, embedding, author row, OON, VQV, DPP, label, index, PageRank,* and the rest — is defined in plain English in the **Glossary** at the end of this guide (just before the cheat sheet). The most important terms are also explained in place the first time they appear.

---

## Part 0 — The five ideas that make everything else make sense

1. **Every view is a separate decision.** Nothing is "broadcast" to your followers. Each person who opens the For You feed gets their own batch of ~1,000 candidate posts (posts competing for a place in their feed — Phoenix retrieval alone is capped at 1,000 — [`phoenix_source.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/sources/phoenix_source.rs), with Thunder, SimClusters, topics, and a cache piling on top), and the model (the AI that powers the feed) asks one question per (person, post): *how likely is THIS person to engage with THIS post?* Your "reach" is the sum of thousands of individual yes/no decisions. One consequence that shapes everything else: the **first** phase of a post's life is near-follower-only — followers are *offered* it instantly and unconditionally (whether it's shown is still per-viewer prediction), while the discovery channels only open once the post earns engagement (Part 2.11).

2. **Follower count is never used in scoring.** The ranking model does not receive your follower count as an input. Your followers are treated as people to be individually predicted, not as an audience to be reached. This one fact explains most of the "unfairness" people report.

3. **Every account carries a learned reputation that entrenches itself.** The model stores a learned "impression" of every account — a vector trained on millions of past outcomes that encodes *what happens when this account's posts are shown to people*. (Technically this stored impression is the account's *embedding* — a list of numbers — and from here on this guide calls it the account's *author row* or *learned reputation*.) Engagement begets more engagement; silence begets silence. This loop, not any rule, is the engine behind accounts that seem "shadow-banned" while others can't fail — and it sorts users into two self-reinforcing classes (the subject of Part 4).

4. **The feed is a tournament for ~35–47 seats.** ~1,000 candidates enter every request; roughly 35–47 survive (the rest are ads, "Who to Follow" boxes, etc. — the constants are `RESULT_SIZE = 35` and `FOR_YOU_MAX_RESULT_SIZE = 47` in [`config.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/config.rs), and the top-50 selection is `TOP_K_CANDIDATES_TO_SELECT`). Most candidates lose every single request. Your post competes against the whole feed, every time.

5. **The engagement numbers you see are symptoms, not causes.** The system shows your post mostly to the people it predicts will engage. So a high engagement rate can coexist with tiny reach (only superfans were served), and a huge reach can come with a low rate (millions of indifferent views diluted it). Never judge your account by its engagement rate.

---

## Part 1 — The signals that actually move your post

The published weights (they multiply *predicted probabilities*, not raw counts — see Part 2.7) rank the levers in this exact order. All of them live in [`home-mixer/params/param.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/param.rs), and the scoring formula that uses them is in [`ranking_scorer.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/scorers/ranking_scorer.rs):

| Signal | Weight | What it means for you |
|---|---|---|
| Share via copy-link | **20.0** | The single strongest *weight* in the system. Content people copy-paste to send to someone specific. |
| Reply (on originals from mutual follows) | **20.0** (5 + 15) | The mutual-follow boost. More in Part 2.11. |
| Reply | **5.0** | **~8× a like's weight** (like is now 0.6 — see the note below the table). |
| Quote / Share via DM | **5.0** | Both carry strong weights. Quotes also credit *you* (Part 2.4). |
| Follow the author | **4.0** | Posts that convert viewers into followers pay well. Note: profile *clicks* pay **0.0** — visits alone are worth nothing; the follow is what counts. |
| Share | **2.0** | Regular share button. |
| Like | **0.6** | Real, but ~8× weaker *weight* than a reply's. |
| Open link / Click | **0.2 / 0.3** | Weak (click was 0.4, now 0.3). Clicking a link barely moves the needle. |
| Dwell time (seconds actually spent looking at the post) | **0.006/sec, capped** | Max ≈ 0.18 points (predicted dwell caps at ~30 s) — less than a third of a like. Real but small; replies and shares dwarf it. |
| Click-dwell (sustained viewing after a click — NEW Oct) | **0.4** | The large continuous term — rewards holding attention post-click, not just the click. |
| Dwell (probability head) | **0.05** | A second, smaller dwell term. Still small next to replies/shares. |
| Photo expand | **0.05** | Small weight. |
| Video open | **0.02** | Cut from 0.07 in Oct. Small either way — video's real value is structural (Part 2.2). |
| Video quality view (VQV — the "watched most of it" signal) | **0.0** | Zeroed in Sept: VQV no longer scores at all. The 10-second gate survives only as an implicit discovery signal (Part 2.2, Part 5). |
| Post unexplored | **0.015** | Trimmed from 0.02 in Oct. In-network only. |
| Report | **−234.0** | See Part 2.7. |
| Mute | **−58.8** | See Part 2.7. |
| Not interested | **−47.52** | Tightened from −43.2 in Oct. See Part 2.7. |
| Block | **−31.2** | See Part 2.7. |

**How to read this table (important — the platform's own framing, VP of Product, Aug 15 2026):** every weight multiplies the model's *predicted probability* of that action for a specific viewer — a number between 0 and 1, not a count. A weight of 5.0 does **not** mean "worth 5 likes": the actual contribution is `5.0 × P(reply)`. Like report −234 (a rare action given a huge weight so it can matter at all), the big positive weights on copy-link shares (20.0) and DM shares (5.0) exist partly *because* those actions are rare — the model's predicted probability for them is typically far lower than for a like, and the weight compensates. So "10× a like" means "10× a like's *weight*," not "10 likes' worth of score": for a given viewer, a predicted like contributes `0.5 × P(like)` and a predicted DM-share contributes `5.0 × P(DM)` — whichever is larger depends on the probabilities, which are personalized per viewer and not published. The same logic cuts both ways: a big weight moves little if the model thinks *you* are unlikely to do that action, and a common action like a like can dominate the score despite its small weight.

The practical takeaway: **design every post around one question — "will someone reply to this, or copy-paste it to send to someone?"** Not "will people like it." Likes carry a small weight (0.6); replies and shares carry the big weights — the model is calibrated to notice them when it predicts them. **Oct addition: hold attention after the click** — the new 0.4 click-dwell term pays for sustained post-click viewing, so threads and long posts that keep a clicker reading now earn a continuous bonus the click alone never bought.

---

## Part 2 — The non-obvious plays

### 2.1 If you're under ~50,000 followers: the cold-start boost is your engine

The code contains **exactly one deliberate "help the little guy" mechanism**, and it's generous — and in Oct it got **50× wider and much shorter**:

- Accounts with **≤ 50,000 followers** (was 1,000) whose post has **< 200 Home-Timeline views** (was 1,000; views elsewhere don't count) and is **≤ 2 hours old** (was 24h, now universal — no treatment-arm subtlety) and isn't a reply or retweet can get **one post per request lifted to slot 15** of the feed (the code targets ranks 15–16) — a near-top position that would otherwise be unreachable for a small account. (Eligibility constants: `ColdStartFollowerCap = 50000`, `ColdStartImpressionThreshold = 200`, `ColdStartMaxPostAgeSecs = 7200`, `ColdStartSlotMin/Max = 15/16`, `LowImpressionsMaxPositionRatio = 0.97` — [`author_cold_start.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/scorers/author_cold_start.rs).)
- The post must already rank within the **top 97%** of the scored pool (was 85%) — it's "lift the best eligible candidate," not "rescue the worst" — and the picker now **Thompson-samples by default** (Beta(0.75+favs, 49.25+views−favs), top-2): among near-zero-data posts the least-proven one gets a shot.
- The boost runs automatically. Your job is just to post **original** posts (replies and retweets are ineligible) **within 2 hours of your audience being online** — the window is now a launch boost, not a day-long one — and not waste the mechanism.

Three consequences worth knowing:

- **The 1,000-follower mark is gone as a cliff — the 50,000-follower mark replaces it, with a 2-hour fuse.** Cross 50K and you lose the boost; past 200 views or 2 hours and the post loses it even if you're eligible. The old empirical note (100–1K bucket 22.9% vs 1M+ 3.7%) predates the rewrite — expect the advantage to reach much larger accounts now, but only inside the 2-hour launch window.
- **Being unknown is judged as "average," not "bad."** Small accounts' learned impressions are diluted by hash collisions (their storage slot in the model is shared with many other accounts), so their posts are evaluated mostly on content, not on a deeply-learned reputation. The worst fate in this system is a *learned* negative reputation (Part 2.8), and small accounts are mostly spared it.
- **Small accounts' posts enter the discovery pool with zero favorites.** The retrieval index (the discovery system's searchable pool of posts) has a dedicated "long-tail" window (`SidTail`) for authors under 1,000 followers that requires **0** favorites to join (`tail_min_fav_count = 0` — [`phoenix-rankall/src/config/mod.rs`](https://github.com/xai-org/x-algorithm/blob/main/phoenix-rankall/src/config/mod.rs)) — while normal posts need at least 1 fav to enter their day-one pool.

### 2.2 The 48-hour clock and the video loophole

- Every post is only a candidate for **48 hours** (`MAX_POST_AGE`, [`config.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/config.rs)). The model's freshness buckets stretch to ~80 hours. Text content is *fresh-then-dead*.
- **Videos ≥ 10 seconds** are different. They generate the "video view" engagement signal (an engagement event counted for discovery — `MIN_VIDEO_DURATION_SECS = 10` in [`implicit_engagement_signals_query_hydrator.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/query_hydrators/implicit_engagement_signals_query_hydrator.rs)) and, crucially, they enter **"evergreen" retrieval pools that keep them eligible for up to 5 years** (`evergreen_video_1825day` — plus, since Sept, age-banded `1fav_video_2day/2to4day/4to14day/4to30day` slices for fav'd videos — [`phoenix-rankall/src/config/mod.rs`](https://github.com/xai-org/x-algorithm/blob/main/phoenix-rankall/src/config/mod.rs)). Even at the very start, a ≥10-second video has a head start: it enters its out-of-network pools **at creation with zero favorites**, while text and photos need at least 1 favorite to reach the main OON (out-of-network — for people who don't follow you) pool ([`phoenixRankAllCandidateProcessor.strato`](https://github.com/xai-org/x-algorithm/blob/main/phoenix-rankall-strato/columns/phoenix_rank_all/phoenixRankAllCandidateProcessor.strato)).
- **Native Articles get none of this.** X's long-form Articles are not a distinct type in the algorithm: no article feature, weight, filter, retrieval pool, or visibility rule exists (the model's per-post flags are reply/retweet/quote/media only — [`recsys.proto`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/crates/serving/xai-recsys-proto/proto/recsys.proto)). An Article is scored on its content — the title and cover feed the post's embedding (its content representation) — and dies on the same **48-hour clock as text**. Write articles if the format serves your content; don't expect format-based reach.

The move: your best content should exist as a **≥10-second video**, because the same idea as text dies in two days while as video it can circulate for years.

### 2.3 The first 24 hours have hard thresholds — and they're tiny

Engagement counts are hydrated into the model live (60-second cache — [`engagement_counts_hydrator.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/candidate_hydrators/engagement_counts_hydrator.rs)), and the retrieval index admits posts by tier ([`phoenix-rankall`](https://github.com/xai-org/x-algorithm/blob/main/phoenix-rankall/src/config/mod.rs)). Two different channels — don't conflate them:

- **Index membership (fav-only gates).** **1 favorite in a day** → the post enters the base out-of-network pools (`1fav_1day`). (Sept: fav processing now emits `1fav` unconditionally — the old 0-fav race that could miss the index is closed.) **32 favorites in a day** → the post *additionally* enters the "hot" pool (`32fav_1day`) — more discovery chances. **Only favorites count toward these two gates**: 20 favs + 100 replies does *not* open the hot pool — replies, reposts, quotes, and views never trigger index admission (the windows are built exclusively on favorite events; Oct: the same fav-only logic applies to the new `1fav_reply`/`32fav_reply` reply windows, and videos are the exception — they enter video windows at creation with 0 favs).
- **Prediction (every engagement counts).** Every fav, reply, repost, quote, and view feeds the model's live count features and raises per-viewer predicted engagement for later requests — the momentum loop. A 20-fav/100-reply post misses the hot pool but will typically out-serve a 32-fav/0-reply post, because reply carries weight 5.0 and the prediction channel decides who is actually shown the post.
- Zero-favorite posts exist only in the fresh-post window (and the small-account tail, Part 2.1; videos have their own creation-time windows).

So the concrete, code-derived goals for a post's first day are **1 fav (minimum, to become *eligible* for discovery — not a guarantee of being served) and 32 favs (to add the hot pool)**. **Indexed ≠ served**: pool membership only makes the post searchable by discovery; each non-follower still has to win a per-viewer top-50 against ~1,000 candidates with the 0.75 OON discount. Most eligible posts are never served OON. There is a real momentum loop here: high live counts → higher predicted engagement → served more → more counts. (Sept: the model now reads live counts through a dedicated log2-MLP embedding path too — velocity shapes predictions directly, not just via buckets.)

### 2.4 Quote, don't repost — the credit shift

This is one of the most counterintuitive findings in the entire code:

- A **repost is not your post**. The model scores the *original* post with the *original author's* embedding (home-mixer resolves a repost to the original tweet and author id before anything else — [`models/candidate.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/models/candidate.rs)). Engagement on your repost trains **the original author's** learned reputation, not yours — in both directions. A repost that flops is also the original author's negative signal. **Your author row (the learned reputation that gates your own posts' reach) is untouched.** You are a distribution agent, not a beneficiary. (Sept: on retweets the model's `AuthorInfo` carries no follow flags and no follower count — `is_following_user`/`followers` are `None`.)
- A **quote is yours**. You wrote text; the candidate author is you; quote engagement trains your row.

If you want engagement to improve *your* account state, quote with your own words. Repost only if your goal is to help the original author. (The one caveat: quoting gives attention to the original — quoted-post clicks are explicitly scored — and if the original is dropped for a viewer, your quote is dropped too.)

**Should you ever repost your *own* post — especially one that performed poorly?** Almost never — and the code explains why a self-repost is a second run of the same failure, not a second chance:

- **The model re-judges the original, not the repost.** A repost is resolved to the original tweet id before anything else ([`candidate.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/models/candidate.rs)), so the model sees the same post embedding, the same content, and the same author state — and it computes post-age from the *original's* creation time, not the repost's ([`util.rs`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/crates/common/xai-recsys/src/util.rs)). Reposting a 3-day-old flop re-scores a 3-day-old post: worst-case freshness features, the same low-propensity predictions. The 48-hour filter *does* pass the repost (it checks the repost's own id — [`age_filter.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/filters/age_filter.rs)), but the model's freshness never resets.
- **Anyone who already saw the original is filtered out.** The previously-seen filter checks the retweeted (original) id too ([`previously_seen_posts_filter.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/filters/previously_seen_posts_filter.rs)), and duplicate reposts of the same original collapse to one copy ([`retweet_deduplication_filter.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/filters/retweet_deduplication_filter.rs)). The repost reaches only people who missed it the first time.
- **The repost is deboosted — yours included, when the switch is on.** In-network reposts get the 0.75 out-of-network discount *if* `EnableOonRescoreForInNetworkRepliesRetweets` is enabled (Part 2.10). The system prices reposts below originals by design.
- **The expected outcome is the original outcome.** Same predictions → the same best-predicted audience is re-served → another round of near-zero outcomes, logged at full weight on your author row — the serving loop's second iteration, reinforcing the exact state that made the post flop. There is no second-chance mechanism: no freshness reset, no new embedding, no honeymoon.
- **The narrow exception:** if the original was essentially never served (posted at a dead hour, before your audience existed, during a cold-start), the flop carries no information — and the repost *does* re-enter the retrieval index as a fresh id for 48 hours, reaching new viewers and new followers who never saw it. That is a real but small effect (the model still scores the original with old-age features and the OON 0.75).
- **And even then, two strictly better moves exist in the code:** post the improved version as a **new original** (fresh id → fresh age features, unexplored credit, cold-start eligible if you're ≤50K followers with <200 views and ≤2h old, no deboost, no dedup collision), or **quote your own post** with added context (a quote is a new post with your own text — and quotes, unlike reposts, do *not* get the 0.75 in-network deboost, which applies only to replies and reposts when the rescore switch is on). Never repost your own flops mechanically or at volume — that pattern is the signature of RTBot, the bot-detection model's "mechanical reposting" classification (Part 6).

### 2.5 Replies: big threads vs small threads (the "blast-radius" gates)

Every reply is routed through gates keyed on the **follower count of the person you're replying to and the thread root** (thresholds: **250,000 followers** for spam-screening and reply-ranking — Oct split, was 120K; coordinated-spam at **180K**, was 1K; Gemma-spam routing floor 125K; `GROK_GEMMA_FOLLOWER_SPLIT` and `GEMMA_REPLY_SPAM_MIN_FOLLOWERS` in [`grox/flows/reply_spam/constants.py`](https://github.com/xai-org/x-algorithm/blob/main/grox/flows/reply_spam/constants.py) / [`task_filter.py`](https://github.com/xai-org/x-algorithm/blob/main/grox/flows/reply_spam/task_filter.py)):

| Thread target & root | Your reply gets |
|---|---|
| Both ≤ 250K followers | **LLM spam screening** (a large-language-model AI — a text-reading AI — checks your reply for spam; a false positive publishes a spam label on that reply) — and **no** reply ranking. Your reply is treated as *potentially suspicious*. |
| Either > 250K followers | **No spam screening** (your "reputation" substitutes), and an **LLM ranks your reply** — a good reply can win placement in front of a huge audience. (Reply scores now only move *down* — only a lower score overwrites, so the minimum so far sticks.) |
| Reply at depth ≥2 with same-author parent, root ≥1K, not high-PR/grey (NEW Oct multi-step flow) | **Gemma-2 thread scoring** — ≥2 same-author flags incl. newest → spam label + ranking demoted to **0.0**. |

There's a deeper problem with small-thread replying. The training data has **no baseline normalization** — no "this conversation is small, so low engagement is expected." Every "served and ignored" impression is a **full-weight zero example**, and with only a handful of viewers there's almost no viewer-side variance to blame. **The model's only explanation for "1 engagement, 4 nothings" is the author. Your row absorbs nearly the entire outcome.** At volume, a patient person replying daily into small dead-end threads builds the same statistical profile as a bot would — it has nothing to do with speed or spam behavior.

The strategic conclusions:
- If you reply **for reach or for your learned state**: prefer threads where the target or the root is **above 250K followers** — no screening, LLM-ranked placement, recap audiences of thousands (notification cards pushed to people who engaged with the thread), and proportionally many positive training examples (records of "shown and engaged" that teach the model). Avoid stacking same-author replies in one thread — the Oct multi-step flow demotes exactly that pattern to 0.0.
- If you reply **for relationship**: know that in the small-conversation lane you are being screened, and each reply is a low-value, author-attributed training example.
- Watch your patterns: the bot-behavior model (bdsm — an AI that watches account behavior patterns) has a dedicated **ReplySpamBot** classification — high-volume replying with low engagement is exactly the signature it recognizes (the enforcement note fires when the dominant bot head — one of its behavior classifications — scores > 0.5 over a policy-defined action count — Sept: the count itself is now redacted, `999999` in the shipped file — [`bdsm/runtime/score_results_sink_focal.py`](https://github.com/xai-org/x-algorithm/blob/main/bdsm/runtime/score_results_sink_focal.py)).

### 2.6 Likes: the biggest lever you have over your *own* feed — and a trap

Liking is never negative for your posts — a like is literally the **training target of the discovery model** (the behavior the discovery model is trained to predict; the retrieval model's `home` head is trained on favorites — [`xrecsys_two_tower.py`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/xrex/configs/xrecsys_two_tower.py)); it's the strongest positive signal in the system. But:

- **What you like steers what you're served.** Your recent actions (5-minute window, `UAS_WINDOW_TIME_MS = 300,000` — [`config.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/config.rs)) plus your likes' effect on the "similar content" discovery source mean your likes curate your feed in real time — and via retraining, durably. Liking bait feeds you more bait ("garbage in, garbage out"). There is no quality filter between you and this.
- **The like-and-unfollow trap.** If you like a post *and* then unfollow, block, mute, report, or mark "not interested" on it, the negative action **dominates** — in the discovery model (the one trained on likes), any of those hard-negative actions (expressions of dislike: block, mute, report, unfollow, "not interested") masks the like out entirely, so the like teaches nothing (`hard_negative_actions` in [`recsys_two_tower_model.py`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/xrex/models/recsys_two_tower_model.py)). (The ranking model has an analogous masking rule, but the released configs ship with it **off** — `mask_candidate_positive_when_negative_action_present = False` — so there the like still teaches and the negative adds its own evidence.) The most common real-world case: liking an account and then unfollowing it. The unfollow trains the model that this author predicts negative outcomes for you.
- **Negative feedback is a veto; likes are votes.** Ten "not interested" clicks outweigh one like in what the model believes about your propensity.
- **Liking mechanically** (rapid, patterned like-binges) is one of the bot signatures the enforcement model recognizes. How you like matters, not just what.

So: like deliberately, never like-then-unfollow, and treat your likes as feed-curation rather than social signaling.

### 2.7 Negative feedback: the invisible killer — and its limits

The negative weights are enormous: **report −234, mute −58.8, not-interested −47.52, block −31.2**. Because they multiply per-viewer *prediction probabilities*, even a ~17.5% predicted probability of "not interested" outweighs *everything* positive a post can earn — for that viewer. And there's a structural trap: any post that ends up net-negative is **compressed into a tiny floor band** (score range [0, 0.001)), where it loses to every net-positive post by a factor of a thousand. **A post is either competitive or dead; there is almost no "slightly punished" state.**

The limits (per the platform's own August 14 clarification, which the code now documents):

- Predictions are **personalized**: negative predictions primarily affect recommendations for *similar users*, not everyone's ranking of the post. A coordinated attack by strangers mostly poisons the rankings for users who resemble the attackers.
- **Only actions on posts served in the Home Timeline count.** Directly navigating to a post (e.g., via groupchat coordination) has no ranking impact.
- The weights scale probabilities, not counts — "one report cancels 468 likes" is explicitly called out as an incorrect interpretation (the report head is −234 because the baseline report probability is >1,000× lower than a like).

Practical conclusions: **don't fear brigades; fear your own audience.** The negative-feedback pattern that matters is the one your *actual* viewers produce — content that reliably generates "not interested" clicks, mutes, or reports (repetitive, annoying, bait-y output) drags the learned state that gates everything else. And driving traffic via direct links/groupchats does nothing for ranking.

### 2.8 Recovery: why it looks like nothing is happening, and what it actually takes

**The floor band means recovery is invisible until it isn't.** An account clawing its way from "catastrophically negative" to "barely negative" ranks exactly the same — dead last. Progress on the way up is unmeasurable from the outside, which is why trapped accounts report "nothing I do matters." The model's predictions must actually **cross the sign** before anything visibly changes. Expect that phase and don't judge per-post during it.

What the code says about the underlying mechanics:

- **Nothing is permanent.** Three hard limits prevent negative history from accumulating without limit: (1) *probability saturation* — once the model is certain (P ≈ 1), more negatives teach nothing; (2) *the rolling training window* — old events age out (length unpublished); (3) *the optimizer forgets* — the optimizer (the math that updates the learned reputation during training) has an explicit exponential decay on its accumulator (its running memory of past updates) (default `half_life_steps = 5000` — Sept: production configs set 2500 on H100 / 5000 on GB300, plus lazy decay and weight decay — [`phoenix/xrex/optimizers/recsys/rowwise_adagrad.py`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/xrex/optimizers/recsys/rowwise_adagrad.py)), so old gradient history doesn't lock an account in place. If this were plain AdaGrad, accumulated history would freeze busy accounts forever; the decay is the strongest evidence in the code that reputations are *not* designed to be permanent.
- **But there's no reset either.** No cooldown, no clean slate, no account-level escape clock exists. Recovery is re-learning: sustained positive outcomes, at enough volume to dominate the recent training window, persisted through the rollout lag (the delay before a newly trained model replaces the live one — a "shadow penalty" phase where you're doing the right things and nothing has changed yet).
- **The volume math is the crux.** A trapped account generating ~2 engagements per post against a history of millions of near-zero impressions can't out-train its own history at that rate. The lever that breaks the arithmetic: **replies in big conversations** — they're served to other people's audiences (so they get views regardless of your reputation), and they produce positive training examples on the *same* author row that gates your posts. It's the one side door out of the trap. (Cost to manage: "recap shown but not dwelled" — `CLIENT_TWEET_RECAP_NOT_DWELLED` — is a dedicated soft-negative signal (mildly negative — a lighter warning than a block or report), and your posts still face your own state.)
- **The 7-day credibility clock is separate.** The explicit credibility score (0–100, daily) re-computes from a **7-day engagement window** (`EngagementWindowDays = 7` — [`user-cred-v2/UserCredV2Config.scala`](https://github.com/xai-org/x-algorithm/blob/main/user-cred-v2/UserCredV2Config.scala)), so enforcement shields can be earned in roughly a week. But it gates *removal* (spam labels, suspensions, challenges), **not delivery** — it never feeds the feed.

### 2.9 The diversity step: stop competing with yourself

After scoring, a determinantal point process (DPP — the math name for the diversity selection; see the Glossary) picks the final ~50 using score + content similarity ([`vm-ranker/dpp.rs`](https://github.com/xai-org/x-algorithm/blob/main/vm-ranker/dpp.rs)). The rules are exact:

- The **highest-scoring post is always picked first**; every later pick must add *new* content. A near-duplicate of an already-chosen post is **zeroed entirely** — removed as thoroughly as a filter drop, regardless of how high its score is.
- **The feed's second copy of any story is always a distinct-content post, never a same-content post with a slightly different score.**
- Age and follower count are not inputs — the DPP is score + content only.
- A post missing a content embedding gets a random vector (effectively orthogonal to everything) and is never similarity-penalized.

Practical rules: **one strong post per story; don't post five variations of the same content.** Mix formats and topics — a prolific author posting *diverse* content is only hit by the (mild, per-request) author-diversity decay (0.5^k per post, floor 0.25); a single author posting five near-identical posts is hit by both. If you're covering a story everyone is covering, be the first or the most distinctive — duplicates of an already-picked version die.

### 2.10 Mutual follows: the +15 boost (originals only)

Original posts from authors who **follow you back** get reply weight 5 → **20** — the largest single positive multiplier in the system (`BidirectionalFollowReplyWeightBoost = 15.0` — [`param.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/param.rs); rollout history in [`docs/BIDIRECTIONAL_BOOST_CHANGE.md`](https://github.com/xai-org/x-algorithm/blob/main/docs/BIDIRECTIONAL_BOOST_CHANGE.md)). The follow relationship in both directions is also now a direct model input (`IS_AUTHOR_FOLLOWED_BY_VIEWER` / `IS_AUTHOR_FOLLOWING_VIEWER` — enabled in `home_direct_packed` since the Aug 19 commit — [`phoenix/crates/common/xai-recsys/src/util.rs`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/crates/common/xai-recsys/src/util.rs)), so mutual-follow status shapes the prediction itself, not just the post-hoc boost.

Two subtleties from the code:

- The boost applies **only to original posts — never to your replies or reposts**, and never to replies/retweets *of* mutual authors.
- Replies and reposts from accounts you *do* follow are explicitly **discounted** (the 0.75 out-of-network factor applies to them too — but only when `EnableOonRescoreForInNetworkRepliesRetweets` is on) — the system values originals over conversation participation. A reply must outscore an original by a third just to tie (when the switch is on).

So: pursue follow-backs from the accounts you follow, and direct your best *original* content at them — that's where the +15 lives.

### 2.11 The OON discount: in-network is structurally favored

Out-of-network posts get a **0.75 multiplier** (`OonWeightFactor`, 0.5 in topic feeds — [`param.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/param.rs)); new-account viewers get 0.00001 (if enabled) — nearly in-network-only feeds. Combined with the cold-start boost and the diversity rules, this means:

- Followers are the cheapest distribution you'll ever get — but they're individually predicted (Part 0, idea 2), so **an engaged small audience beats a massive passive one**. A 12K-follower senator can out-deliver a 12M one; that's the code working as designed.
- If you're a new account: follow people and interact — in-network content is what you'll be served.

**Your followers are the seed audience — by construction.** Follow status is the only thing that makes distribution *unconditionally offered*: your post enters every follower's candidate pool without passing any engagement or index gate (the discovery system's eligibility lists) — but **whether it's shown to a given follower is still a full per-viewer decision** (filters, per-viewer prediction, ranking, diversity, top-50 cut). The in-network channel (Thunder) is in-memory with a 48-hour window and no engagement threshold ([`thunder/`](https://github.com/xai-org/x-algorithm/blob/main/thunder/config.rs)). **Who actually sees your post first:** followers who (a) open the feed within 48 hours, (b) haven't already been shown it (seen/served dedup — a follower who saw it in an earlier request is excluded), (c) aren't filtered (muted keywords, blocks), and (d) have a per-viewer predicted-engagement score high enough to survive the top-50 cut against ~1,000 other candidates — in practice: **currently-active followers with a recent engagement history with you**. "Whoever happens to be online" is the pool; the prediction is the selector. Every out-of-network channel is gated instead:

- **The main discovery index (the searchable pool of out-of-network posts) admits posts only after ≥ 1 favorite — favorites only** — the home retrieval corpus in the training pipeline is literally the 1-fav pool (`1fav_1day` — [`retrieval_dataset.py`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/xrex/data/retrieval_dataset.py)); the 32-fav "hot" pool opens at 32 favs, again favorites only (Part 2.3). Replies, reposts, quotes, and views never open an index window.
- **The 0-fav OON paths are the fresh window, the video creation-time windows, and the small-account tail** — `post_creation_1day` arrives with a ~2-minute delay and no real engagement signals (a weak candidate that must also beat the 0.75 OON discount); videos ≥10s enter video windows at creation with 0 favs; authors under 1,000 followers get the `SidTail` pool at 0 favs. All three are candidacy only — each viewer still has to be won individually.
- **Between the gates, every engagement still expands who sees the post — through prediction, not pool membership.** The gates above are *index membership* (fav-only); the audience itself grows with every fav, reply, repost, quote, and view via the momentum loop (Part 2.3): the model's live engagement-count features (60-second freshness) raise the predicted engagement for every viewer, so the post wins more seats in later feed requests — including for followers who missed the first wave or were ranked out. Favorites never change the follower channel directly (it's candidate-from-creation, fav-independent), but they change how many of those followers — and non-followers — actually get shown the post. One timing note: favorite-driven index updates stop after roughly two days ([`favoriteEventProcessor.strato`](https://github.com/xai-org/x-algorithm/blob/main/phoenix-rankall-strato/columns/favoriteEventProcessor.strato)), so day-one engagement decides the *gates*; after that, only the prediction channel remains.

So the code never *restricts* a post to followers — but the typical lifecycle is: **followers see it first → engagement expands who sees it (each fav/reply/repost/quote/view improves the model's predictions for the next request — the momentum loop; 1 fav additionally opens the base OON pool, 32 favs the hot pool) → discovery expands — or doesn't, if the seed audience doesn't engage.** A post whose seed audience never engages it typically never leaves the seed: without the first favorite there is no base-pool membership (fresh/video/tail windows aside), and without per-viewer prediction wins there is no serving even with membership. **How big is the seed?** There's no fixed number — it's every follower feed request in the first 48 hours, winnowed by activity (who actually opens the feed), seen-dedup (followers already shown it are excluded — the pool shrinks as it converts), filters, and per-viewer ranking. Larger follower bases mean a larger raw pool and faster gate-crossings (more early favorites → sooner OON expansion) — that's the real size advantage; it never raises per-viewer delivery probability, which is the same machinery for everyone (Part 0, idea 2). Practical consequences: (1) your first-hour engagement is the *gate* to everything else — post when your engaged followers are online, not at audience-low hours; (2) the "my post flopped because my followers didn't see it" intuition is backwards — your followers are the one audience guaranteed to be *offered* it; what varies is the prediction each of them gets; (3) the >100%-of-followers accounts in the data (celestepoasts, atmoio) are what the seed unlocks when it converts: the OON tail is *earned* distribution, not luck.

### 2.12 Being quoted and quoted-adjacent rules

- Quoted-post clicks are explicitly scored (weight 0.05) — your quotes donate attention to the original. If you want attention, you want *to be quoted*.
- **Cascade drops**: if the original post is dropped for a viewer (safety label, legal, etc.), your quote/repost of it is dropped for that viewer too.
- **Brazil 2026 election filter** (added Aug 14, no feature switch — no configuration flag to disable it — [`home-mixer/filters/brazil_2026_election_filter.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/filters/brazil_2026_election_filter.rs)): posts, retweets, quotes, and reply-ancestors from **2,800** hardcoded user IDs (Oct; was 2,388/665) reported to Brazil's Electoral Court are dropped — unless the viewer follows the account. Jurisdiction-level drops exist and can change without any scoring change.

### 2.13 The 5-minute viewer window

The model's picture of *you as a viewer* is your last **5 minutes** of actions (`UAS_WINDOW_TIME_MS = 300,000`, capped at `MaxSeqLength = 1024` — [`config.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/config.rs)). What you do right now shapes what you're served right now — and, via retraining, what you'll be served later. As a viewer you have more immediate control than you think; as an author you have less than you'd like (your state moves on training cycles, not requests).

---

## Part 3 — The myths: what NOT to optimize

| Common belief | Reality (from the code) |
|---|---|
| Blue check / verification boosts reach | **Never read by the feed path** — the account fetch that feeds ranking pulls follower count, username, and NSFW flags (adult-content flags), nothing else ([`gizmoduck_hydrator.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/candidate_hydrators/gizmoduck_hydrator.rs)). No verification feature, weight, or filter exists in delivery. The only place verification appears is the credibility system's "trust mass" (see *PageRank / teleport mass* in the Glossary), which shields from *enforcement* (automated labels, challenges, suspensions) — a removal shield, not a boost. Don't pay for a blue check for reach. |
| Follower count matters | Never an input to scoring. Only the *relationship* (follow / mutual-follow flags) is a model feature. A large but passive list is worth less than a small engaged one. |
| Posting frequency affects reach | No frequency feature anywhere. Per-request diversity decay resets every time anyone opens the app; per-author caps are ceilings, not penalties. Frequency only matters to keep fresh posts in the 48-hour window. |
| Engagement rate explains reach | It's measured on the already-chosen audience (Part 0, idea 5). The model predicts things public totals don't include: per-viewer reply/share probability, predicted dwell, follow-through. |
| Profile completeness / pinned posts / highlights | Not read by the algorithm. |
| "Shadow-banning" (administrative) | The invisible-giant effect is *emergent*, not a flag: a learned low-propensity equilibrium. No rule drops these accounts — they still get thousands of views. The difference matters: a flag can be appealed; an equilibrium only changes through outcomes. |
| Coordinated attacks / brigades kill your reach | Platform's own clarification: predictions are personalized (attackers' reports mostly affect users *like the attackers*), and only Home-Timeline-served impressions count. Your own audience's genuine negative feedback is the real risk. |
| Native Articles / long-form posts get special reach treatment | **No.** No article flag exists anywhere in the feed path — no model feature (the per-post bools are only reply/retweet/quote/media — [`recsys.proto`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/crates/serving/xai-recsys-proto/proto/recsys.proto)), no weight, no filter, no retrieval pool, no visibility rule. Articles are scored as ordinary posts on their content (title/cover/body feed the post's embedding), obey the same 48-hour clock, and get no video-style evergreen pools. In the sample data, article posts over-delivered (25.95% vs 9.65% views/follower) — but that's 37 posts, 23 from one account, so treat it as cold-start + content quality, not a format boost (Part 2.2). |

One more myth about metrics: **profile clicks are worth exactly 0.0** (`ProfileClickWeight = 0.0` — [`param.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/param.rs)). Don't optimize for profile visits; optimize for follows.

---

## Part 4 — The two classes: how the algorithm sorts users, and the trap your own interactions can set

Most people assume reach is a continuum — post better, get more. The code suggests something sharper: **the system tends to sort accounts into two learned classes — a high-engagement class and a low-engagement class — and the sorting reinforces itself.** This is not a policy and not a shadow-ban; it is what happens when a model's predictions control the data the model is trained on.

**The loop, end to end.** Every account carries a learned "impression" (Part 0, idea 3). Low predicted engagement → few serves → few outcome examples → the impression keeps encoding low engagement (the serving loop). Low engagement also weakens the *discovery* representation, so the account stops being proposed to new viewers at all (the discovery loop). And because any post that goes net-negative is compressed into the floor band [0, 0.001) (Part 2.7), partial recovery is invisible — there is no "slightly less dead" state. The escape hatches are precisely bounded: the cold-start boost (Oct: ≤ 50K followers, <200 views, ≤2h, one slot per request, originals only — Part 2.1), the long-tail retrieval pool (≤ 1,000 followers — Part 2.1), the mutual-follow +15 (requires follow-backs — Part 2.10), and the day-one thresholds (Part 2.3). Notice what's missing: nothing between ~50K followers and "big enough that its audience crosses thresholds by itself."

**The 32-fav hot pool is de facto a size gate.** Reaching `32fav_1day` within the first day takes 0.003% of a 1M-follower account's audience but 0.6% of a 5K account's (Part 2.3). The first-hours race structurally favors large accounts and feeds a second momentum loop: live counts (60-second freshness) → higher predicted engagement → more serves → more counts.

**Two classes, not two sizes.** The split is by learned *propensity*, not follower count — Fetterman (538K) delivers 54% of followers while dawn_com (1.4M) delivers 0.07%. But size decides which levers exist:

- **Small (≤ ~1K):** the dilution shield (judged "average," not "bad" — Part 2.1) plus the two small-account boosts (cold-start now extends to 50K, but the long-tail pool stays ≤1K). The price is an average-ish ceiling until outcomes accumulate — dilution protects you from negatives but caps your upside.
- **Mid (~1K–100K):** partly covered now — the cold-start launch boost (≤50K, 2h/200 views) softens the band's lower half, but the dilution shield has faded (the embedding is now yours, warts and all), follow-backs from the engaged class are rare, and the hot pool favors bigger audiences. "Average" becoming "yours" is still the band with the fewest durable levers.
- **Large (100K+):** the embedding is fully personal — no dilution to soften a negative state (dawn_com's trap is *its own*) — but momentum, the hot pool, and follow-back networks generally keep the engaged half engaged.

**The inadvertent trap: your social neighborhood is training data.** Most users never touch the levers above on purpose — they interact with their peers, and that interaction pattern is itself the mechanism:

- **Replying inside small, quiet threads is the worst training example you can generate for yourself.** A reply there is served to a handful of people; the outcome is "1 engagement, 4 nothings"; and because the training data has no baseline normalization (no concept of "this conversation is small, so low engagement is expected") and almost no viewer-side variance to blame, **the model attributes the near-zero outcome to you** — your author row, the same row that gates your posts, absorbs it at full weight (Part 2.5).
- **Small-thread replies also put you under LLM spam screening** (the blast-radius gates — Part 2.5) and accumulate "recap shown but not dwelled" soft-negatives.
- **The mirror is symmetric.** Reply inside big threads (target/root > 250K followers) and your replies ride other people's audiences: thousands of impressions, proportionally many positive examples, no screening. The same account can nudge its own row in either direction purely by choosing *where* it replies.
- **Your likes and follows inside a small circle help the circle, not you.** Liking is a viewer-side signal (Part 2.6): it feeds the liked post's training, never your row. Generosity inside a small circle is fine socially — it is not self-improvement.
- **Clusters of small accounts can quietly drag each other down.** When everyone in a circle serves mostly to each other, everyone's outcomes are near-zero, and each row learns "this author produces nothing" from outcomes that were really a function of the audience (tiny, homogeneous, already-engaged elsewhere). The algorithm has no "small audience" excuse — only the outcome. This is the least-visible way the two classes reproduce themselves: not through any rule, but through the statistics of who talks to whom.

**What to do about it.** (1) Treat your replies as training data and choose their audience: high-blast-radius threads feed your row; dead-end threads tax it. (2) If you're ≤ 50K, keep feeding the cold-start launch mechanism (2h/200 views) and — if ≤1K — the long-tail pool — they are the only unconditional help in the system. (3) If you're mid-size, your levers are mutual follow-backs, big-thread replies, and the first-day thresholds — and your biggest risk is becoming a regular contributor to a small, quiet circle. (4) Expect the boundary to feel sticky: the floor band hides progress, and the training window + rollout cadence (both unpublished) set the real timescale.

---

## Part 5 — Other side effects that quietly hurt accounts

Beyond the two-class dynamic, the code supports several more ways users can be harmed — none of them named "penalties," all of them side effects of mechanisms doing something else. Roughly in order of how often they bite:

1. **The enforcement shield tracks size and verification, not behavior.** The credibility score (0–100, daily) is PageRank mass (influence that flows through the follow graph — each follower passes you a share of their mass) seeded with extra teleport mass (a baseline share of mass given to premium/verified accounts) ([`UserCredV2App.scala`](https://github.com/xai-org/x-algorithm/blob/main/user-cred-v2/UserCredV2App.scala)). Enforcement skips accounts with cred ≥ 50 or high-PR ≥ 54 (a "high PageRank" designation), and the no-score fallback is follower count > 25,000 ([`enforcement_user.yaml`](https://github.com/xai-org/x-algorithm/blob/main/abuse-enforcement-service/service-lib/rules/enforcement_user.yaml), [`IsHighPageRankUser.df`](https://github.com/xai-org/x-algorithm/blob/main/botmaker-rules/scarecrow/derived-feature/IsHighPageRankUser.df)). Net effect: the accounts most exposed to challenges, labels, and suspensions tend to be small non-verified ones — regardless of how they behave — while large/verified accounts carry the shield even when they abuse it. The mitigation is to stay label-clean, because the enforcement rules themselves still require a model-score trigger.

2. **Sub-10-second videos forfeit the entire video machinery.** Video views only count at ≥ 10 seconds (`MIN_VIDEO_DURATION_SECS = 10` — [`implicit_engagement_signals_query_hydrator.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/query_hydrators/implicit_engagement_signals_query_hydrator.rs)), and video-quality-view credit requires strictly **> 10 s** (`min_video_duration_ms > 10,000` — an exactly-10-second video clears the signal gate but not the VQV gate; viewers with ≥ 10K followers are exempt — [`candidates_util.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/util/candidates_util.rs)). **Sept: the VQV scoring weight is 0.0 — "watched most of it" no longer scores directly**, only via the implicit discovery signal. A 9-second video generates none of the video signals, earns no VQV, and — unlike a photo — can't even fall back on photo signals. Make it > 10 s or make it a photo.

3. **Live engagement counts saturate — in buckets.** The count features are bucketed and capped (favorites ≤ 18, replies ≤ 13, reposts ≤ 15, quotes ≤ 13, views ≤ 25 — [`recsys_model.py`](https://github.com/xai-org/x-algorithm/blob/main/phoenix/xrex/models/recsys_model.py)). A post with 20 favorites and one with 20,000 favorites present identically to the count feature; beyond the cap, momentum flows only through training outcomes. (**Sept**: `home_direct_packed` also feeds live counts through an uncapped log2-MLP embedding — velocity now has a second, finer-grained channel — but the bucket caps above still apply.) "It already has 10K likes" is not itself a boost.

4. **Your feed can be re-steered within ~300 ms of your own actions.** The discovery source consumes your recent explicit signals (favorites, reposts, bookmarks, replies, shares) with a 300 ms cache ([`explicit_engagement_signals_query_hydrator.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/query_hydrators/explicit_engagement_signals_query_hydrator.rs)), and the model's picture of you is a 5-minute rolling window ([`config.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/config.rs)). One impulsive like can shift what the similar-content source proposes for the rest of the session — an instant echo chamber with no quality filter (Part 2.6).

5. **The mutual-follow boost concentrates inside established networks.** The +15 applies only where follow-backs already exist (Part 2.10) — i.e., mostly inside the engaged class. Outsiders rarely win follow-backs from large accounts, so the largest single positive weight in the system compounds class separation rather than crossing it. (Sept: the mutual-follow signal is now three layers deep — +15 boost, direct model feature, enabled follow flags — the features landing in the Aug 19 commit.)

6. **The transparency tool excludes the accounts most at risk.** Under the Hood requires account age ≥ 365 days and ≥ 10 posts/month ([`underTheHoodReport.User.strato`](https://github.com/xai-org/x-algorithm/blob/main/under-the-hood/strato/columns/underTheHoodReport.User.strato)). New and low-volume accounts — exactly the ones most likely to carry a label they don't know about — cannot check theirs. If you're young or sparse, you're flying blind by design.

---

## Part 6 — Knowing your own state

- **Check your labels.** X's transparency tool (`under-the-hood` — [`under-the-hood/`](https://github.com/xai-org/x-algorithm/tree/main/under-the-hood)) shows the labels applied to your account and posts (eligibility: account 365+ days old, 10+ posts/month). Spam/abuse/NSFW labels drive visibility drops — worth knowing whether you carry any.
- **The credibility score** (0–100, daily): ≥ 50 exempts you from automated enforcement rules (`pagerank_skipped: cred.score >= 50.0` — [`enforcement_user.yaml`](https://github.com/xai-org/x-algorithm/blob/main/abuse-enforcement-service/service-lib/rules/enforcement_user.yaml)); ≥ 54 (high-PR) also skips real-time spam labeling, with > 25K followers as the fallback ([`IsHighPageRankUser.df`](https://github.com/xai-org/x-algorithm/blob/main/botmaker-rules/scarecrow/derived-feature/IsHighPageRankUser.df)). It gates removal, not delivery — but removal is the permanent kill, so it matters.
- **Label lifetimes**: enforcement labels expire at 30 days (`ttl_msec: 2592000000` — [`enforcement_user.yaml`](https://github.com/xai-org/x-algorithm/blob/main/abuse-enforcement-service/service-lib/rules/enforcement_user.yaml)), real-time spam labels at ~26 hours ([`scarecrow/legacy/SpamLabelStoreRegistry.scala`](https://github.com/xai-org/x-algorithm/blob/main/scarecrow/legacy/SpamLabelStoreRegistry.scala)), post→account label aggregation at 7 days ([`safety-label-user-agg`](https://github.com/xai-org/x-algorithm/blob/main/safety-label-user-agg/postToUserLabelRules.strato)). Suspensions are permanent until actioned. Labels are bounded; the learned reputation is not (Part 2.8).
- **The enforcement layer detects AI slop** (`llm_slop_user` / `llm_slop_post` rules → SpamHighRecall / RiskyHighVizReply labels — Sept: joined by a `panda_reports_embedding_v10_rough_spam → suspend` rule — [`enforcement_user.yaml`](https://github.com/xai-org/x-algorithm/blob/main/abuse-enforcement-service/service-lib/rules/enforcement_user.yaml) / [`enforcement_post.yaml`](https://github.com/xai-org/x-algorithm/blob/main/abuse-enforcement-service/service-lib/rules/enforcement_post.yaml)), plus bot patterns (LikeBot, RTBot, ReplySpamBot, FollowBot, EngagementAmplifier, MultiActionBot, TweetSpamBot — [`bdsm/runtime/heads.py`](https://github.com/xai-org/x-algorithm/blob/main/bdsm/runtime/heads.py); enforcement note fires when the dominant head scores > 0.5 over a policy-defined action count — Sept: the count is redacted, `999999` in the shipped file). These are removal risks, not reach factors — but they end accounts.
- **"Not administratively punished" is verifiable** (posts are served, no all-surface drop — no restriction applied to every viewer; such a drop would produce ~zero views). **"No negative history" is not** — blocks, mutes, reports, and out-of-network-only labels (restrictions that apply only to people who don't follow you) are private. Don't assume a quiet account is clean; don't assume a struggling one is dirty.

---

## Part 7 — The honest caveats

1. **Published defaults ≠ guaranteed production values.** Live weights are feature-switch driven (controlled by configuration flags) and read from an internal config file (`/usr/local/config/features/home-mixer/main/rust_home_mixer.yml` — [`config.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/config.rs)). The released values are synced from prod by cron, so they're likely close — but "likely" is doing work.
2. **Redacted pieces** (parts of the system withheld from the public release): the reranking service's value model (only the diversity step ships), bot-detection thresholds **including the bdsm action-count gate (redacted to `999999` in Sept)**, Grok LLM prompts, the training data window, and the checkpoint rollout cadence (how often newly trained versions of the model replace the live one). The last two are the actual answer to "how long does a reputation change take" — and they're unpublished.
3. **The data can't tell you about *you* specifically.** The mechanisms are mechanical; your account's future depends on hidden state only X can see.
4. **What you can verify**: an account's posts are being served (not suspended, no all-surface drop label — those produce ~zero views). What you can't: negative feedback history and OON-only restrictions, which look identical to "low score" in aggregate view counts.

---

## Glossary — every technical term, in plain English

Words below are defined as this guide uses them. Anything else that looks like code (`param.rs`, `1fav_1day`, `32fav_1day`, `TOP_K…`) is a name from the open-source release — treat it as a label, not something you need to understand. Terms are in alphabetical order.

- **All-surface drop** — a restriction applied to *every* viewer of a post (a suspension, or a post-level label like SPAM). It would produce near-zero views. By contrast, an out-of-network-only restriction looks identical to "just a low score" in view counts.
- **Author diversity decay** — the per-request reduction applied to an author's 2nd, 3rd, 4th… posts inside a single feed (each extra post is multiplied down toward a floor of 0.25). It resets every time anyone opens the app — it is not a lifetime posting limit.
- **Author row** (learned reputation, learned impression) — an account's stored track record in the model: a list of numbers encoding what has historically happened when this account's posts were shown to people. One row per account; everything the account posts is scored with it; it changes only through retraining (Part 2.8). See also *Embedding*.
- **bdsm** — the bot-behavior detection system: a model that watches account behavior patterns and classifies them into types (FollowBot, LikeBot, ReplySpamBot, RTBot, and more). It feeds enforcement, not feed ranking, and its operating thresholds are not published.
- **Blast-radius gates** — reply-processing rules keyed on how large an audience a conversation could reach (the follower counts of the person you reply to and the thread's root post). Small threads: your reply is AI-spam-screened and not ranked. Large threads: no screening; an AI ranks your reply instead (Part 2.5). Oct: the split sits at 250K (screen + rank), coordinated-spam at 180K, plus a multi-step same-author-reply flow.
- **Blue check / verification** — paid premium status. Never read by the feed algorithm; it only feeds the credibility score's teleport mass (below) — an enforcement shield, not a reach boost (Part 3).
- **Candidate / candidate pool** — a post being considered for a viewer's feed in one request. Each request assembles ~1,000 candidates and keeps ~35–47.
- **Checkpoint / rollout** — a saved snapshot of the model's learned state. Live predictions come from the latest checkpoint until a newly trained one replaces it; that schedule is not published.
- **Cold-start boost** — the one deliberate small-account favoritism in the code: for accounts with ≤ 50,000 followers (Oct; was 1,000), one eligible post per request (under 200 Home-Timeline views, ≤ 2 hours old, not a reply or retweet, top 97% of the pool) is lifted to slot 15–16 of the feed (Part 2.1). Thompson-samples by default.
- **Credibility score (user-cred)** — a daily 0–100 "trust" number per account, computed from PageRank-style math over the follow graph plus a 7-day window of favorites/reposts. It gates enforcement (labels, suspensions, challenges) — never feed delivery (Parts 2.8, 5, 6).
- **Dedup (seen/served filters)** — filters that remove posts the viewer was already shown, so a post's potential audience shrinks as it converts.
- **Discovery** — the out-of-network layer that finds posts for people who don't follow the author, by searching the index for content and authors similar to what the viewer engages with. See also *Retrieval*.
- **Diversity step (DPP)** — the post-scoring stage that stops a feed from filling with near-identical content: it always keeps the highest-scoring post, then picks the highest-scoring posts most *different* from the ones already chosen, and scores everything else to zero — removing it entirely (Part 2.9). "DPP" = "determinantal point process", the mathematical name of the selection algorithm.
- **Dwell / dwell time** — the seconds a viewer actually spends looking at a post. "Not dwelled" = shown but skipped. Three scoring terms: continuous dwell (0.006 per second, capped at ≈ 0.18 points total), **click-dwell (0.4, Oct — sustained viewing after a click)**, and a dwell-probability head (0.05).
- **Embedding** — a list of numbers representing something (an account, a post's content) to the model. Similarities between these numbers drive predictions, retrieval, and the diversity step. See also *Author row*, *SID*.
- **Enforcement** — automated account actions (spam labels, challenges, suspensions) triggered by model scores. The "sharp end" of reputation: it can end accounts, but it is not part of feed scoring (Parts 5, 6).
- **Engagement** — any measurable action on a post: like, reply, repost, quote, bookmark, share, follow, dwell, and more.
- **Engagement rate** — interactions divided by views, measured on the audience the system already chose to show the post to — a symptom of who was served, not a cause of reach (Part 0, idea 5).
- **Evergreen pools** — retrieval candidate lists that keep certain videos eligible for distribution for up to 5 years, versus ~48 hours for text and photos (Part 2.2). Since Sept, fav'd videos also get age-banded `1fav_video_*` slices (2/2-to-4/4-to-14/4-to-30 days).
- **Feature switch** — a configuration flag that turns a mechanism on or off. Production values live in an internal config file not shipped with the code (Part 7).
- **Feed path** — the whole pipeline that assembles and scores a feed: sources → filters → scoring → diversity → content gate.
- **Filter** — a pipeline step that removes posts from consideration: too old, already seen, blocked author, muted keyword, dropped by the content gate, and so on.
- **Floor band** — the score range [0, 0.001) where every post whose score goes net-negative lands: such posts rank below every net-positive post by a factor of a thousand, and degrees of negativity barely matter. A post is either competitive or dead (Part 2.7).
- **For You feed / Home Timeline** — X's main algorithmic timeline, assembled per request. "Home Timeline" is the same feed; only actions on posts *served there* count toward ranking (Part 2.7).
- **Freshness buckets** — the model's hourly age steps for a post (up to ~80 hours). Predicted engagement declines as a post ages; after 48 hours it stops being a candidate at all.
- **Hard-negative / soft-negative** — training-signal terms. A hard negative is an explicit dislike action (block, mute, report, unfollow, "not interested") — it can erase the positive signal of a like on the same post. A soft negative is mildly negative (e.g., a recap notification shown but not opened). Both teach the model what to avoid (Parts 2.6, 2.8).
- **Hash collision (dilution)** — the model stores accounts in shared slots keyed by hash codes; small accounts share their slot with many strangers, so their reputation reads as "average" rather than precisely theirs. Large accounts own their slot outright — their state is undiluted (Part 2.1).
- **Hot pool** — the retrieval pool a post enters after 32 favorites within a day (favorites only — 20 favs + 100 replies does not qualify) — more discovery chances (Part 2.3). Membership is eligibility, not serving.
- **Hydration** — fetching a candidate's live data (engagement counts, media info, author info) before scoring. "Hydrated into the model" = fetched and attached.
- **Index / retrieval index** — the stored, searchable pools of posts the discovery system searches for out-of-network content. Membership is gated by *favorite* engagement: 1 favorite for the base pool, 32 for the hot pool (replies/reposts/quotes/views never open a window; videos enter at creation) — and membership is only eligibility. Each viewer still has to be won individually (Parts 2.3, 2.11).
- **In-network / out-of-network (OON)** — in-network: posts from accounts the viewer follows. Out-of-network: everyone else. Out-of-network posts are scored at a 0.75 discount (0.5 in topic feeds) — in-network is structurally favored. In-network replies/reposts are discounted too, but only when a dedicated switch is on (Part 2.11).
- **Label** — a marker applied to a post or account (spam, NSFW, abusive, etc.) that triggers visibility rules: the post is dropped, shown behind a warning, or restricted. Labels expire on a schedule (enforcement labels 30 days, real-time spam labels ~26 hours, post→account aggregation 7 days) (Part 6).
- **LLM** — a large language model: an AI program that reads and writes text. Used here for spam screening, reply ranking, and content classification. ("Grok prompts" — the exact instructions given to X's classification AIs — are among the parts not published; Part 7.)
- **Momentum loop** — the self-reinforcing cycle where a post's engagement raises its predicted engagement, which gets it served to more people, which raises its engagement (Parts 2.3, 2.11).
- **Mutual follow** — two accounts that follow each other. Original posts from mutual follows get the +15 reply-weight boost (Part 2.10).
- **PageRank / teleport mass** — the math behind the credibility score. PageRank mass is influence that flows through the follow graph: every follower passes you a share of their mass. Teleport mass is the baseline share of mass the calculation hands to premium (verified) accounts. High mass = high credibility = enforcement shield (Part 5).
- **Phoenix / Thunder / SimClusters** — system names from the code: Phoenix is the family of main ranking and discovery models; Thunder is the in-memory store that serves posts from accounts you follow; SimClusters is a discovery source based on "people who engage with X also engage with Y" clusters.
- **Predicted probability** — the model's estimate, for each (viewer, post) pair, of how likely that viewer is to perform a given action (like, reply, share, report…) — a number between 0 and 1. Every weight multiplies a predicted probability, not a raw count (Part 1).
- **Premium** — accounts with blue/gray/gold/organization/affiliate verification.
- **Recap** — a notification card pushed to people who engaged with a thread ("X replied to a post you engaged with"). Replies reach large audiences this way; a recap shown but not opened is a soft-negative signal (Parts 2.5, 2.8).
- **Retraining / training window** — the periodic process of updating the model from new data (the only way any learned state changes), and the recent slice of history each retraining learns from (older events age out; the window's length is unpublished).
- **Retrieval** — the discovery layer that finds out-of-network candidates by searching the index. See also *Discovery*.
- **Serving loop** — the core trap: low predicted engagement → few serves → few engagement examples → the reputation keeps encoding "low engagement" → low predictions (Part 4).
- **SID (semantic ID)** — a compact content code per post, derived from its images/video/text; posts with similar content share code prefixes. The model's way of knowing what a post is about.
- **Slot** — one of the ~35–47 positions in a feed response. "Slot 15" is a near-top position (the cold-start boost's target).
- **Top-50 cut** — the final selection step that keeps the 50 highest-scoring candidates per request before the content gate.
- **Training example** — one record of "this viewer was shown this post; here is what they did (or didn't do)." The model learns from these. A "full-weight zero example" is a shown-and-ignored impression recorded at full strength, with no allowance for the conversation being small (Part 2.5).
- **UAS (user action sequence)** — the list of a viewer's recent actions fed to the model: a rolling 5-minute window, capped at 1,024 actions. The model's picture of you *right now* (Part 2.13).
- **Under the Hood** — X's transparency tool showing the labels applied to your account and posts (eligibility: account 365+ days old, 10+ posts/month) (Part 6).
- **VQV (video quality view)** — the "watched a meaningful portion of the video" signal. A video earns it only if it is longer than 10 seconds (strictly), and it is skipped for viewers with 10K+ followers. Since Sept its scoring weight is **0.0** — it survives only as an implicit discovery signal, not a score term (Part 5).
- **Weight** — the fixed number multiplying each predicted probability in the final score (reply 5.0, report −234). Weights are not counts: "reply is ~8× a like" means ~8× the *weight* at equal predicted probability — not "8 likes' worth of score" (Part 1).
- **Zero example** — see *Training example*.

---

## The one-page cheat sheet

**DO**
1. Design every post around one question: *will someone reply to this, or copy-paste the link to send it to someone?* (reply 5.0, copy-link share 20.0 — vs like 0.6; note the weights scale predicted probabilities, not counts — a DM share isn't "10 likes," Part 1). **Hold the click**: sustained post-click reading now pays 0.4 click-dwell.
2. Post **originals**, not replies/RTs — especially if you're small (cold-start slot-15 boost requires it; mutual-follow +15 requires it; replies/RTs are discounted). **Post them inside the 2-hour launch window** — the boost expires fast now.
3. Make your best content a **≥10-second video** (evergreen pools: 5 years of eligibility vs 48 hours for text — and Articles get no loophole: they're ordinary 48-hour posts, now with an unpublished-article VF drop)
4. Win the first day: **1 fav** makes the post *eligible* for discoverability (indexed, not guaranteed served); **32 favs in a day** adds the hot pool — and the first fav must come from your seed audience, because base-pool membership doesn't open until it does (Part 2.11). Replies and reposts help through predictions either way.. **New: fav'd replies get their own index** — good replies are now retrievable, not just ranked.
5. **Quote with your own words** instead of reposting (repost credit goes to the original author)
6. Reply into **big threads** (target/root > 250K followers): ranked, not spam-screened, bigger audiences, positive training signal. Don't stack same-author replies in one thread (Oct demotes that to 0.0).
7. **Like deliberately**; never like-then-unfollow; don't like bait (your likes train your own feed)
8. Pursue **follow-backs** from accounts you follow (+15 reply weight on your originals, plus a direct model feature)
9. **One strong post per story** — the diversity step zeroes near-duplicates
10. If you're stuck: sustained positive outcomes at volume, via the side door (replies in big conversations); expect a long invisible phase before anything visibly changes
11. Know your class and its levers: ≤50K → cold-start launch boost (2h/200 views); mid-size → follow-backs + big-thread replies; everyone → win the first day (Parts 2.1, 2.3, 4)

**DON'T**
1. Pay for verification for reach (zero delivery effect; it's only an enforcement shield)
2. Obsess over follower count, posting frequency, engagement rate, or profile completeness (none are scored)
3. Habitually reply into small dead-end threads (spam-screened + full-weight "no action" examples attributed to your author row)
4. Generate negative feedback from your *own* audience (report −234 / mute −58.8 / not-interested −47.52 / block −31.2 — and net-negative posts are compressed into a floor band)
5. Expect a big-account repost to save you (scored per viewer, carrying your original's reputation; engagement on it credits the original author)
6. Worry about brigades or drive direct-link traffic expecting ranking impact (per-viewer predictions; only Home-Timeline-served impressions count)
7. Post sub-10-second videos expecting video treatment (no video signal, no VQV score — make it ≥10 s or a photo — Part 5)
8. Assume the enforcement shield is earned — it tracks size and verification, so stay label-clean (Part 5)
9. Repost your own failed posts for a "second chance" (same post re-scored with the original's age, deboosted, seen-filtered — you re-run the failure; post a new original or quote yourself instead — Part 2.4)

---

---

## Sept 2026 update — what you'll actually notice

This section collects the Sept 2, 2026 code changes (commit `85ac72a`, 13 commits Aug 17–Sept 2) that you might feel as a user, in plain English. The rest of the guide above is already updated for them; this is the changelog. Nothing about the core playbook changes — replies and shares still rule — but four things got visibly stricter, broader, or faster.

**Most noticeable**

1. **The reply bar moved 4× higher.** The spam-screen vs reply-rank gates stepped from 30K to **120K followers** ([`task_filter.py`](https://github.com/xai-org/x-algorithm/blob/main/grox/flows/reply_spam/task_filter.py)). The gate checks two accounts — the person you're replying to and the thread root: if **both are at/below 120K** the reply is spam-screened and unranked; if **either is above 120K** it skips screening and is eligible for ranked placement. Threads in the 30K–120K band flipped lanes — what used to be "big-thread" treatment is now small-thread treatment. Reply scores also only move *down* now (only a lower score overwrites — the minimum so far sticks — and scores ≤ 1.0 are additionally fanned out to regional caches in parallel), so a bad low score is stickier. What to do: the old advice ("reply into big threads") still holds, but "big" now means **either the target or the root is above 120K** (Part 2.5, cheat sheet #6).
2. **The Following feed got stricter about mutes and blocks.** In the Following pipeline only (the For You mute filter still matches post text only): muted words now also match quoted-post and reply-ancestor text, and quote/repost authors who blocked you are now enforced. Gap to know: the new Following blocked-by check looks only at the quoted/retweeted author — a plain post whose direct author blocked you is not removed on this path. If you mute a word, expect fewer quote-threads slipping through in Following; if someone blocked you, expect to see less of their quoted/reposted content in Following.
3. **The Brazil election list grew 665 → 2,388 accounts** ([`brazil_2026_election_filter.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/filters/brazil_2026_election_filter.rs)). Same follower-exempt logic, much larger list: matching posts (author, repost, quote, or ancestor) are dropped for non-followers in the For You pipeline; followers of each listed account are exempt (Part 2.12).
4. **Spam and impersonation removal risk went up.** A new `panda_reports_embedding_v10_rough_spam → suspend` rule joined the enforcement rules, and the Scarecrow/BotMaker stack gained a human-review reporting path (`CreateXReviewReportProd/Staging`) — a new capability, though no shipped rule invokes it yet. The code-visible direction is one-way: more paths to suspension for spammy or impersonation-style accounts; nothing in the code changes normal posting behavior (Part 6).

**Moderately noticeable**

5. **Watch-time was repriced, slightly.** Video-open weight rose 0.05 → **0.07**, the "watched most of it" (VQV) weight went to **0.0**, and a small dwell-probability term (0.05) turned on ([`param.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/params/param.rs)). "Watched most of it" no longer scores directly — the 10-second gate survives only as a discovery signal — while dwell earns through two small terms. Net ranking impact is second-decimal; the practical advice is unchanged: make it > 10 s or make it a photo (Parts 1, 2.2, 5).
6. **Videos got a better mid-life.** The discovery index added age-banded video slices out to 30 days (Main 14 → 17 windows — net of a removed `evergreen_video_grok` pool — SID 10 → 15), and a 0-favorite indexing race that could miss the pool was closed (`1fav` is now emitted unconditionally, with a `1fav_uec_count_race` counter). Videos now climb a finer-grained maturity ladder; text is unchanged (Parts 2.2–2.3). (Both landed inside this window — the 0-fav fix in the Aug 17 commit, the video slices in the Aug 25 commit — not on Sept 2 itself.)
7. **Hot posts accelerate slightly faster.** `home_direct_packed` now reads live engagement counts through a dedicated finer-grained channel (a log2-MLP embedding in `recsys_feature_prep.py`) on top of the old buckets (fav ≤ 18, reply ≤ 13, repost ≤ 15, quote ≤ 13, view ≤ 25 — retained), and the mutual-follow signal is three layers deep (the +15 boost — unchanged — plus direct follow-direction model features, enabled since the Aug 19 commit). Winners still win the same way — just a touch faster (Parts 2.3, 2.10).
8. **Ads sit next to safer content.** The default ad blender is now risk-aware (`multi_risk`, brand-safety verdict V2 on): on the default path, high-risk posts never sit ad-adjacent (they fall back to plain feed slots). Other blenders remain selectable in code. Mostly advertiser-facing; you may notice saner ad placement.
9. **Safety labeling got slightly less trigger-happy.** Adult-content "Hard" verdicts in the "deluxe" flow now need two models to agree (or a judge model) — the standard flow only logs a disagreement metric, no downgrade — child-safety errors default to no-violation, and violent-media errors soften to a lesser label. Fewer false positives on normal content (Part 6).
10. **Cold-start counts Home-Timeline views only.** Non-Home views (profile, search) no longer burn your < 1,000-view eligibility — slightly more generous for small accounts (Part 2.1).

**Background (little or no visible change)**

- The experimental MPN scoring path was deleted in the Aug 25 commit — its `EnableMpnScoring` switch was replaced by an off-by-default `MultiplierPreOffset` flag (the `gated_dwell_regret` value-model gate predates this window; default mode is `weighted`) — the diversity step's zeroing behavior is unchanged.
- The reranker request now carries post IDs plus score (plus the retweeted ID, viewer ID, and DPP params — no follower counts, Phoenix head scores, head weights, slate context, or timestamps) — follower count reaches ranking through fewer paths than ever.
- The 16 NSFW-gating countries are identical but now runtime-configurable without a deploy — future coverage changes need no code release.
- The embedding optimizer's fused update gained explicit forgetting plus shrinkage (lazy decay + weight decay — previously it raised `ValueError`; H100 half-life 2500, GB300 5000), alongside a new Muon dense optimizer, and posts older than ~14 days now present zeroed engagement counts — supports faster recovery over time, nothing immediate.
- A new model-requested enforcement path exists but is switched off until X configures it — no effect today; the bot-detection action-count gate is now redacted from the public file (transparency-only change).

---

## Oct 2026 update — what you'll actually notice

This section collects the Oct 13, 2026 code changes (commit `e1a0375`, 28 commits Sept 3–Oct 13) that you might feel as a user, in plain English. The rest of the guide above is already updated for them; this is the changelog. Nothing about the core playbook changes — replies and shares still rule — but the small-account boost got much wider and shorter, the reply bar moved again, and the content gate gained new withholding states.

**Most noticeable**

1. **The cold-start boost got 50× wider and 12× shorter.** Eligibility moved from ≤1,000 followers / <1,000 views / 24h to **≤50,000 followers / <200 Home-Timeline views / ≤2 hours** (top 97% of the pool, Thompson-sampling on). If you have 1K–50K followers, you now get a launch boost you never had — but only for 2 hours and 200 views, so post when your audience is online. Past 50K, 200 views, or 2 hours, nothing changes (Part 2.1, cheat sheet #11).
2. **The reply bar moved again, 120K → 250K.** Spam-screening and reply-ranking now split at **`GROK_GEMMA_FOLLOWER_SPLIT=250K`** (coordinated-spam at 180K, Gemma-spam floor 125K). Threads in the 120K–250K band flipped lanes back to small-thread treatment. "Big" now means **either the target or the root is above 250K** (Part 2.5, cheat sheet #6). New: stacking same-author replies in one thread (depth ≥2, same-author parent, root ≥1K) goes to a thread scorer that demotes to **0.0** on ≥2 flags — don't dogpile your own thread.
3. **Watch-time was repriced again.** Like 0.5→**0.6**, video-open 0.07→**0.02**, click 0.4→**0.3**, and a big new **click-dwell term at 0.4** — holding attention after a click now pays. "Not interested" tightened −43.2→**−47.52**. Net: likes matter a touch more, clicks/video-opens less, sustained reading more (Part 1).
4. **The Brazil election list grew 2,388 → 2,800 accounts** ([`brazil_2026_election_filter.rs`](https://github.com/xai-org/x-algorithm/blob/main/home-mixer/filters/brazil_2026_election_filter.rs)). Same follower-exempt logic (Part 2.12).
5. **Good replies are now retrievable.** The discovery index added **`1fav_reply` / `32fav_reply`** windows — fav'd non-self replies get their own pools, not just ranked placement. A reply that earns favorites can now resurface via discovery (Part 2.3).
6. **Unpublished Articles now drop.** Draft/soft-deleted/unpublished Articles are withheld on the hydrated path (the first article-specific visibility rule) — publish before you promote (Parts 2.2, 3).

**Moderately noticeable**

7. **A popular-posts tap turned on.** A new **`PopularPostsSource`** (on by default, 500 results) feeds most-followed-active posts into every For You request — more big-account competition per request. A new SID-similarity source exists but ships off.
8. **Safety got tombstones and age-prompt blurs.** Withheld content can now be a **tombstone** (age-verification / app-update / local-regulations) or a blur with an **age-verification prompt** (selfie/persona), plus **limited-engagement** prompts — states between "shown" and "dropped" you may see as a viewer (Part 6).
9. **The transparency tool covers legal withholds.** Under-the-Hood reports now include law-driven visibility limits (which country), via the new `jetfuel/` page (Sept 18 + Oct).

**Background (little or no visible change)**

- The scorer moved from `ranking_scorer.rs` to a shared `xai-value-model/` crate consumed by the reranker (which can now optionally compute the value model itself — ships off); the dwell-regret experimental mode was deleted — the diversity step's zeroing behavior is unchanged.
- The retrieval model gained a third `ld` (recap-dwelled) head alongside home/immersive, SID snapshots moved v5→v8, and training gained a tiny reply ranker — discovery heads, not user levers.
- Three enforcement guardrails (`action_control`, `off_switch`, `overturn_hold` vs a new ledger service) all ship default-off — no effect today; the only rule change is a mock follower-count skip with no prod effect.

---

*Sources: [`algo-analysis.md`](algo-analysis.md) and [`account-analysis.md`](account-analysis.md), reflecting the Aug 13–14, 2026 open-source releases **through the Oct 13, 2026 state** of the X For You recommendation algorithm ([`xai-org/x-algorithm`](https://github.com/xai-org/x-algorithm), commits `a389166` and `c65aa17` through `e1a0375`). All inline file links point at the current `main` branch of that repository.*

<!-- REVIEW-BASELINE-FOOTER rev=e1a0375a903899984545c663bb0bccb3cdcdf046 short=e1a0375 date=2026-10-13T08:05:54Z -->
