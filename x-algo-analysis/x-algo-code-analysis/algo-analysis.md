# X Algorithm Analysis: Post Reach and Visibility

<!-- REVIEW-BASELINE repo=https://github.com/xai-org/x-algorithm.git rev=85ac72a1bba41f21615e3f0bca56da75970a6633 short=85ac72a date=2026-09-02T19:46:48Z prev=c65aa179db7bdd61e2c2821eac87f208a105c053 range_commits=13 -->

This document tracks research into the X "For You" feed algorithm to understand how post reach and visibility are determined, and what factors influence these metrics. Current as of the **Sept 2, 2026** tree (commit `85ac72a`; the Aug 13–14 history is preserved in the addenda below).

## Key Questions to Answer

1. **How does the algorithm determine reach or visibility for a post? What factors increase or diminish reach?**
2. **What account attributes affect reach of a post?**
3. **How truthful is the README documentation?** → See Section 11
4. **How does account reputation factor into reach, and how is reputation determined?** → See Section 12

> **Related documents:** this is the primary, code-citing technical reference. [account-analysis.md](account-analysis.md) is the technical addendum that answers scenario-specific questions against this code, and [user-guide.md](user-guide.md) is the non-technical plain-English playbook distilled from both.

> **DOC ACCURACY NOTE (August 2026 revision):** This document was rewritten for the **Aug 13, 2026 release** (commit `a389166`, +2053 files / +363K lines). The previous (July 2026) revision is archived below. The Aug release is a fundamentally different codebase:
>
> - **The scoring weights are no longer hidden.** All weight values (favorite 0.5, reply 5.0, report −234.0, …), `MAX_POST_AGE = 48h`, `TOP_K_CANDIDATES_TO_SELECT = 50`, `OonWeightFactor = 0.75`, `AuthorDiversityDecay = 0.5`, and `UAS_WINDOW_TIME_MS = 5 min` are now **open in code** (`home-mixer/params/param.rs`, `home-mixer/params/config.rs`). Section 5's old "weight configuration is missing" finding is obsolete.
> - **`visibility-filtering/` (the VF service) is now open source** — every drop rule, the ALLOW/INTERSTITIAL/DROP verdict model, and the user/tweet safety labels are visible (Sections 3, 11, 12.5).
> - **`vm-ranker/` is now open source**, but the proprietary value-model inference inside it was **redacted**: the released server only runs a DPP (determinantal point process) diversity filter over multimodal embeddings. The head-weight formula now lives client-side in home-mixer (Section 5.3, 12.5).
> - **`phoenix/` now ships the production training code** (`phoenix/xrex/`), real model configs (8 layers, emb 2560, history 1022, candidate 64, SID 6×256), and synthetic-data training; the old artifacts zip is gone — **no pretrained checkpoint is included** (Section 5.1, 10).
> - **Account reputation is no longer "absent from the codebase"**: `user-cred-v2/` (PageRank credibility), `abuse-enforcement-service/`, `scarecrow/` (BotMaker + spam-label store), `bdsm/`, `agatha/`, and user-level safety-label aggregation are all released (Section 12).
> - The old doc's July claims (dead-code `weighted_scorer`/`oon_scorer`/`author_diversity_scorer`, VMRanker "omission", "no reputation score anywhere") are superseded. The old release's scoring chain (RankingScorer + VMRanker override) survives, but with new heads, modes, and values.

> **AUG 14, 2026 ADDENDUM (commit `c65aa17`):** A follow-up commit the next day (21 files, +2,271/−32 lines) added: **(1)** official comments in `param.rs`/`ranking_scorer.rs`/README clarifying that weights scale *predicted probabilities, not raw counts* ("one report cancels 468 likes" is called out as an incorrect interpretation; the report head is −234 because the baseline report probability is >1,000× lower than a like), and that mass block/report campaigns mostly affect *similar* users because recommendations are personalized and only Home-Timeline-served impressions count; **(2)** `Brazil2026ElectionFilter` — a new pre-scoring filter (no feature switch) dropping posts from 665 hardcoded user IDs reported to Brazil's Electoral Court (plus their retweets/quotes/thread ancestors), unless the viewer follows the account (Section 3, 9); **(3)** three new serving-side model bool features — `IS_AUTHOR_FOLLOWED_BY_VIEWER`, `IS_AUTHOR_FOLLOWING_VIEWER` (both follow directions, candidate + history tokens) and `IS_STALE_POST14D` (zeroes engagement counts for posts > 14 days old when `enable_stale_post`, ships off); their indices live in the internal `FEATURE_CONFIG_PY` config, not the shipped `feature_config.py` (Section 5.1, 12.6); **(4)** grox blast-radius thresholds raised 15,000 → **30,000** (Section 9, 12.3); **(5)** a cold-start **Thompson-sampling** variant (off by default) (Section 5.3); **(6)** a new `video_14day` SID retrieval window (now 10) (Section 5.4); **(7)** bdsm enforcement-note structure now visible (dominant bot head > 0.5, ≥ 30 actions; templates redacted) (Section 12.4). None of this overturns the analysis; Section 1.B, 3, 5.1, 5.3, 5.4, 9, 12.3, 12.6 and 13.3 carry the details.

> **SEPT 2, 2026 ADDENDUM (commit `85ac72a`, 13 commits Aug 17–Sept 2):** scoring weights moved (`video_open` 0.05→**0.07**, `vqv` 0.05→**0.0**, dwell probability head 0.0→**0.05**); **MPN scoring removed** (replaced by a narrower `MultiplierPreOffset` gate, default off); the VMRanker request was stripped to ids + score (no follower counts, head weights, or slate context); cold-start eligibility now counts **Home-Timeline views** and the Thompson variant keeps **top-2**; `FOLLOWING_MAX_RESULT_SIZE` 100→**110**; UAS aggregation is now `DENSE_WITH_LONG_DWELL` for scoring and retrieval; the default ads blender is **`multi_risk`** with brand-safety verdict V2 on; grox blast-radius gates stepped 30K→**120K**; the Brazil list grew 665→**2,388** IDs (README notes an Aug 27 update); rankall went Main 14→**17** / SID 10→**15** windows (age-banded `1fav_video_*`, grok pool removed, min-age gating with prefloor sidecars) and the training EVERGREEN corpus moved to `video_4to14day`; model training moved to a **Muon** dense optimizer (emb lr 0.28, half-lives 2500/5000) with engagement-count MLP plus follow-direction and stale-post features **enabled** in `home_direct_packed` (bool indices now shipped); the VF `rules/` rewrite is behavior-preserving (same 28+26 rules, pinned by a golden corpus) with a now-dynamic NSFW country list; enforcement gained a `panda_reports_embedding_v10_rough_spam → suspend` rule plus a GrowthBook-gated generic-action path; the bdsm action-count gate is now **redacted** (`999999`, was 30). Sections 1.A–B, 1.G, 2, 3, 5.1, 5.3–5.4, 6, 9–13 carry the details. None of this overturns the analysis.

> **ARCHIVED DOC ACCURACY NOTE (July 2026 revision, prior release):** Earlier sections described `weighted_scorer.rs`, `oon_scorer.rs`, and `author_diversity_scorer.rs` as the active scoring pipeline. These files were dead code in that release (removed from `home-mixer/scorers/mod.rs`). The active scorer chain was `PhoenixScorer → RankingScorer → VMRanker`. `RankingScorer` merged weighted scoring + author diversity + OON weighting into one scorer, and `VMRanker` (a proprietary gRPC value-model ranker) ran after it and could **override** the final score entirely. In the Aug 2026 release, `weighted_scorer.rs`, `oon_scorer.rs`, and `author_diversity_scorer.rs` are **deleted outright**, `vm-ranker/` is open (but head-model-redacted), and RankingScorer gained new scoring modes.

---

## What's New in the Aug 13, 2026 Release (Executive Summary)

| New system | What it is | Relevance to reach |
|---|---|---|
| **`home-mixer/params/` (open)** | All weight values + structural constants in code | Removes the #1 limitation of prior analysis (exact weights known) |
| **`visibility-filtering/`** | The post-ranking content gate: ALLOW / INTERSTITIAL / DROP | Full drop-rule catalogue now auditable (Section 12.5) |
| **`vm-ranker/`** | Reranking service (DPP diversity only; head-model redacted) | DPP can zero-out candidates; diversity layer in final ranking (Section 5.3) |
| **`user-cred-v2/`** | Daily PageRank "credibility mass" over the follow+engagement graph | Reputation score now exists; gates enforcement & spam treatment (Section 12) |
| **`abuse-enforcement-service/`** | Rule-based enforcement (suspend/label/challenge) from model scores | Account-level actions with 500K/day caps (Section 12.4) |
| **`scarecrow/` + `botmaker/`** | BotMaker rule engine; spam-label store (TTL 26h), rate limits | Real-time labeling of posts (SPAM etc.) (Section 12.5) |
| **`phoenix/` (production code)** | Training code, real configs, synthetic data; **no checkpoint** | Model architecture now fully known (Section 5.1) |
| **`phoenix-rankall/`** | Kafka→parquet retrieval-index maintenance (engagement-threshold windows) | Which posts enter candidate pools (Section 5.4) |
| **`simclusters/` source in home-mixer** | 3rd OON source (cluster similarity, 20M users/145K clusters) | Additional OON distribution layer (Section 1.D) |
| **SID (semantic IDs)** | 6×256 residual-quantized codes from multimodal embeddings | Post representation for retrieval & ranking (Section 5.1) |
| **Scoring: new heads/modes** | `video_open`, `open_link`, `quoted_vqv`, `post_unexplored`, `not_dwelled`, `dwell_time` (dwell prob head active at 0.05 since Sept), dwell-regret mode, `MultiplierPreOffset` gate (MPN mode removed Sept), author cold-start boost, bidirectional-follow boost | New explicit levers (Sections 1, 5.3) |
| **`under-the-hood/`** | Transparency report (labels per account/post, 365d age + 10 posts/mo eligibility) | External accountability surface |

**Aug 14, 2026 addendum (commit `c65aa17`):** official weights-interpretation comments (weights × predicted probabilities, not counts) · `Brazil2026ElectionFilter` (665 hardcoded IDs, follower-exempt, no feature switch) · 3 new serving-side bool features (both follow directions + 14-day stale-post flag) · grox blast-radius thresholds 15K → 30K · cold-start Thompson-sampling variant (off) · SID `video_14day` window · bdsm enforcement-note structure. See Section 1.B, 3, 5.1, 5.3, 5.4, 9, 11, 12.3.

**Sept 2, 2026 addendum (commit `85ac72a`, 13 commits Aug 17–Sept 2):** weight moves (video_open 0.07, vqv 0.0, dwell prob 0.05) · MPN removed (`MultiplierPreOffset`, off) · VMRanker request slimmed to ids + score · grox gates 30K → 120K · Brazil list 665 → 2,388 IDs · rankall Main 17 / SID 15 windows · Muon training + stale-post/follow/count features enabled in `home_direct_packed` · VF rules refactor (behavior-preserving) with dynamic NSFW country list · new panda-reports suspend rule + generic-action path · bdsm count gate redacted. See the Sept addendum above and Sections 1.A–B, 1.G, 2, 3, 5.1, 5.3–5.4, 6, 9–13.

---

## System Architecture Overview

The X For You feed algorithm is a recommendation system combining:

1. **In-Network Content (Thunder)**: posts from accounts the viewer follows, held in-memory with 48h retention
2. **Out-of-Network Content**: **Phoenix retrieval** (two-tower transformer, SID-based) + **SimClusters** (cluster-similarity ANN) + **PhoenixTopics/PhoenixMoE/CachedPosts**
3. **Ranking (Phoenix transformer)**: predicts per-action probabilities, blended by `RankingScorer` weights, then DPP-diversified by `VMRanker`
4. **Visibility Filtering (VF)**: a separate service that decides ALLOW / INTERSTITIAL / DROP per post × viewer (now open source)

**File Location**: `README.md` — system overview (Aug 13, 2026); see Section 11 for the truthfulness audit.

Two pipelines compose the feed:
- **Post Pipeline** (`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`): find → hydrate → filter → score → select → VF.
- **Blending Pipeline** (`for_you_candidate_pipeline.rs`): adds ads, Who to Follow, prompts, push-to-home at fixed positions (`WHO_TO_FOLLOW_POSITION = 6`, `PROMPTS_POSITION = 0`), then `BlenderSelector` blends ads — default **`multi_risk`** (Sept; safe/medium/high-risk partitioning, BSR-aware slotting, high-risk posts never ad-adjacent; brand-safety verdict V2 on) — and `AdAdjacentServedFilter` re-sorts for ad adjacency (never drops organic posts; drops the ad if no swap candidate).

**Servers** (home-mixer/lib.rs): `ForYouFeedService`, `FollowingFeedService`, `RankedFollowingFeedService`, `ScoredPostsService`, `PhoenixScoresService` (+ reverse-chron following pipeline).

---

## 1. Post Reach and Visibility Determination

### A. Scoring Pipeline (current)

**Location**: `home-mixer/scorers/ranking_scorer.rs` (1572 lines) + `scorers/phoenix_scorer.rs` + `scorers/vm_ranker.rs`

**Scorer chain** (phoenix_candidate_pipeline.rs:386-407): `PhoenixScorer → RankingScorer → VMRanker` → TopKScoreSelector (50).

**1. PhoenixScorer** — calls the Phoenix gRPC ranker (`PredictNextActionsRequest`, up to `PHOENIX_CLIENT_MAX_CANDIDATES = 2800` — phoenix_request.rs:11), returns logits for **60 action heads + 8 continuous heads** (see Section 5.1). Scores are applied to the **original** tweet id (`candidate_scores(&get_original_tweet_id())`, phoenix_scorer.rs:109; Sept: backbone scores + slate context stamped the same way).

**2. RankingScorer** — `compute_weighted_parts` (ranking_scorer.rs:431-523):

```rust
terms = [
  favorite×0.5, reply×reply_weight_for(c), retweet×1.0,
  photo_expand×0.05, video_open×0.07, click×0.4, open_link×0.2,
  profile_click×0.0, vqv×vqv_weight, share×2.0, share_via_dm×5.0,
  share_via_copy_link×20.0, dwell×dwell_weight_for(c), quote×5.0,
  quoted_click×0.05, quoted_vqv×quoted_vqv_weight (0.0; duration-checkable), dwell×dwell_weight_for(c) (**0.05** + mutual-follow originals), dwell_time×0.004,
  click_dwell_time×0.0, active_secs_5m_residual×0.0, follow_author×4.0,
  post_unexplored×0.02 (additive, in-network only) | or dwell_time×(1+post_unexplored×α) (multiplicative mode, off),
  not_interested×−43.2, block_author×−31.2, mute_author×−58.8,
  report×−234.0, not_dwelled×−0.02,
]
pos = Σ positive terms, neg = −Σ negative terms
offset_score(pos−neg):  combined < 0 → (combined + negative_sum)/total_sum × 0.001
                         else       → combined + 0.001      (NEGATIVE_SCORES_OFFSET)
```

**3. Post-scoring adjustments** (ranking_scorer.rs:703-873):
- **Author diversity**: `multiplier = (1−0.25)·0.5^k + 0.25` for the k-th post of an author in the pool (decay 0.5, floor 0.25, enabled).
- **In-network reply/retweet deboost is gated, not always-on.** Replies and reposts from accounts the viewer follows get the 0.75 OON factor **only when `EnableOonRescoreForInNetworkRepliesRetweets` is on** (ranking_scorer.rs:677-687) — the README's blanket "replies and reposts from accounts the viewer does follow are discounted" overstates it. Still unknown whether prod enables it.
- **New-user OON factor**: viewers with account age < `NewUserAgeThresholdSecs` **and** following ≥ `NEW_USER_MIN_FOLLOWING = 5` get `NEW_USER_OON_WEIGHT_FACTOR = 0.00001` (vs 0.75) — a **75,000× stronger in-network bias**. Default `NewUserAgeThresholdSecs = 0` (off unless feature-switched).
- **`MultiplierPreOffset`** (`false` by default; replaced the removed **MPN scoring** in the Aug 25 commit): when on, applies diversity×OON multipliers **only to the positive net** pre-offset (`m·net` if net ≥ 0, else net), then cold-start; the default path runs cold-start → diversity → OON (ranking_scorer.rs).
- **Author cold-start boost** (`author_cold_start.rs`): one eligible post is lifted to slot 15 (see Section 5.3).

**4. VMRanker** (vm_ranker.rs, 118 lines — the fold/author-diversity override path was deleted in the Aug 25 commit) — sends `RankRequest` (viewer id, per-candidate tweet id + retweeted id + score, hardcoded `value_model_id="dpp"`, DPP params `theta=0.65, max_selected_rank=150` — follower counts, head weights, slate context, and author/follower fields are no longer sent) to the `vm-ranker/` service; overrides scores with the response map (fallback: `c.score`). A candidate DPP didn't select still scores **0.0** — i.e., DPP acts as a hard diversity cap. (Plus observability plumbing: `RerankerHeadTag=0`, backbone scores, served slate context with SID/recon fields.)

### B. Weights — NOW KNOWN (home-mixer/params/param.rs:314-480; Sept 2026 values)

**Sept 2026 moves (Aug 25 commit):** `video_open` 0.05→**0.07**, `vqv` 0.05→**0.0** (the VQV head contributes nothing), dwell **probability** head 0.0→**0.05** (now active; continuous `cont_dwell_time` unchanged at 0.004). Everything else below is unchanged.

| Weight | Default | Weight | Default |
|---|---|---|---|
| favorite | **0.5** | not_interested | **−43.2** |
| reply | **5.0** (+**15.0** bidirectional boost) | block_author | **−31.2** |
| retweet | **1.0** | mute_author | **−58.8** |
| photo_expand | **0.05** | report | **−234.0** |
| video_open | **0.07** | not_dwelled | **−0.02** |
| click | **0.4** | follow_author | **4.0** |
| open_link | **0.2** | post_unexplored | **0.02** |
| profile_click | **0.0** | cont_dwell_time | **0.004** |
| vqv | **0.0** | cont_click_dwell_time | **0.0** |
| share | **2.0** | active_secs_5m_residual | **0.0** |
| share_via_dm | **5.0** | dwell (prob head) | **0.05** |
| share_via_copy_link | **20.0** | quote | **5.0** |
| quoted_click | **0.05** | quoted_vqv | **0.0** |

**Interpretation**: the strongest positive levers are **shares via copy link (20)**, **reply (5, +15 if mutual-follow original)**, **quote/share-via-DM (5)**, **follow-author (4)**, **share (2)**. Likes are cheap (0.5). The dominant negatives are **report (−234)**, **mute (−58.8)**, **not-interested (−43.2)**, **block (−31.2)**. **Note**: `profile_click`, `click_dwell_time`, `active_secs_5m_residual`, `quoted_vqv`, and `vqv` are zero in defaults (Sept: VQV was zeroed, dwell prob activated). Continuous **dwell_time** (0.004) is sigmoid-activated and de-normalized by norm-scale 30 (recsys_model.py:3162-3176), so predicted dwell is **capped at ~30 s → max contribution ≈ 0.12** — less than a quarter of a like. Dwell is a real signal but a small one; the dwell probability head is now **0.05** (Sept).

**Aug 14 official clarification (param.rs:279-307, ranking_scorer.rs:418-446, README):** the weights scale **predicted probabilities** (or predicted continuous values), **not raw engagement counts** — so weight ratios are *not* count equivalences, and "one report cancels 468 likes" is explicitly called an incorrect statement. The report head is −234 because the **baseline probability of a report is >1,000× lower than a like** — the weight exists to give a rare action enough influence to matter. **The same logic applies symmetrically to the positive weights** (platform framing echoed by the VP of Product on Aug 15, 2026): copy-link share (20.0) and DM share (5.0) are rare actions, so their weights are inflated relative to the common like (0.5); every term's contribution is `weight × P(action)`, so "reply is 10× a like" is a *weight* comparison, not a value equivalence — the ranking of levers depends on the (unpublished, per-viewer) predicted probabilities. The same comments argue that mass block/report campaigns cannot "significantly suppress reach" because (1) predictions are personalized — reports from bad actors "primarily affect recommendations for users who are similar to the bad actors," not everyone's ranking of the post; and (2) only actions on posts **served in the Home Timeline** count toward ranking (direct navigation, e.g. via groupchat coordination, has no ranking impact). This reframes the negative-weight math in [Section 14 of the companion account-analysis](account-analysis.md): the −234 is a bound on a *per-viewer* prediction, not a subtraction from a global score.

### C. Dwell-Regret Scoring Mode (experimental, off by default)

`ValueModelMode = "weighted" | "dwell_regret_sigmoid" | "gated_dwell_regret"`. In dwell-regret modes, RankingScorer replaces the weighted sum with (ranking_scorer.rs:535-600):

```rust
score = dwell_time × 2·sigmoid(Σ α·(P_i/mean_i − 1) / T) × exp(neg.min(0)/T)
```

i.e., dwell time modulated by a candidate's engagement probabilities **relative to the candidate pool's mean**, with negative-action exponential damping (`T = 10.0`, dwell floor 1.0). A logistic **gate model** (`value_model_gate.rs`, 19 viewer features: seq length, action counts over 28d/7d/1d, active days, account age, followers/following; threshold −0.634264, bias 1.033918) decides which viewers get it ("gated" mode).

### D. Candidate Sources

`phoenix_candidate_pipeline.rs:315-323` — **seven** sources:
1. **Thunder** (in-network, up to 1200) — @grok/@gork/@products posts are **blacklisted server-side** (thunder_service.rs:29-36)
2. **TweetMixer** (800; source ships disabled — `EnableTweetMixerSource=false`)
3. **Simclusters** (NEW; 800 max, `LOG_FAV_LONGEST_L2_EMBEDDING_TWEET` query → `LOG_FAV_BASED_TWEET` candidates, `MODEL_20M_145K_2020`, min score 0.5, ≤48h old, driven by explicit+implicit engagement signals)
4. **Phoenix retrieval** (1000)
5. **PhoenixTopics** (1000, reuses `PhoenixMaxResults`)
6. **PhoenixMOE** (200; ships disabled — `EnablePhoenixMOESource=false`)
7. **CachedPosts** (no cap; cache size `MaxPostsToCache=750`)

### E. VF (Visibility Filtering) — now open source

Final gate, per post × viewer: **ALLOW / INTERSTITIAL / DROP** (`visibility-filtering/rules/` — `tweet_rules.rs` + `author_rules.rs`, wired by `registry.rs`; Sept 2026: the old per-family rule files were consolidated with **zero behavior change** — same 28 + 26 rules in the same order, pinned by a `golden_corpus.rs` test contract). In-network posts are evaluated on the `TimelineHome` rule set; OON posts on `TimelineHomeRecommendations` (base rules + 26 OON-only drops that can only drop). Full rule catalogue in Section 12.5. Home-mixer sends candidates in 50-chunk batches (XAI client default 400 ms timeout; fails closed: missing results = `UnspecifiedReason` drop).

### F. Post-Type Handling: Native Articles Are Not a Distinct Type

X's native **Articles** (long-form posts) get **no special treatment anywhere in the feed path** — verified absence at every layer:

- **Model input**: `TweetInfo` carries no article flag; `TweetBoolFeatures` is only `is_for_you_page / is_promoted_tweet / is_reply / is_retweet / is_quote / has_media` (recsys.proto:1105-1112). No article feature exists in the training code or `feature_config.py`.
- **Scoring**: no article weight, no article head, no article adjustment in `RankingScorer`. In thunder, an article counts toward the **original** bucket (`MAX_ORIGINAL_POSTS_PER_AUTHOR = 50`; the per-author caps only distinguish replies 30 / originals 50 / videos 100 — thunder/config.rs:5-7).
- **Eligibility**: no article filter; articles obey the same `MAX_POST_AGE = 48h`; **no article retrieval window** exists in phoenix-rankall (evergreen pools are video-only), and no article engagement signal (the implicit-signal hydrator covers photo-expand/VQV/video only).
- **Analytics**: the `TweetTypeMetricsHydrator` bitset has `RETWEET/REPLY/SUBSCRIPTION_POST/VIDEO/…` but **no ARTICLE bit** (tweet_type_metrics_hydrator.rs:22-132).
- **VF**: no open rule references articles — the TES client trait's `get_is_article` method exists but no shipped rule calls it (only an `unreachable!()` test mock, tes_hydrator.rs:646).
- **Where articles do appear — content understanding only**: grox hydrates article metadata (title, media, cover) and feeds it into the multimodal-embedding renderers (`grox/flows/mm_emb/renderer.py:65-70, 88-89`, `renderer_v82.py:317-320`, `phoenix/reference/mm_encoder.py:143-166`) and tells its LLM classifiers the post is an article ("`[Article Post]` …", `grox/core/lm/post.py:95`; still no `[Video Post]` label anywhere in grox). So an article is judged **by its content** — title + body media + cover enter the post's embedding and therefore its SID — exactly like any other post's text/media, just with no *type* signal.

**Empirical check** (sample-1; collector flag `IsArticle` = bit 2, `PF = {IsQuote:1, IsArticle:2, HasPhoto:4, HasVideo:8, HasURL:16, HasMention:32}`): 37 of 10,000 posts are articles, 23 of them from one account (ilex_ulmus). Pooled medians: views/follower **25.95%** vs 9.65% for all other posts; engagement/view 0.86% vs 1.67%; median views 3,935 vs 5,051. The delivery advantage is real but rests on a tiny, one-account-skewed sample, and the dominant article poster is a small account — the cold-start boost alone can explain most of the gap. "Longer dwell" is an inference (dwell isn't in the dataset), not a measured signal (Section 13.5).

### G. The Role of Followers in Distribution: the unconditional seed channel

Follow status is the only thing that makes distribution *unconditionally offered*: a post is entered into every follower's candidate pool without passing any engagement or index gate. Whether it is actually *shown* is still a full per-viewer decision (below). Everything else in this section is about the gated channels; this subsection is about the one channel with no gate:

**The follower channel (Thunder) — unconditional candidacy, per-request, but still per-viewer prediction.** Your post enters every follower's candidate pool the moment they request the feed: in-memory 48h retention, no engagement threshold, no index membership, no discovery prediction — subject only to the `MAX_POST_AGE = 48h` window and per-author caps (1200 posts total / 50 originals per author). **Which followers actually see it is still decided by the full per-viewer pipeline**: the seen/served filters (a follower who was already shown it — or was served it in the last 10 minutes — is excluded; previously_seen_posts_filter.rs, previously_served_posts_filter.rs), muted keywords and blocks (both directions), the per-viewer Phoenix prediction × weights, author-diversity decay, the DPP diversity step (it can be zeroed if the follower's feed already picked similar content), the TopK-50 selection, and VF for that viewer (follower-exempt labels don't apply to followers; exclusive content still drops unless subscribed). So the *initial audience is not "whoever is online"* — it is the subset of followers who (a) request the feed within 48h, (b) haven't been shown it yet, (c) are not filtered, and (d) have a per-viewer predicted-engagement score high enough to survive the top-50 cut against ~1,000 candidates — in practice: **currently-active followers with a recent engagement history with you** (the model predicts them best). Passive followers, and followers who already saw the post in an earlier request, are ranked out or filtered out.

**The out-of-network channels are all gated** (phoenix-rankall-strato, eventProcessing.strato):
- **Index membership is engagement-triggered**: the `1fav`, `32fav`, `1fav_topic`, `video`, `nsfw_video` windows are built when a *favorite event* fires (phoenixRankAllCandidateProcessor.strato:78-92, 455-467) — and the training pipeline's **HOME retrieval corpus is literally the 1-fav pool** (`post_sid_v5_256x6_snapshots/1fav_1day.parquet`, retrieval_dataset.py:231-235). A 0-fav post is absent from the primary OON index.
- **The only 0-fav OON entry is the fresh window**: `post_creation_1day` (24h retention, no favs required — config/mod.rs:142) is built from the post-creation event, which arrives with a **2-minute delay** via the content-understanding topic (postCreationEventForwarder.strato:17). Index admission also consults VF at `TimelineHomeRecommendations` (`shouldDropPostByVF`, eventProcessing.strato:246-265) — posts VF would drop for non-followers never enter the OON index at all.
- **Fav events drive index membership — power-of-two thresholds with a 1-minute cooldown**: the **first** favorite triggers index processing immediately (the uncached path has no time throttle — only age <49h and fav count <2³⁰; favoriteEventProcessor.strato:136-148). Each later favorite re-triggers processing only when all five checks in the cached path pass (favoriteEventProcessor.strato:122-134, eventProcessing.strato:42-60): the fav count has reached the **next power-of-two threshold above the last processed count** (`nextPowerOfTwo` — 2, then 4, then 8, then 16, then 32; favoriteEventProcessor.strato:14-21), **≥1 minute** has passed since the last processing round, <48h have elapsed since the first round (`maxTimeSinceFirstUpdate`), the post is <49h old (`maxTweetAge`), and the fav count is still below 2³⁰ (`maxEngagements`). Consequences: the thresholds are *relative*, so triggers land at approximately the **2nd, 4th, 8th, 16th and 32nd favorites** (the intermediate rounds merely refresh the index); a second favorite within the same minute as the first is coalesced (no extra processing — but also no need: the first favorite already built `1fav_1day`); favorites that don't push the count to a power of two (the 3rd, 5th–7th, 9th–15th, 17th–31st) cause no re-processing at all; and the **first processing round that observes ≥32 favorites opens `32fav_1day`** (32 is a power of two, so the 32nd favorite triggers it on schedule — or the next round after it, if the 1-minute cooldown delays it). **Crucially, these triggers never expand any audience pool**: favorites play no role in the in-network channel at all (thunder serves followers' candidates from creation, fav-independent), and no OON window admits the post between 1 and 32 favs — the 2/4/8/16 rounds are index maintenance (metadata/embedding retries), not admissions. Audience growth with accumulating favorites runs through the model's **live engagement-count features** (60-second cache, bucketed — Section 5.1), which raise predicted engagement for every viewer in later requests: the momentum loop. The first two days decide a post's OON index membership.

**The emergent "followers-first" pattern.** The code never *restricts* a new post to followers — but the initial audience is overwhelmingly followers, and it is structural, not accidental: (1) thunder enters the post into every follower's pool instantly with no gate; (2) the main OON pool requires ≥1 fav, and the 0-fav fresh window is 2 minutes delayed and secondary (a content-only candidate); (3) even when proposed OON, a 0-engagement post must outscore in-network candidates while carrying the 0.75 OON discount. The typical lifecycle is therefore: **followers see it first → their favorites open the OON pools (the first fav triggers the `1fav_1day` build immediately — pipeline latency, not a 1-minute wait; the 32nd fav opens `32fav_1day`) → discovery expands — or doesn't, if the seed audience doesn't engage**. The empirical signature: dawn_com's 0.07% V/F is follower-only delivery with zero OON lift; the >100% V/F accounts (celestepoasts, atmoio) are the mirror — OON dominated because their posts cleared the gates. The user-experience intuition that "new posts are only shown to your followers" is directionally right as a *first-phase* description, wrong as a rule: followers are the *entry ticket*, not the audience.

**How large is the initial pool?** The code defines no "initial audience size" — the seed is *every follower feed request while the post is <48h old*, winnowed by four per-viewer gates: (a) **activity** — the follower must actually request the feed in that window; (b) **seen-dedup** — followers already shown the post are excluded (10-min served exclusion + seen filters), so the pool *shrinks as it converts*; (c) **filters** — muted keywords, blocks both directions, VF per viewer; (d) **per-viewer ranking** — the post must survive the top-50 cut of that follower's ~1,000-candidate pool. The only hard numeric caps are per-request/per-viewer (thunder: 1200 posts total, 50 originals per author), and the cold-start "low impressions" definition (<1000 **Home-Timeline** views since Sept). **The size advantage of larger follower bases is real but operates through velocity, not delivery probability**: more followers offered → more early favorites → faster crossing of the 1-fav/32-fav gates → earlier and larger OON expansion; the model never reads follower count, so per-viewer delivery probability is size-independent. Two seed-stage asymmetries: **videos ≥10s enter their video OON windows at creation with 0 favorites** (buildVideoIndex on the PostCreation event — phoenixRankAllCandidateProcessor.strato:468-473), while text/photos need ≥1 fav for the main OON pool — so video posts start with a broader initial OON pool as well as longer eligibility (video windows up to 5 years); and the mutual-follow +15 and cold-start boost raise the fraction of followers whose top-50 the post survives.

### Factors That Increase Reach

1. High predicted engagement per head (weighted sum above)
2. **Mutual-follow status**: original posts from authors who follow you back get reply weight 5→20 (documented in `docs/BIDIRECTIONAL_BOOST_CHANGE.md`; launched Jul 13 2026 at 20, reduced to 15 Jul 24)
3. **Cold-start boost**: posts by authors with ≤1000 followers and <1000 **Home-Timeline** views (Sept), non-reply/retweet, ≤24h old (treatment arm only), ranking within the top 85% of the scored pool (position ratio < 0.85) → one post lifted to slot 15 (author_cold_start.rs)
4. In-network status (no OON discount) and "post unexplored" boost (in-network only, 0.02)
5. Continuous dwell time (0.004, capped ≈0.12 max) + dwell probability head (**0.05** since Sept) — modest but real
6. Shares via copy link / DM / quote / reply predictions
7. Video quality views (`vqv`, gate: video duration strictly **>** `MinVideoDurationMs = 10,000` ms — an exactly-10-second video clears the implicit-signal gate but not this one; a viewer with ≥10K followers is exempt from the duration gate; **Sept: the VQV scoring weight is now 0.0, so the gate feeds only the implicit discovery signal, not the score**)
8. **DPP**: candidates with distinct embeddings (esp. multimedia) survive diversity selection better

### Factors That Diminish Reach

1. Negative predictions: report (−234) > mute (−58.8) > not-interested (−43.2) > block (−31.2)
2. OON discount 0.75 (0.5 topics; 0.00001 for eligible new users); replies/reposts from followed accounts also discounted (only when `EnableOonRescoreForInNetworkRepliesRetweets` is on) (gated — `EnableOonRescoreForInNetworkRepliesRetweets`)
3. Author repetition (diversity decay 0.5^k, floor 0.25) — and DPP diversity can zero out similar posts entirely
4. VF drops (11 user labels, 17 tweet labels — see 12.5), nullcast, paywall, age > 48h
5. Old posts: `MAX_POST_AGE = 48h`; model age buckets cap at 80h

---

## 2. Account Attributes Affecting Post Reach

### Direct Account Attributes (gizmoduck_hydrator.rs)

Fetched per candidate author: **`followers_count`**, **`screen_name`**, retweeted screen name, **NSFW flags** (`nsfw_admin || nsfw_user || NSFW_HIGH_PRECISION` label → `nsfw_author`; + `POSSIBLY_NSFW_ACCOUNT` → `nsfw_author_ads`), plus a new **`nsfw_author_phoenix`** model input (nsfw_user || nsfw_admin || NSFW_HIGH_PRECISION || POSSIBLY_NSFW_ACCOUNT — stamped as `SAFETY_BIT_AUTHOR_NSFW` on non-retweets; gizmoduck_hydrator.rs:139-149, candidate.rs:238-244). Still **no verification status** — the algorithm never reads `is_blue_verified` (Section 12.6; empirically confirmed in Section 13.7). **Sept:** on retweets, the model no longer receives the author context at all — `is_following_user`/`followers` are now `None` when a retweet id is present (candidate.rs:265-274).

### New account-level inputs (Aug 2026)

1. **Bidirectional follow** (`bidirectional_follow_hydrator.rs`): `is_mutual_follow_author` — mutual follows get the +15 reply-weight boost on originals.
2. **Engagement counts** (`engagement_counts_hydrator.rs`): fav/reply/repost/quote/view/bookmark counts per post (60 s TTL cache) — fed to the model and to cold-start eligibility.
3. **Author cold-start eligibility** (`author_cold_start.rs`): `author_followers_count ≤ 1000` + **Home-Timeline views** `view_count_on_home < 1000` (Sept: was all-surface `view_count`) + not reply/retweet + ranked within top 85% of the scored pool → one boost per request (the ≤24h age check applies in the experiment's treatment arm; the default holdout arm ignores age).
4. **Semantic IDs** (`semantic_id_hydrator.rs`): 6-level SID codes per post (multimodal-derived).
5. **Media info** (`media_info_hydrator.rs`): `has_media`, `min_video_duration_ms` (replaces the removed `has_media`/`video_duration` hydrators).
6. **New-account viewer filter** (`new_user_min_engagement_filter.rs`): for viewer accounts < 30 min old (or resurrected within 30 min), OON posts below an engagement threshold (metric `fav`, ratio mode off) are dropped. **The filter ships disabled** (`EnableNewUserMinEngagementFilter=false`).
7. **Author-level experiment buckets** (`author_rules.rs`): authors can be bucketed into experiments (e.g., MoE codivert treatment/control) that change whether their posts' scores survive.
8. **AI-trend feedback context** (`ai_trend_feedback_context_hydrator.rs`, Sept): when `EnableAiTrendFeedbackContext` is on (ships off), the top-2 trending topics among eligible originals are hydrated as post context (`ai_trend_name/id`) — a possible future discovery signal, currently inert.

### Indirect account influence

- **User-cred score** (Section 12.2): gates spam labeling and enforcement, not feed scoring directly.
- **Safety/user labels** (Section 12.5): 11 user labels → VF drops (e.g., SpamHighRecall, Compromised, Impersonation, NSFW*, ReadOnly, DoNotAmplify — the last two exempt followers).
- **Subscription status** (subscription_hydrator.rs + VF `DropExclusiveTweetContentRule`).
- **Viewer-side**: followed topics/starter packs, muted keywords, installed apps, inferred gender, IP geo — all sent in the Phoenix request context.

---

## 3. Filtering and Content Eligibility

### Pre-Scoring Filters (in order, phoenix_candidate_pipeline.rs:346-373)

| # | Filter | Removes |
|---|---|---|
| 1 | DropDuplicates | same post from multiple sources |
| 2 | CoreDataHydration | posts that failed text/metadata load |
| 3 | **AgeFilter** | posts > **48h** (`MAX_POST_AGE`, config.rs:36) |
| 4 | SelfTweet | viewer's own posts |
| 5 | **OONRetweetReply** | OON reposts, OON replies, replies with missing parents |
| 6 | OONNsfwSimclusters | SimClusters posts by NSFW authors (viewer not following) |
| 7 | RetweetDeduplication | duplicate reposts |
| 8 | IneligibleSubscription | subscriber-only posts |
| 9-11 | PreviouslySeen ×2 (+backup), PreviouslyServed | already shown/served (bloom filters) |
| 12 | **ViewerMutedKeyword** (renamed; was `MutedKeyword`) | posts matching viewer's muted keywords (tokenizer-based) |
| 13 | AuthorSocialgraph | viewer blocks/mutes author, **author blocks viewer**, quoted/retweeted author blocks viewer |
| 14 | **Brazil2026Election** | posts from **2,388 hardcoded user IDs** (Sept: was 665; grew 665→2,328 on Aug 25, →2,315, →**2,388** on Sept 1 — test asserts `len() == 2388`, brazil_2026_election_filter.rs:5048;/README notes "Account list updated August 27, 2026") reported to Brazil's Electoral Court (brazil_2026_election_filter.rs, 5,082 lines), plus retweets/quotes of them and replies with them among their ancestors — unless the viewer **follows** the account (no feature switch) |
| 15 | Video | video posts when request excludes video |
| 16 | TopicIds | out-of-topic / excluded topics (1502 lines; topic filtering `PostBased50Pct`) |
| 17 | NewUserMinEngagement | new-account viewer: low-engagement OON posts (ships disabled) |
| 18 | InventoryHoldout | deterministic % holdout per (post, viewer) via splitmix hash, 100 buckets (ships disabled — `EnableInventoryHoldout=false`) |

**Sept: the Following pipeline gained parity filters** — `FollowingViewerMutedKeywordFilter` (mutes also match quoted-tweet and reply-ancestor text, via new `quoted_post_text`/`ancestor_texts` hydrators), `FollowingBlockedByHydrator` (author-blocks-viewer now enforced), and its own `VFFollowingCandidateHydrator` (TimelineHome-only verdicts); night-owl sources now populate `retweeted_user_id` to feed these. The For You order above is otherwise unchanged (Sept table: 18 filters, matching the code — the README table's Aug-era 17-vs-18 drift is now moot).

### Post-Selection (phoenix_candidate_pipeline.rs:400-421)

`VFCandidateHydrator` (in-network candidates **and retweeted source ids** → TimelineHome rules; OON candidates + ancestors/quotes → Recommendations rules; 50-chunk, 400 ms) → **VFFilter** (drop on any non-SafetyResult reason, or SafetyResult with `Action::Drop`) → `AncillaryVFFilter` (drop posts whose ancestors/quotes/reposts dropped) → `DedupConversationFilter`.

**Repost semantics for the filters:** `AgeFilter` computes age from the candidate's *own* snowflake id (age_filter.rs:16-20) — a fresh repost passes 48h even when its original is old — but the model's post-age features are computed from the *original* id that home-mixer sends (candidate.rs `get_original_tweet_id()` → util.rs `original_age_sec`, xai-recsys util.rs:505), so the repost is scored with the original's age. `PreviouslySeenPostsFilter` also checks the retweeted id (`related_post_ids_iter` now also covers reply parents — candidates_util.rs:13-17), so reposts of posts a viewer already saw are dropped; `RetweetDeduplicationFilter` keeps the *first-seen* copy per original per request (first wins, original included — retweet_deduplication_filter.rs:19-29). Net effect for a self-repost of a poorly performing post: the same post id → same embedding and author state, re-scored with the original's (worse) age features, deboosted ×0.75 in-network (if the rescore switch is on), removed for everyone who saw the original — with no freshness reset; the only structural benefit is a fresh retrieval-index entry (the repost is a new id in the fresh windows). See account-analysis Part 22.7.

### Other pipelines' filters

- `SelfReplyChainFilter` + `ConversationGapAncestorHydrator` (reverse-chron pipeline only — Sept: ancestor-expand failures now fall back to original ancestors instead of erroring, and the hydrator also fills `ancestor_texts` for mute matching), `FollowingRetweetDeduplication`, `AdAdjacentServed` (blending), `PushToHomeDedup`, `InvalidConversationModule`, `PopularTopicsAuthorDedup`.

---

## 4. Key Algorithm Insights

1. **No vocabulary — hash embeddings everywhere**: 2 hash functions each for user/item/author (moduli 2_859_568_897 / 2_361_375_383 / 631_860_353). New posts representable immediately. (README claim #3 ✓)
2. **Candidate isolation in ranking**: candidates cannot attend to each other; right-anchored RoPE positions; scores consistent and cacheable. (✓)
3. **Multi-action prediction**: 60 actions + 8 continuous heads; weighted blend is an explicit, tunable, **fully visible** step.
4. **Ranking and visibility are separate services** (README design decision #4): `home-mixer` ranks; `visibility-filtering/` decides show/drop. The VF service is now open and its rules are enumerable.
5. **The "no hand-engineered features" era is over as a claim**: the README no longer makes it, and the code is full of hand-built features (engagement-count buckets, tweet-type metrics, VQV duration gates, follower thresholds, mutual-follow boosts, cold-start heuristics, minhash Jaccard).
6. **Continuous training code is released**; no checkpoint ships. Training data pipeline: `reference/dump_gen.py` synthetic dumps + `world.py` simulation (10,000 users / 2,000 authors / 50,000 posts / 16 topics).
7. **UAS (user action sequence)**: aggregated per-viewer sequences are the model's main input; window is **5 minutes** (`UAS_WINDOW_TIME_MS = 300_000`) with `MaxSeqLengthScoring/Retrieval = 1024`.

---

## 5. HIGH IMPACT FACTORS: Deep Dive Analysis

### 5.1. The Phoenix Model — now fully specified (production configs)

**Ranking (`home_direct_packed`, phoenix/xrex/configs/xrecsys.py:239-313):**

| Config | Value |
|---|---|
| layers | **8** |
| emb_size | **2560** (emb_table_width 1024) |
| attention | 20 query / 4 KV heads, key_size 128, qk_norm, right-anchored RoPE, **causal=False** (candidate isolation), flash-attention v3/v4 (Pallas/CUTEDSL) |
| history_seq_len | **1022** (seqpack, packed per-user events, block 128) |
| candidate_seq_len | **64** |
| output | **64 action logits** (60 used) + 8 continuous heads (DWELL_TIME MAE/tweedie) |
| SID | 6 levels × 256-code codebook, 1024-dim, hash-level + optional cross-attention |
| vocab | user 100M, item 100M, author 30M, IP 10M (hashed, 2 hashes each) |
| training | total 1e11 samples, rowwise-AdaGrad embeddings (**lr 0.28**, half-life 2500 H100 / 5000 GB300, lazy decay + small wd — Sept), **Muon** dense (Sept; was AdamW: 7.1e-4 H100 / 5e-4 GB300), checkpoint every 150/300, copy_port 9988 |
| post age | `POST_AGE_MAX_MINUTES = 4800` (~80h) bucketed hourly (82 buckets incl. overflow) |

**Sept: `home_direct_packed` now learns from live counts + follow directions + stale posts.** The production feature-prep config enables `enable_engagement_counts` (live fav/reply/repost/quote/view counts through a log2(1+c)/32 → MLP-64 → embedding path — velocity now shapes predictions directly, not just via buckets), `enable_is_author_followed_by_viewer` + `enable_is_author_following_viewer`, and `enable_stale_post` (recsys_feature_prep.py:628-667, xrecsys.py:272-292). The bool-feature indices are now **shipped** (`feature_config.py:42-44`: `isStalePost14d=0`, `isAuthorFollowedByViewerSeq=1`, `isAuthorFollowingViewerSeq=2`; `STALE_POST_14D_TTL_SEC = 1,213,200`).

**Features** (recsys_feature_prep.py:130-217, 382-592, 609-667): user token (hashed user id + IP) + user-features token (country 250, language 200, location Fourier 32-d, gender, **inferred gender**, age brackets, installed apps 64-multihot); history/candidate tokens = post (2 hashes) + author (2 hashes) + action multihot + product surface (16) + post-age buckets + timezone (32) + hour-of-day (25) + SID + multimodal-embedding projection (v5, 1024-d) + engagement-count buckets (32; fav capped 18, reply 13, repost 15, quote 13, view 25). **Sept: `home_direct_packed` additionally feeds live counts through a log2(1+c)/32 → MLP-64 embedding path plus both follow-direction flags** (recsys_feature_prep.py:609-667) — velocity and mutual-follow shape predictions directly, on top of the buckets. **Negative-feedback masking**: any of 9 negative actions on a candidate zeroes all other (non-negative) heads (recsys_model.py:2781-2823) — **ships disabled**: `mask_candidate_positive_when_negative_action_present=False` by default and no shipped config enables it.

**NEW (Aug 14, shipped since Sept):** three serving-side bool features (`phoenix/crates/common/xai-recsys/src/util.rs`): (a) **`IS_AUTHOR_FOLLOWED_BY_VIEWER`** — viewer follows the candidate's author, and (b) **`IS_AUTHOR_FOLLOWING_VIEWER`** — the author follows the viewer (mutual-follow direction), both stamped on candidate **and** history tokens from `isAuthorFollowedByUser` / `author_info.is_following_user`; and (c) **`IS_STALE_POST14D`** — since the Sept update, `home_direct_packed` sets `enable_stale_post: True` (xrecsys.py:272), so candidates older than `STALE_POST_14D_TTL_SEC` = **1,213,200 s** (~14.04 days; now shipped in `feature_config.py:74`) have their engagement counts **zeroed** and the flag set.

**Retrieval (two-tower, recsys_two_tower_model.py + xrecsys_two_tower.py):** user tower = transformer over history + user features (**no learned per-user embedding**, `use_user_embedding=False`); candidate tower builds representations from **SID + hashed author + post features** (max_posts 10.24M table, 28.67M combined); cosine top-K, in-batch negatives + **64 global negatives**, log-Q correction, learned temperature; two heads: `home` (positive = favorite) and `immersive` (8 positive actions). Sept: retrieval attention is now normalized + uncapped (`qk_norm` False→True, `attn_logit_cap` 80.0→disabled) — discovery similarities shift slightly, heads/targets unchanged. Retrieval serves top-K from an **index stored inside the checkpoint** (phoenix/README.md:114-117) — maintained by `phoenix-rankall` (Section 5.4).

**Gen-Recs & SID retrieval (new model families):** `recsys_gen_recs_model.py` — generative retrieval that predicts the target post's **multimodal embedding** directly (autoregressive decoding against the post table); `recsys_sid_retrieval_model.py` — seq2seq model that generates the **next-post SID** (6 codebook levels, beam width 128). Both experimental complements to two-tower.

**Serving:** Rust engine (`xai-recsys-engine`: admission control, SID hydration, embedding gather, checkpoint distribution, retrieval top-K) + Python/JAX inference (`xrex/inference/model_runner.py`). gRPC `RecsysPredictor.PredictNextActions` / `RecsysRetrievalPredictor.RetrieveTopKCandidates`; request carries `UserActionSequence`s (Arrow columnar), `TweetInfo[]` (author flags, engagement counts, **semanticIds**, language, media bools), `UserContext` (age/gender/geo/apps/followers/following/blockers/muters counts).

**No checkpoint ships**: TRAINING.md:90 "No pretrained checkpoint is included"; training is on synthetic data (QUICKSTART seed 20260721, 12,288 rows; world.py sim). The old `oss-phoenix-artifacts.zip` (2.9 GB, 128-dim/4-layer) is **gone**.

### 5.2. Filtering Pipeline: Binary Impact Factors

Unchanged in structure; now with exact values: `MAX_POST_AGE = 48h` (was "unknown, likely 6-24h" — the estimate was wrong), `MinVideoDurationMs = 10,000`, inventory holdout % (0 by default — ships disabled), `NewUserMinEngagementFilter` (30-min account age window; ships disabled). VF adds the binary post-selection gate (Section 12.5).

### 5.3. Scoring Multipliers — with real numbers

1. **VMRanker DPP** (highest impact, post-scoring): greedy-MAP determinantal point process over **v5 multimodal embeddings (f16, 1024-d)**. Quality score `qf = exp(α·(score/max_score))`, kernel `L_ij = qf_i·qf_j·cos(emb_i, emb_j)`, `α = θ/(2(1−θ))`, θ = 0.65 (client) / 0.5 (server). Candidate pool = top `max_selected_rank` by score (150 client-sent / 100 server default); greedy selection stops at `top_k` = 50 or epsilon; **non-selected candidates are scored 0.0 → effectively removed**. Missing embeddings → random unit vector. Duplicate/identical embeddings collapse to one survivor; orthogonal content is promoted. This is the *only* diversity mechanism that can remove in-network posts.
2. **Author diversity** (0.5^k, floor 0.25) — per-pool exponential decay, resets per request.
3. **OON factor** (0.75 / 0.5 topics / 0.00001 new-user) + in-network reply/retweet deboost (gated by `EnableOonRescoreForInNetworkRepliesRetweets`).
4. **Cold-start boost**: lift one eligible post to **slot 15** (`ColdStartSlotMin=15, Max=16`) — eligibility: author ≤ 1000 followers, **< 1000 Home-Timeline views** (Sept: `view_count_on_home`; treatment-arm-only ≤ 86400 s age check), position ratio **< 0.85** (i.e., ranked within the top 85% of the non-zero-scored pool; the bottom 15% are ineligible). `EnableViewerColdStart=true`. **Thompson-sampling variant** (`EnableColdStartThompsonSampling`, off by default) draws a Beta(α0 + favs, β0 + views·scale − favs) reward per eligible post (α0=0.75, β0=49.25, scale=1.0), keeps the **top-2** sampled rewards (Sept: was top-5, `ColdStartTsTopK` 5→2), and promotes the highest-scoring post among them — favoring the *least-proven* candidate when data is near-zero (author_cold_start.rs).
5. **Bidirectional reply boost** +15 (reply 5 → 20) — the largest single positive weight in the system.
6. **Dwell-regret mode** (off by default; gate model controls rollout).

### 5.4. phoenix-rankall: what gets into the retrieval index (Sept 2026: Main 17 / SID 15)

Kafka (`phoenix_rank_all_indexing_event`) → windowed parquet indexes per window name (config/mod.rs:141-198; Sept: Main 14→**17**, SID 10→**15** — four age-banded `1fav_video` windows added each, `evergreen_video_grok_30day` removed from both, plus bounded `video_4to14day`/`nsfw_video_4to14day` in SID):

- **Main (17)**: post_creation_1day · 1fav_1day · 1fav_2day · **1fav_video_2day · 1fav_video_2to4day · 1fav_video_4to14day · 1fav_video_4to30day** (NEW Sept — fav'd videos fan out to `video` *and* `1fav_video` streams) · **32fav_1day** · video_2/4/7/14/30day · nsfw_video_2/7day · **evergreen_video_1825day** (5 years!) · evergreen_nsfw_video_1825day (grok pool deleted)
- **Topic (7)**: 1fav_1day + topic variants ×5; **SID (15, was 10)**: 1fav_1day, 1fav_video_2day/2to4day/4to14day/4to30day, video_2/4/**14**day, nsfw_video_2/7/14day, evergreen_video 5y, imagine_4day, video_4to14day, nsfw_video_4to14day (NO grok pool); **SidTail**: long-tail authors (< 1000 followers, ≥ 0 favs — note `tail_min_fav_count = 0`).
- **Sept mechanics — min-age windows + 0-fav emission:** bounded `*4to14day`-style windows only serve posts **older** than `min_age` (young posts are held in a `prefloor_*` sidecar and survive restarts from it; `PREFLOOR_SLACK_SECS=6h`); the fav-event processor now emits `1fav` **unconditionally** (0-fav race no longer misses the index) and `buildMMEmbMetadataDump` covers 0-fav posts too; the fresh-post path moved to the v3 content-understanding topic (2-min delay preserved).



**Engagement thresholds decide corpus membership** — a post with ≥1 fav enters the 1-day index; ≥32 favs enters 32fav_1day; videos have their own windows up to 5 years (evergreen + Sept's age-banded 1fav_video_* slices). Blacklist files (`blacklisted_post_ids.txt`, `blacklisted_author_ids.txt`) applied in the topic pipeline. The **Analysis pipeline** archives served phoenix scores (`home_mixer_phoenix_scored_candidates`, prefix `phoenix.HomeExperiment1Lap7.`, 7-day retention, parquet; Sept: the cached score payload no longer carries per-head prediction scores).

**Index admission mechanics (phoenix-rankall-strato, Section 1.G):** the `post_creation` window is built from the content-understanding unified-posts topic with a **2-minute forwarder delay** (postCreationEventForwarder.strato:17; Sept: topic moved to v3, key-first `(Long, Event)`), no engagement required, 24h retention. The `1fav` / `32fav` / `1fav_topic` / `video` / `nsfw_video` windows are built **only on favorite events** (phoenixRankAllCandidateProcessor.strato:455-469; Sept: fav'd immersive videos fan out to **both** `video` and the new `1fav_video` streams, and `build1FavIndex` now emits **unconditionally** — the 0-fav race no longer misses the index, counted via `1fav_uec_count_race`); the first favorite triggers immediately, and later favorites re-trigger only when the count crosses the next **power-of-two threshold** (2, 4, 8, 16, 32, … — favoriteEventProcessor.strato:14-21) at most **once per minute**, stopping 48h after the first update / 49h after creation (eventProcessing.strato:28-40). Index admission consults VF at `TimelineHomeRecommendations` (`shouldDropPostByVF`) — posts VF would drop for non-followers never enter the index. The training pipeline's HOME retrieval corpus is the **1-fav pool** (`1fav_1day`); `post_creation_1day` is the default global-negative pool in the ranking data pipeline (retrieval_dataset.py:231-235, parquet_recsys.py:863-865).

---

## 6. Reach Optimization: Hierarchy of Influence (updated with real values)

### 🏆 TIER 1: CRITICAL (binary elimination)

1. **VF drops** (11 user labels, ~20 tweet labels incl. SPAM/PDNA/BOUNCE/FOSNR, legal takedowns, nullcast, NSFW gating, exclusive content) — Section 12.5
2. **Post age > 48h** (`MAX_POST_AGE`)
3. **Author suspended/deactivated/protected + viewer blocks** (both directions)
4. **Muted keywords**, seen/served dedup, self posts
5. **DPP non-selection** (score forced to 0) — especially for near-duplicate content

### 🥈 TIER 2: HIGH IMPACT (large multipliers)

1. **Report/mute/not-interested/block predictions** (−234/−58.8/−43.2/−31.2)
2. **In-network vs OON** (0.75; 0.5 topics; 0.00001 new users) + in-network reply/retweet deboost
3. **Mutual-follow original posts** (+15 reply weight → effectively ~20 vs 5)
4. **Cold-start boost** (≤1000 followers + <1000 Home-Timeline views → slot 15)
5. **Share via copy link / DM / quote / reply predictions** (20/5/5/5)
6. **Dwell time** (0.004, capped ~0.12 — modest; Section 1.B)

### 🥉 TIER 3: MEDIUM

1. Video duration ≥ 10 s (VQV gate, implicit-signal minimum — Sept: the VQV *weight* is 0.0, so the gate now matters only via the implicit discovery signal, not scoring) + video_open (0.07)
2. Author diversity decay (0.5^k)
3. Follow-author prediction (4.0)
4. Engagement-count features (log2-MLP path + buckets) + follower count (as model inputs & cold-start gate)
5. New-user OON lockdown (if enabled)

### 🏅 TIER 4: LOW

1. Likes (0.5), photo expand (0.05), click (0.4), open link (0.2)
2. Product surface / timezone / hour-of-day context
3. `profile_click` (0.0 default), `quoted_vqv` (0.0), `active_secs` (0.0)

---

## 7. Important Files Summary

| File | Purpose | Notes (Aug 2026) |
|------|---------|------------------|
| `README.md` | System overview | Rewritten Aug 13, 2026; see Section 11 |
| `home-mixer/params/param.rs` | **ALL weights + FS defaults** | **OPEN**: exact values in Section 1.B |
| `home-mixer/params/config.rs` | Structural constants | MAX_POST_AGE=48h, TOP_K=50, RESULT_SIZE=35, UAS window 5 min, NEGATIVE_SCORES_OFFSET=0.001 |
| `home-mixer/scorers/ranking_scorer.rs` | Active scoring | Weighted + diversity + OON + `MultiplierPreOffset` (MPN removed Sept) + dwell-regret; 1572 lines |
| `home-mixer/scorers/phoenix_scorer.rs` | ML client | 2800 candidates max (phoenix_request.rs), cluster routing, stamps backbone scores + slate context |
| `home-mixer/scorers/vm_ranker.rs` | VMRanker client | Slim request (ids + score, hardcoded "dpp"); fold logic deleted Sept |
| `home-mixer/scorers/author_cold_start.rs` | Cold-start boost | ≤1K followers, <1K Home-Timeline views → slot 15; Thompson-sampling variant (off), top-2 since Sept |
| `home-mixer/scorers/value_model_gate.rs` | Dwell-regret gate | 19-feature logistic, threshold −0.634264 |
| `home-mixer/filters/brazil_2026_election_filter.rs` | Brazil 2026 election filter | **2,388** Electoral-Court-reported user IDs (Sept; test asserts 2388); drops their posts/retweets/quotes/thread ancestors unless the viewer follows (Section 3) |
| `home-mixer/filters/viewer_muted_keyword_filter.rs` + `following_viewer_muted_keyword_filter.rs` | Mute filters | Sept rename + Following variant (matches quoted/ancestor text) |
| `vm-ranker/` | Reranking service | **OPEN**: DPP only; head-model redacted |
| `visibility-filtering/` + `-client/` | Content gate | **OPEN**: full rule registry (Section 12.5) |
| `user-cred-v2/` | PageRank credibility | **OPEN**: score = 165.2 + 7.07·ln(mass) (Section 12.2) |
| `abuse-enforcement-service/` | Enforcement | suspend/label/challenge; cred ≥ 50 exempt; Sept: new panda-reports suspend rule + GrowthBook-gated generic-action path |
| `scarecrow/` + `botmaker/` + `botmaker-rules/` | BotMaker rules | tweet labels, spam-label store (TTL 26h); Sept: new XReview human-review reporting path |
| `bdsm/` | Inauthentic-behavior model | 8 heads incl. LegitimateUser; thresholds redacted; Sept: the ≥30 action-count gate is now redacted too (`999999`) |
| `agatha/` | User-label modeling | PMI features over graph signals |
| `safety-label-user-agg/` | Post→user label rules | POSSIBLY_NSFW_ACCOUNT etc. |
| `phoenix/xrex/` | **Training + inference code** | real configs (8L/2560), synthetic data; no checkpoint |
| `phoenix/xrex/configs/xrecsys*.py` | Prod model configs | Section 5.1 table |
| `phoenix-rankall/` | Retrieval index maintenance | windowed parquet, SID hydration, analysis store |
| `simclusters/` + `home-mixer/sources/simclusters_source.rs` | 3rd OON source | MODEL_20M_145K_2020, min score 0.5 |
| `thunder/` | In-network store | 48h retention, 2-min trim, 8K QPS, @grok blacklist |
| `grox/` | Content understanding | reply-spam buckets ≤100/≤500/≤1000; upa deluxe fav ≥ 64 |
| `under-the-hood/` | Transparency tool | labels per account/post; 365d age + 10 posts/mo |
| `home-mixer/scorers/weighted_scorer.rs` etc. | Legacy scorers | **DELETED** in this release |

---

## 8. Strategic Recommendations for Reach (updated)

### 🎯 HIGH PRIORITY (must optimize)

1. **Stay under 48h post age**; freshness matters (model age buckets + AgeFilter).
2. **Avoid any safety/spam label**: SPAM, SPAM_HIGH_RECALL, BOUNCE, PDNA, FOSNR, NSFW*, DO_NOT_AMPLIFY → VF drops (author-exempt for some; follower-exempt for AbusiveHighRecall/DoNotAmplify).
3. **Mutual follows**: get authors of large accounts you want to reach to follow you back — original posts get reply weight 5→20 (plus a dormant +2.0 dwell-bonus mechanism that ships at 0). (Replies/RTs of mutual authors do NOT get the boost.)
4. **Avoid actions that train report/mute/block predictions** (−234/−58.8/−43.2) — these dominate the negative side.
5. **Drive reply + share-via-DM/copy-link + quote + follow conversions** — the highest positive weights.

### 🚀 MEDIUM PRIORITY

1. **Small accounts: exploit the cold-start boost** — accounts ≤1000 followers with <1000-view **Home-Timeline** posts can get one post lifted to slot 15 per request (non-reply/non-retweet, ≤24h old in the treatment arm). Low-impression posts benefit structurally.
2. **Video ≥ 10 s** for the implicit video signal and video_open head (0.07). Sept note: the VQV *scoring* weight is now 0.0 — "make it ≥10 s" still buys the implicit discovery signal plus evergreen eligibility, but no score term.
3. **Dwell time**: modest but real (0.004 × predicted dwell, capped ~30 s ≈ 0.12 max — see Section 1.B). Long reading sessions help, but replies/shares dwarf them.
4. **In-network growth** (followers) — OON 0.75 penalty, and new-user viewers get 0.00001.
5. **Distinct content** (avoid near-duplicates) — DPP zeroes similar posts.

### 📈 LOW PRIORITY

1. Likes (0.5) — signal, but 10× weaker than reply weight.
2. Profile completeness / verification — **still not read by the feed algorithm** (affects enforcement-exemption indirectly via user-cred premium teleport; Section 12.2/13.7).

---

## 9. Discovered Constants vs Unknown Values (Aug 2026)

### ✅ FOUND VALUES (in code, defaults)

**Scoring (home-mixer/params/param.rs; Sept 2026):** favorite 0.5 · reply 5.0 · retweet 1.0 · photo_expand 0.05 · video_open **0.07** · click 0.4 · open_link 0.2 · profile_click 0.0 · vqv **0.0** · share 2.0 · share_via_dm 5.0 · share_via_copy_link 20.0 · dwell (prob) **0.05** · quote 5.0 · quoted_click 0.05 · quoted_vqv 0.0 · follow_author 4.0 · post_unexplored 0.02 · cont_dwell_time 0.004 · cont_click_dwell_time 0.0 · active_secs_5m 0.0 · not_interested −43.2 · block_author −31.2 · mute_author −58.8 · report −234.0 · not_dwelled −0.02 · **bidirectional reply boost +15.0** · dwell boost 0.0 · **AuthorDiversityDecay 0.5 · Floor 0.25** · **OonWeightFactor 0.75 · TopicOon 0.5** · **NEW_USER_OON_WEIGHT_FACTOR 0.00001 · NEW_USER_MIN_FOLLOWING 5** · NewUserAgeThresholdSecs 0 (off) · NEGATIVE_SCORES_OFFSET 0.001 · MinVideoDurationMs 10,000 · EnableQuotedVqvDurationCheck false

**Structure (params/config.rs):** **MAX_POST_AGE = 48h** · **TOP_K_CANDIDATES_TO_SELECT = 50** · RESULT_SIZE = 35 · FOR_YOU_MAX_RESULT_SIZE = 47 (35+4+8) · RANKED_FOLLOWING_MAX_RESULT_SIZE = 38 · FOLLOWING_MAX_RESULT_SIZE = **110** (Sept; was 100) · **UAS_WINDOW_TIME_MS = 300,000 (5 min)** · MaxSeqLengthScoring/Retrieval = 1024 · EngagementSignalsMaxPerType = 15 · WHO_TO_FOLLOW_POSITION 6 · PROMPTS_POSITION 0

**Cold start:** ColdStartImpressionThreshold 1000 (**Home-Timeline** views since Sept: `view_count_on_home`) · ColdStartSlotMin/Max 15/16 · ColdStartFollowerCap 1000 · ColdStartMaxPostAgeSecs 86400 · LowImpressionsMaxPositionRatio 0.85 · EnableViewerColdStart true · EnableColdStartThompsonSampling false · ColdStartBetaAlpha0 0.75 · ColdStartBetaBeta0 49.25 · ColdStartTsTopK **2** (Sept; was 5) · ColdStartImpressionScale 1.0 · ColdStartTrackedIds ""

**VM ranker:** cluster "Experiment3" · hardcoded `value_model_id="dpp"` (FS keys removed Sept) · θ 0.65 · max_selected_rank 150 · 400 ms timeout · server: top_k 50, θ 0.5, max 100, embedding dim 1024 (v5 f16) · slim request (tweet/retweeted ids + score only — no follower counts, head weights, slate context) · MPN fold path **deleted**

| training | total 1e11 samples, rowwise-AdaGrad embeddings (**lr 0.28**, half-life 2500 H100 / 5000 GB300, lazy decay + small wd — Sept), **Muon** dense (Sept; was AdamW: 7.1e-4 H100 / 5e-4 GB300), checkpoint every 150/300 |

**Retrieval (two-tower):** history 1023 · candidate 64 · emb 1024 · 8 layers · 16/4 heads · max_posts 10.24M (28.67M combined) · 64 global negatives · 2 heads (home/immersive) · pos = fav · hard negs 7 (report, not-interested, see-fewer, unfollow, block, mute, not-relevant) · soft neg = not-dwelled · Sept: attention normalized + uncapped (`qk_norm` on, logit cap disabled)

**user-cred-v2:** jump probability α 0.2 · engagement blend β 0.5 · 7-day engagement window · convergence 0.001 / max 50 iters · score 165.2 + 7.07·ln(mass), clamp [0,100] · VIT threshold 55 · gates ±5% (rows, mass [0.99,1.005], VIT count) · premium = blue/gray/gold/org/affiliate

**Enforcement:** 500,000/day + 500,000 post/day caps (`MAX_ENFORCEMENTS_PER_DAY`, `MAX_POST_ENFORCEMENTS_PER_DAY`, config.rs:241-244) · 24h dedup · 15-min skip dedup · cred skip: score ≥ 50 or is_high · scarecrow high-PR threshold **54** · fallback followers > 25,000 (only when no score and no `PREDICTED_HIGH_PAGE_RANK` label) · spam-label store TTL 26h · HLL precision 60 s / TTL 25 h · **Sept: new `panda_reports_embedding_v10_rough_spam → suspend` rule + GrowthBook-gated `requested_actions` generic-action path (model-requested suspend/label/challenge; inert until allowlisted)**

**VF:** adult age 18 · 16 gating countries (Sept: same defaults, now **runtime-configurable** via feature switches, `visibility-filtering/params.rs`) · safety-label cache TTL 30 s (<5 min posts) / 60 s · twemcache 20 ms · manhattan SCAN_LIMIT 512/200 ms · exclusive-content rule · follower-exempt labels: AbusiveHighRecall, DoNotAmplify · Sept: `rules/` consolidated (`tweet_rules.rs` + `author_rules.rs`, same rules/order, golden-corpus pinned)

**Thunder:** retention 172,800 s (48h) · trim 2 min (5 shards, 10-min cycle per user) · max 1200 posts / 600 videos / reply 30·author / original 50·author / video 100·author · 8,000 QPS · blacklist @grok/@gork/@products

**Rankall windows (Sept 2026):** post_creation_1day · 1fav_1day/2day · **1fav_video_2day/2to4day/4to14day/4to30day** (NEW) · 32fav_1day · video_2/4/7/14/30day · nsfw_video_2/7day · evergreen_video_1825day (5y) · evergreen_nsfw_video_1825day · ~~evergreen_video_grok_30day~~ (removed) · sid: 1fav_1day + 1fav_video_* + video_2/4/**14**day + video_4to14day + nsfw_video_2/7/14day + nsfw_video_4to14day + evergreen_video_5y + imagine_4day (15 windows) · min-age gating w/ prefloor sidecars · tail: authors < 1000 followers, min fav 0 · training EVERGREEN corpus now `video_4to14day` (was `evergreen_video_1825day`) · analysis: 7-day retention · commit_after_dump, versions_to_keep 3, dump every 180 s

**grox:** reply-spam buckets ≤100 / ≤500 / ≤1000 · spam-detection follower threshold **1000** · **blast-radius thresholds 120,000** (Sept; stepped 30K→40K→60K→80K→100K→120K across Aug 17–25 commits — `FOLLOWER_COUNT_THRESHOLD_FOR_SPAM_DETECTION` + `FOLLOWER_COUNT_THRESHOLD_FOR_REPLY_RANKING`, task_filter.py:17,185) · upa deluxe fav threshold 64 · Sept: reply scores only move **down**, per-post dedupe removed, LLMs see follower counts, PTOS safety overhaul (adult-Hard needs dual-model agreement, ChildSafety fail-closed, ViolentMedia fail-soft, high-fav video lane gated on motion-reveal, off by default)

**New-account filter:** MaxAccountAgeSecs 1800 (30 min) · resurrection 1800 · metric fav (default) · OON-only

**SimClusters:** max 800 results · MIN_SCORE 0.5 · MODEL_20M_145K_2020 · query LOG_FAV_LONGEST_L2_EMBEDDING_TWEET · candidate LOG_FAV_BASED_TWEET · ANN scan 50 clusters · 2M-entry cache/600 s

**Dwell-regret:** temperature 10.0 · dwell floor 1.0 · gate bias 1.033918 · threshold −0.634264 · hysteresis 0.0 · WINDOW 28 days

**Post types (verified absence):** no article constants exist — no ARTICLE bit in tweet-type metrics (tweet_type_metrics_hydrator.rs:22-132), no article retrieval window (rankall), no article feature in the model (recsys.proto, feature_config.py), no article VF rule, no article weight. Thunder caps classify posts as replies (30) / originals (50) / videos (100) only — articles fall in the original bucket (Section 1.F).

### ❌ STILL UNKNOWN (redacted or runtime)

| Constant | Function | Status |
|---|---|---|
| Production FS overrides | Actual live weights | Read from `/usr/local/config/features/home-mixer/main/rust_home_mixer.yml` (config.rs:1) — code defaults are "synced" to prod by cron (README.md:404) but not guaranteed |
| NewUserAgeThresholdSecs prod value | New-user OON lockdown | Default 0; prod likely > 0 |
| VM ranker head-value model | The actual "value model" | **Redacted** — server reads no scores; DPP only |
| bdsm operating thresholds | Inauthentic detection | Sentinel 9.99 in code |
| Grox LLM prompts | Spam/quality classification | j2 files not published (README.md:418) |
| Some botmaker rules | Labeling | Not published |
| Phoenix training data windows/cadence | Continuous retraining | Training code open (Kafka-streaming feeds), but no production data window/cadence constants |
| Follower-count threshold for reply-ranking (grox) | Reply ranking gate | **120,000** (task_filter.py:185; stepped up from 30K across Aug 17–25); the spam-screening gate uses the same 120K (task_filter.py:17) |
| `MultiplierPreOffset` prod value (Sept) | Replacement for removed MPN mode | Ships false; prod unknown |
| New bool-feature indices (`IS_AUTHOR_FOLLOWED_BY_VIEWER`, `IS_AUTHOR_FOLLOWING_VIEWER`, `IS_STALE_POST14D`) | Model inputs | **Shipped since Sept** (`feature_config.py:36-44,74`, incl. `STALE_POST_14D_TTL_SEC = 1,213,200`) |
| `enable_stale_post` prod value | 14-day stale-post zeroing | Ships **True in `home_direct_packed`** (xrecsys.py:272) since Sept; grpc feed forces it off |

---

## 10. 🕒 Engagement History Windows: Account Recovery Timeline (updated)

### Known windows (all now open)

| Window | Value | Side | Source |
|---|---|---|---|
| **UAS aggregation window** | **300,000 ms = 5 minutes** | Viewer | params/config.rs:34 |
| Max sequence length (scoring/retrieval) | **1024** | Viewer | param.rs (MaxSeqLengthScoring/Retrieval) |
| Model history capacity (ranking) | **1022** (seqpack) | Viewer | xrecsys.py |
| Model history capacity (retrieval) | **1023** | Viewer | xrecsys_two_tower.py |
| Post age max (model) | 4800 min (~80h) | Post | recsys_model.py:93 |
| Post age max (feed) | **48h** | Post | params/config.rs:36 |
| Thunder retention | 48h | Post | thunder/args.rs:48 |
| Cold-start post age | 86,400 s (24h) | Post | author_cold_start.rs |
| Engagement signals (per type) | max 15, recency-sorted | Viewer | param.rs |
| VF safety-label cache | 30 s (fresh posts) / 60 s | Post | visibility-filtering |
| user-cred engagement window | **7 days** | Account | UserCredV2Config.scala:25 |
| Credit snapshot freshness | ≤ 7 days (warm start) | Account | UserCredV2App |
| Spam label store TTL | 26 h | Post | scarecrow SpamLabelStoreRegistry |
| **Stale-post flag** | **14 days** (`STALE_POST_14D_TTL_SEC = 1,213,200` — shipped in `feature_config.py:74` since Sept; `home_direct_packed` enables it, xrecsys.py:272) — zeroes engagement counts + sets flag | Post | xai-recsys/src/util.rs |

### Architecture change: UAS hydrator replaced

The old `user_action_seq_query_hydrator.rs` is **deleted**. It's now:
- **Scoring/retrieval sequences**: `scoring_sequence_query_hydrator.rs` / `retrieval_sequence_query_hydrator.rs` call the UAS aggregation service (`UserActionAggregationClient`) with **5-minute window**, `MaxSeqLength = 1024`, aggregation **`DENSE_WITH_LONG_DWELL`** for both scoring and retrieval (Sept: scoring moved first in the Aug 25 commit, retrieval followed by Sept 2; param.rs:128-138), source `NEW_UUA_ARROW`, separate `UaasModelType::{Ranking, Retrieval}`. The sequence is serialized to Arrow and sent to Phoenix.
- **Engagement signals** (for SimClusters source + new-user handling): `explicit_engagement_signals_query_hydrator.rs` (favorite/retweet/reply/bookmark/share/self-tweets from Twistly, 300 ms) and `implicit_engagement_signals_query_hydrator.rs` (photo-expand, VQV, immersive VQV; video views require **≥10 s** and non-promoted).

### Account status change implications (revised)

- **Viewer-side forgetting is fast**: the UAS window is **5 minutes** — the model's view of a viewer's recent behavior is a 5-minute rolling window (truncated to last 1024 actions). Long-horizon viewer features come from the user-features token (profile age/gender/country/apps) and the 28-day action-count features in the dwell-regret gate — **not** from the transformer history.
- **Author-side "reputation"**: still no decay in the model (Section 12.7); now the **user-cred PageRank** provides a daily-updated account score that gates enforcement, with a 7-day engagement window — i.e., credibility responds to engagement trends over ~7 days of favs/retweets.
- Recovery logic from the old section (recent actions matter most, window-based forgetting) still holds but the window is minutes, not days.

---

## 11. README Truthfulness Audit (Aug 13, 2026 README, + Aug 14 update)

The README was **rewritten** (482 lines, "Latest update — August 14th, 2026", README.md:30). It no longer makes the two most-contested claims of the old release (256-dim/2-layer model; "no hand-engineered features"). The **Aug 14** commit added a "Latest Updates" section (weights clarification + Brazil 2026 election filter announcement). **Sept:** the section was renamed "Notable Updates" and the Brazil line gained *"(Account list updated August 27, 2026.)"* — matching the 665→2,388 list growth in the filter file. Verified against code:

### ✅ VERIFIED TRUE

| # | README Claim | Evidence |
|---|-------------|----------|
| 1 | Two sources: Thunder (in-network, in-memory) + Phoenix retrieval & SimClusters (OON) (README.md:60-63, 99-104) | `thunder/` (48h in-memory store) + `simclusters_source.rs` + `phoenix_source.rs` ✓ (code also has 4 more internal sources: TweetMixer/Topics/MoE/CachedPosts — README simplifies to the 2 advertised families) |
| 2 | Request-path stages: query hydration → sources in parallel → hydration → pre-scoring filters → scoring → TopK → post-selection VF | Exact match to `phoenix_candidate_pipeline.rs` chain ✓ |
| 3 | "user action sequence — the main input to the model" (README.md:92-93) | scoring_sequence_query_hydrator.rs (UAS, 5-min window, Arrow) ✓ |
| 4 | Phoenix predicts per-action probabilities; list of heads (README.md:328-332) | 60 action heads incl. all listed (favorite…not dwelled) ✓ |
| 5 | "weights are in home-mixer/params/param.rs" (README.md:341) | **TRUE — this was the previous release's biggest omission; now shipped** ✓ |
| 6 | Three adjustments: author diversity (decay to floor), OON discount (factor < 1, incl. replies/reposts from followed accounts), new-author boost (impressions below threshold lifted to target position) (README.md:347-349) | ranking_scorer.rs (0.5^k, floor 0.25; 0.75; cold-start) ✓ |
| 7 | **VMRanker now documented**: "a separate service that reorders the result" / DPP over embeddings (README.md:276, 351) | vm-ranker/ open; DPP over v5 embeddings ✓ (README understates: DPP zeroes non-selected candidates) |
| 8 | Visibility filtering: ALLOW/INTERSTITIAL/DROP; recommendations-only rules that "can only drop"; same post allowed to a follower (README.md:203-206, 211-213, 393-394) | visibility-filtering/rules/registry.rs — exact ✓ |
| 9 | Filter tables (17 pre-scoring + 3 post-selection, in order) (README.md:358-376, 384-388) | Exact match to the **Aug 13** code order ✓ (AgeFilter 48h ✓) — **drifted Aug 14**: code now has 18 pre-scoring filters (Brazil2026ElectionFilter added; README not updated). **Sept:** README still lists 17 — and now also misses the `ViewerMutedKeyword` rename, the Following-pipeline additions, the multi_risk blender, and every weight/structure change below |
| 10 | Labeling path: content understanding (grox, media-model-proxy, clip, agatha, bdsm, user-cred-v2) → labeling rules (scarecrow/botmaker, abuse-enforcement-service, safety-label-user-agg) → storage → VF | All components exist ✓ |
| 11 | Hash-based embeddings, no vocabulary (README.md:460-462) | 2 hashes/entity ✓ |
| 12 | Candidate isolation in ranking (README.md:456-458) | `causal=False` + right-anchored RoPE, candidates can't attend to each other ✓ |
| 13 | Multi-action prediction (README.md:452-454) | ✓ |
| 14 | Thunder blacklists @grok/@gork/@products (not claimed in README, but in code) | thunder_service.rs:29-36 — a notable transparency point |
| 15 | "What's not in this repo": Grox prompts, some botmaker rules (README.md:412-427) | Honest disclosure ✓ |
| 16 | Under the Hood transparency tool + eligibility (README.md:438-442) | under-the-hood/ (365d age, 10 posts/mo) ✓ |
| 17 | "no pretrained checkpoint is included" (phoenix/TRAINING.md:90); synthetic data for training | ✓ — artifacts zip removed |

### ⚠️ PARTIAL / UNDERSTATED

| # | Claim | Reality |
|---|-------|---------|
| 1 | Architecture diagram shows only Thunder + Phoenix/SimClusters as sources | Pipeline actually runs 7 sources (Thunder, TweetMixer, Simclusters, Phoenix, PhoenixTopics, PhoenixMOE, CachedPosts) — README deliberately presents the two primary families |
| 2 | VMRanker "reorders the result" | Understated: DPP **removes** (scores 0) all non-selected candidates; the original value-model head inference is redacted — README doesn't disclose that the open server has no model |
| 3 | "aim is for experiments at ≥10% traffic to be visible in the repo" (README.md:402) | Aspirational; several major mechanisms ship at defaults that are off (`MultiplierPreOffset`, dwell-regret, new-user OON, cold-start Thompson sampling) with the real prod values living in the excluded FS YAML |
| 4 | Weights "in param.rs" | The values are defaults mirrored from prod via cron; live prod values are feature-switch driven (README discloses the sync mechanism ✓). **Aug 14**: the README + code now carry an explicit clarification that weights scale predicted probabilities, not raw counts (Section 1.B) ✓ |
| 5 | "replaces the Phoenix demonstration model with the code used to train the models the feed uses" (README.md:43) | True in structure, but the repo cannot reproduce production models: no pretrained checkpoint, no production data pipeline (synthetic only) |
| 6 | Filter tables (17 pre-scoring + 3 post-selection, in order) (README.md:358-376, 384-388) | **Drifted by the Aug 14 commit**: the code now has **18** pre-scoring filters — `Brazil2026ElectionFilter` was added between `AuthorSocialgraphFilter` and `VideoFilter`, but the README table still lists 17 and the "Latest Updates" section doesn't mention the filter-table change (it does announce the filter itself). Same for the new model bool features and grox 30K thresholds: announced in code, absent from README's table. **Sept: the drift is now much larger** — the README table misses the Brazil count (665→2,388), the mute-filter rename/additions, the 0.07/0.0/0.05 weight moves, MPN→`MultiplierPreOffset`, the slimmed VMRanker request, LONG_DWELL aggregation, ColdStartTsTopK 2, `FOLLOWING_MAX_RESULT_SIZE` 110, the multi_risk blender, grox 120K gates, and the enabled stale-post/count features |

### ❌ FALSE OR MISLEADING — none found (major improvement over old release)

The two old false claims are gone:
- ~~"256-dim embeddings, 2-layer transformer"~~ → no model numbers in README; actual prod = 8 layers / 2560-dim (Section 5.1).
- ~~"eliminated every hand-engineered feature"~~ → claim removed entirely.

### 📊 TRUTHFULNESS VERDICT (Aug 2026)

**~85% accurate — still the most transparent release, but the README is now drifting.** The README's architecture, scoring description, and labeling path match the code (as of the Aug 13 commit, plus the Aug 27 Brazil note). Remaining gaps: (1) the 7-source reality vs 2-source diagram, (2) VMRanker's redacted head-model and zeroing behavior, (3) prod FS values not shipped (structural: the sync mechanism is disclosed), (4) **Sept drift**: the README's filter table still lists 17 filters while the code has 18 (+ Following additions + the mute-filter rename), the weights table misses the 0.07/0.0/0.05 moves, and nothing in the README reflects MPN→`MultiplierPreOffset`, the slimmed VMRanker request, LONG_DWELL aggregation, TsTopK 2, FOLLOWING_MAX 110, the multi_risk blender, grox 120K gates, or the enabled stale-post/count features. The Aug 14 commit's strongest transparency point is the explicit correction of a public misconception (weights ≠ count equivalences) in the code itself. Compared to the old release (model-size error + missing VMRanker + false "no heuristics" claim), this is a qualitatively different document.

---

## 12. Account Reputation & Reach: How Reputation Determines Visibility (major revision)

### 12.1 The headline finding (revised)

The July claim — "**there is no reputation score anywhere in this codebase**" — is **now false**. The Aug release ships a full account-health stack. But the nuance survives: **no reputation score feeds the For You *scoring* directly**. Reputation acts through four layers:

| Layer | System | Effect on reach |
|---|---|---|
| A. Learned author embeddings | Phoenix model (ranking + retrieval) | Implicit, per-author engagement propensity (Section 12.2 of old doc still applies; vocab now user/item 100M, author 30M, 2 hashes) |
| B. **Explicit credibility score** | **user-cred-v2** (PageRank mass) | Gates spam labeling, enforcement, challenge/suspend; NOT fed to feed scoring |
| C. Follower count + graph | gizmoduck, socialgraph, bidirectional follow | OON status, mutual-follow boost, VF rules, cold-start eligibility, model inputs |
| D. Labels (post + account) | grox, scarecrow, safety-label-user-agg, abuse-enforcement, media models → **VF rules** | Drop/interstitial with exact rule catalogue |

### 12.2 B (new): user-cred-v2 — the explicit credibility score

**`user-cred-v2/`** (Scalding daily batch, prod since 2026-04-27):

- **Algorithm**: PageRank-style mass propagation over the **follow graph** (edges keep their follower→followee orientation: each user distributes `mass/outDegree` to the accounts it follows, and accumulates mass from its followers — the `.swap` at `UserCredV2App.scala:295` is only a validation step, not an orientation flip), with **engagement-weighted teleport**: fav/retweet edges (7-day window) weight the teleport distribution, blended with a uniform distribution over **premium users** (blue/gray/gold verified, org, affiliate) at β = 0.5. Jump probability α = 0.2; convergence at L1 diff ≤ 0.001 (max 50 iterations).
- **Score**: `165.2 + 7.07·ln(mass)`, clamped to [0, 100]. Daily Manhattan snapshot.
- **Sanitization**: suspended/deactivated/erased/restricted users dropped; linked-account (expansion) edges scrubbed; near-zero users flagged; warm starts keep prior mass.
- **Gate**: ±5% checks on row count, mass sum [0.99, 1.005], VIT count (score ≥ 55) vs prior snapshot; failures quarantine.

**Consumers** (this repo):
- `abuse-enforcement-service`: `pagerank_skipped: cred.is_high || cred.score >= 50.0` → accounts with credibility ≥ 50 are **exempt from automated enforcement rules** (user rules yaml:26-31; post rules require additionally `!skip_author_credibility_prechecks`).
- `scarecrow`: `IsHighPageRankUser` = cred score ≥ **54** OR label `PREDICTED_HIGH_PAGE_RANK`; fallback followers > 25,000 when no score and no label. High-PR authors are skipped by spam labeling (e.g. bot 3226 URL-verdict SPAM labeling).
- VF user-label aggregation excludes high-PR authors from `POSSIBLY_NSFW_ACCOUNT`/`NSFW_HIGH_PRECISION` user labels (safety-label-user-agg).

**Reach interpretation**: credibility is a *shield* — it exempts accounts from spam labels, suspensions, and challenges. It does not directly boost feed position. Notably, **blue verification matters here**: premium users get extra teleport mass (making the cred graph lean toward verified accounts), so blue-check accounts are more likely to hold high cred → enforcement exemption. This is the *only* place verification enters the algorithm (Section 13.7's "verification has zero effect" finding was about feed delivery and still holds).

### 12.3 C (updated): follower count & graph signals

- Follower count is hydrated per candidate (gizmoduck) and used: (a) cold-start eligibility (≤1000), (b) SimClusters/rankall tail eligibility (authors < 1000 followers), (c) tweet-type metrics buckets (0-100/1K/10K/100K/1M) and the served-metrics bitset (RETWEET/REPLY/VIDEO/… — **no ARTICLE bit**), (d) model `UserContext` follower counts. **Sept:** what it is NOT used for grew — the VMRanker request no longer carries author follower counts at all, and on retweets the model no longer receives `is_following_user`/`followers` (candidate.rs). (e) grox's reply/PTOS LLMs now *see* follower counts in their prompt context (not a score — prompt text).
- **grox reply ecosystem**: three blast-radius gates in `grox/flows/reply_spam/task_filter.py` — basic LLM spam screening runs when the in-reply target AND thread root are both **≤ 120,000 followers** (**Sept**; stepped up from 30K — skipped above, reason `reply_ranking_target`, line 17/81-83); coordinated-spam detection runs only when root ≥ **1,000** followers (line 97/158); LLM reply-ranking runs only when target or root > **120,000** followers (line 185/216-220, reason `low_blast_radius`). So replies in small conversations are the ones screened; replies in big threads are ranked instead. **Sept behavior changes:** reply scores now only move **down** (a later higher re-score is dropped — the first/lowest score sticks, with low scores fanned out to regional caches faster); the per-post per-minute dedupe on spam/ranking annotations was **removed** (more LLM volume, faster re-scores); the reply pipeline re-sourced to the v3 unified-posts topic with spam-screening added to the recovery path.
- **Bidirectional follow (NEW, big)**: mutual-follow authors' original posts get reply weight +15. This is the strongest explicit graph-derived multiplier in the system, documented in `docs/BIDIRECTIONAL_BOOST_CHANGE.md` (A/B Jul 10; launch Jul 13 at 20; reduced to 15 Jul 24).
- **Block/mute both directions**: viewer↔author + quoted/retweeted authors (author_socialgraph_filter.rs:27-56) + VF `ViewerBlocksAuthorRule`/`ViewerMutesAuthorRule`/`MutedRetweetsRule`/`AuthorIsProtectedRule` (logged-out or non-follower).
- **New-user viewer lockdown**: viewers < NewUserAgeThresholdSecs with ≥5 followings get OON factor 0.00001 (if enabled) — new-account viewers see an almost purely in-network feed.

### 12.4 D (new): enforcement — the sharpest reputation lever

**`abuse-enforcement-service/`** (Rust): consumes Kafka model-score results (dedup classes: none/label/challenge/suspend) → fact gathering (gizmoduck, cred, grey badge, UAS) → **CEL rule pipeline** (rules mirrored from GrowthBook `xai_abuse_enforcement_service_rule_user/post`) → rate limits → AIS actions:

| Action | Effect |
|---|---|
| `suspendUser(perm:false, PlatformManipulation)` / `suspendUser(perm:true, Cse)` | Account suspended |
| `addLabelsV2` (SpamHighRecall, TTL 30d) | User label → VF drops for non-followers |
| `addPostLabelsV2` (RiskyHighVizReply, SpamHighRecall — 30d) | Post labels → VF drops |
| `bounceArkose` / `bounceCaptcha` / `spamLivenessCheck` | Challenge flows |
| caps | 500,000/day + 500,000 post/day; 24h dedup; 15-min skip dedup |

User rules: `anchor_campaign_suspend`, `llm_slop_user` → SpamHighRecall, `SpamEmbeddingMajorityPoster`, `inauthentic_detection_v45_*`, `cluster_spam*` (suspend / arkose / label), **`panda_reports_embedding_v10_rough_spam` → suspend (NEW Sept)**, terminal `act_suspend`. Post rules: `llm_slop_post` → RiskyHighVizReply, `gibberish_post`/`fast_reply_spam_post`/`anchor_campaign_post`/`SpamEmbeddingPtosDistilled` → SpamHighRecall, **`cse_reports_embedding_v2` → suspend author (perm:true)**. **Sept: both pipelines gained an `act_requested_actions` step** — scorers can now request specific actions per score (suspend/label/challenge) via a new generic-action executor (`generic_actions.rs`: per-entity allowlists, 16-action cap, TTL 0 = no expiry), but it is **inert until GrowthBook allowlists are configured** (absent = deny-all), and the cred/follower shields still win over it — plus a `decisions_json` Kafka mirror and an XReview human-review reporting path in scarecrow (`XReviewIntakeRegistry`, `CreateXReviewReportProd/Staging`).

**`bdsm/`** (inauthentic behavior): transformer over user action sequences, 8 heads: FollowBot, LikeBot, EngagementAmplifier, ReplySpamBot, TweetSpamBot, RTBot, MultiActionBot, **LegitimateUser**; graduated actioning (challenge → suspend); operating thresholds redacted (sentinel 9.99). The enforcement-note structure is visible — a note fires when the dominant bot head scores > 0.5 over a policy-defined action count, naming the head (e.g. `[model: FollowBot=0.91]`); **Sept: the count gate itself is now redacted** — `min_actions_for_enforcement` ships as `999999` (was 30; sink_policy.yaml:13, version 2026-08-14), so only the > 0.5 pick logic is verifiable and the real gate value is withheld like the probability thresholds. The note *templates* and per-head key-action interpolators are the redacted sentinel `"<redacted>"` (score_results_sink_focal.py).

**`agatha/`**: PMI-based user-label modeling (spam_suspended, rate_based, nsfw, reports_per_fav, blocks_per_fav, favs) over graph signals (follow/fav/block/mute/abuse-report/DM/RT/list).

### 12.5 D (new): visibility-filtering rules — the full catalogue (now open)

**Verdict model**: `Allow | Interstitial(FilteredReason) | Drop(FilteredReason)`. First drop ends evaluation; drop beats interstitial; interstitial beats allow. In-network = `TimelineHome` rules (28); OON adds `TimelineHomeRecommendations` rules (**26**) that can only drop. **Sept:** the rule files were consolidated (`tweet_rules.rs` 1158 lines + `author_rules.rs` 354 + `context.rs`/`rule_spec.rs`, old per-family files deleted) with **byte-identical behavior** — the registry wires the same 28 + 26 in the same order (`registry.rs:81-99`, test-pinned), now with a machine-readable `golden_corpus.rs` contract (in-network vs OON matrices, follower/self-view/opt-in exemptions, drop-beats-interstitial ordering). One real change: the 16 NSFW-gating countries are now **runtime-configurable** (defaults identical, refreshed from feature switches every 30 s with a Scala-file drift check — `visibility-filtering/params.rs`), not a hardcoded const. Also pinned: `EGREGIOUS_NSFW` and `RECOMMENDATIONS_BLACKLIST` have **no** VF rule (Allow-everywhere).

**Base rules (both surfaces):** suspended/deactivated/erased/offboarded author; protected author (logged-out or non-follower); viewer blocks/mutes author; muted retweets; tweet labels **PDNA / BOUNCE / SPAM / FOR_EMERGENCY_USE_ONLY / FOSNR level-3 (hateful conduct, violent speech, abuse, civic integrity)** → drop; nullcast tweets; superseded edits; legal takedowns (author-country matched; author-exempt); **NSFW age gating** (viewer age < 18 → drop even with opt-in; logged-out → drop; no-stated-age in 16 gating countries [ar, au, br, ca, de, es, fr, gb, id, it, kr, mx, nl, ph, pt, th] → drop); **exclusive (super-follow) content** → drop unless viewer subscribes; interstitials for NSFW_HIGH_PRECISION / GORE_AND_VIOLENCE_HIGH_PRECISION / NSFW_CARD_IMAGE / nsfw-flagged author (when not opted-in).

**OON-only drops:** DMCA media; geo-restricted media (`xx` = worldwide); NSFW user/admin author flags; tweet nsfw flags; NSFW_HIGH_RECALL / NSFW_HIGH_PRECISION (OON) / NSFW_CARD_IMAGE / GORE_AND_VIOLENCE (OON); **DO_NOT_AMPLIFY**; MALICIOUS_URL; SPAM_HIGH_RECALL; NSFW_TEXT; FOSNR abuse-insults; plus **11 user labels** (the table below):

| User label | Followers exempt? |
|---|---|
| NSFW_HIGH_RECALL, NSFW_HIGH_PRECISION, SPAM_HIGH_RECALL, COMPROMISED, READ_ONLY, IMPERSONATION_HIGH_PRECISION, NSFW_AVATAR_IMAGE, NSFW_BANNER_IMAGE, NSFW_NEAR_PERFECT | no (they are OON-only — allowed in-network — but OON they still drop followers) |
| **ABUSIVE_HIGH_RECALL** | **yes (logged-in followers)** |
| **DO_NOT_AMPLIFY** | **yes** |

Self-view exempt. Labels come from: tweet safety-label store (L1 in-process cache 1M entries, 30 s/60 s TTL → L2 twemcache `slm_{tweet_id}` → L3 Manhattan `jammer/safety_label_store/tweet_labels`), gizmoduck user labels, TES (takedowns/edits), socialgraph, exclusive-content store. Label provenance includes `GrokAnnotationAction{source}` and `BotmakerAction{rule_id}`.

**`safety-label-user-agg/`**: post-level labels aggregate to user labels — `POSSIBLY_NSFW_ACCOUNT` (windowSize 10, minimumMatchingPosts **11** — as configured the rule is unsatisfiable, since only 10 posts are ever examined; 60-day max post/trigger age, 7d TTL; excludes high-PR/grey) and `NSFW_HIGH_PRECISION` (≥3 of 5 posts) — written to gizmoduck by "Grox".

**`scarecrow/`**: event-driven BotMaker rules apply tweet labels in real time (e.g., bot 3226: bad URL verdict → SPAM/SPAM_HIGH_RECALL on author's tweet unless high-PR/gray), via AIS `AddLabelsV2` with `safetyLabelType`; spam-label store TTL 26 h; per-rule rate limits (token bucket; limits come from runtime config — shipped rules carry empty `actionLimits`); `HaltUntilHumanIntervention` mode; rules ≥ 100,000 are transpiled. **Sept: new `XReviewIntake` downstream + `CreateXReviewReportProd/Staging` units** — BotMaker rules can now file proactive human-review reports (post/profile), not just labels.

### 12.6 What is NOT part of reputation (verified)

- **Verification**: never read by feed scoring/ranking (gizmoduck hydrates no verification field). Only indirect: premium status feeds the user-cred teleport distribution (12.2) — affects enforcement exemption, not delivery. (Sept: the feed also now forwards an `nsfw_author_phoenix` model signal — verification still absent.)
- **Engagement-per-impression ratio**: not present; per-post counts hydrated and bucketed into the model (Sept: plus the log2-MLP count path in `home_direct_packed`).
- **A standalone "author quality" feature**: the model uses hashed author id only (plus SID, multimodal, counts) — though **Aug 14** added the two follow-direction relationship flags (`IS_AUTHOR_FOLLOWED_BY_VIEWER`, `IS_AUTHOR_FOLLOWING_VIEWER`) as model inputs (candidate + history), so "does the viewer know this author" is now part of the prediction itself, not just the +15 scoring boost.
- **bdsm/agatha/cred scores**: not inputs to Phoenix or RankingScorer.

### 12.7 Bottom line: how reputation is actually determined (revised)

1. **Explicit credibility (user-cred-v2)**: PageRank over follow + 7-day fav/retweet teleport; score 0-100 daily; premium-biased; exempts from enforcement ≥ 50, high-PR ≥ 54. This is the closest thing to a "reputation score" and it gates *removal*, not *promotion*.
2. **Labels**: spam/abuse/NSFW/FOSNR labels (LLM + models + rules) → VF drops; follower-exemptions for two labels; post-level labels aggregate to account-level.
3. **Graph**: follows (OON 0.75), mutual follows (+15 reply boost **and, since Aug 14, follow-direction model features**), blocks/mutes both directions, subscriptions, account age (new-user 0.00001), viewer age (VF NSFW gating).
4. **Learned author embeddings** (unchanged analysis): 30M-author hash vocab, collision-diluted for small accounts; the candidate vector remains ~1 part author + post/age/SID/multimodal parts.
5. **Enforcement**: model-score-driven rules can suspend, label (30d SpamHighRecall), challenge, or cap at 500K/day per surface.

### 12.8 The Learning Timescale (revised — training code is now open)

The old section's core finding (no training loop in the repo) is **superseded**: `phoenix/xrex/` ships the production trainer. What we now know:

- **Training**: dense params via **Muon** since Sept (H100 lr 7.1e-4, GB300 5e-4; prod uses a tuned RMS-normalized-Adam derivative per TRAINING.md:14-17 for other surfaces); **embeddings via sparse rowwise AdaGrad** (**lr 0.28**, half-life 2500 H100 / 5000 GB300, lazy decay + small weight decay). Total data budget 1e11 samples; checkpoints every 150/300 steps with copy-port distribution (9988) to serving. (Plus a new StableHLO bundle-export path for rollout artifacts — no cadence change.)
- **Feedback loop confirmed**: home-mixer side effects publish every prediction (`PHOENIX_SCORES_TOPIC`), 5%-sampled reranking scores (`RERANKING_TOPIC`), client events, served candidates — each engagement outcome is a training example for the next checkpoint.
- **Still unknown**: production data windows, retrain/rollout cadence, learning-rate schedules in prod. The model has **no author-side time dimension** (unchanged): author embedding freshness = latest checkpoint. Post age (≤80h buckets) is the only *post-time* feature (timezone/hour-of-day features exist but are viewer-local context, not author-side) — plus `IS_STALE_POST14D`, which zeroes engagement counts for posts > ~14 days old (**now enabled in `home_direct_packed`**).
- **Quicksand verdict (updated)**: unchanged structurally (collision-diluted embeddings, post-heavy candidate representation, viewer-side 5-min UAS forgetting, content-based VF drops) — plus two *recovery-friendly* mechanisms: the **cold-start boost** actively promotes low-impression ≤1K-follower authors' posts toward slot 15, and **credibility is re-computed daily from a 7-day engagement window**, so enforcement shields can be earned in ~1 week and lost if behavior lapses. The Sept training changes strengthen this further: lazy decay + weight decay on the embedding optimizer (rows now shrink toward zero, not just decay history) — the author-embedding quicksand still depends on hidden retraining cadence.

---

## 13. Empirical Validation: 100-Profile Sample (`x-filter-profile-stats-sample-1.json`)

### 13.1-13.2 Data, methodology, baseline — unchanged

Baseline: median V/F 9.49% (Q1 3.81%, Q3 18.46%); median likes/view 1.22%. (Sample collected pre-Aug-2026 release; the scoring architecture these numbers exercise is the same RankingScorer family, now with exact weights.)

### 13.3 Finding 1: follower count negatively correlated with per-follower delivery — mechanism now quantified

The code now gives exact numbers for the structural small-account advantages:

- **Cold-start boost** (author_cold_start.rs): accounts ≤ **1,000 followers** with posts < **1,000 Home-Timeline views** (Sept) ranked within the top 85% of the scored pool get a post lifted to **slot 15** — a concrete, per-request promotion for small accounts. (Aug 14 added a Thompson-sampling picker variant, off by default, top-2 since Sept — Section 5.3.)
- **post_unexplored** weight (0.02, in-network only) — extra credit for posts never seen.
- **Hash-collision dilution** (unchanged): 30M/100M author vocab, 2 hashes — small accounts average with collision partners ("average author" rather than "bad author").
- **New-user OON lockdown** (0.00001) only applies to *viewers*, not authors.
- Follower count is never a Phoenix model input (Sept: the old `author_followers_count` → VMRanker request path is **gone** — the request carries no follower data at all) — though the **Aug 14** commit added the *relationship* flags `IS_AUTHOR_FOLLOWED_BY_VIEWER` / `IS_AUTHOR_FOLLOWING_VIEWER` as model inputs (Section 5.1; enabled in `home_direct_packed` since Sept): count, no; relationship, yes. (Sept caveat: on retweets the model now receives **neither** flag nor any follower count.)

So the −0.404 correlation and 22.86% (100-1K bucket — a single account) vs 3.65% (1M+) delivery gap are consistent with: no follower feature in the model + cold-start promotion for ≤1K + follower-base quality effects.

### 13.4 Finding 2: the delivery ceiling — mechanism now identifiable

dawn_com (1.38M followers, medV/F 0.065%, best post 0.52%) is **structurally ineligible** for every small-account mechanism:

- Cold-start requires ≤ **1,000 followers** — 1.38M is 1,380× over the cap.
- post_unexplored is in-network-only and tiny (0.02).
- The learned author embedding for a 1.38M-follower account is essentially "personal" (few collision partners) → low engagement propensity embeds permanently in its own bucket.
- R/V ~0.007% (near-zero replies) starves the highest positive weight in the system (reply 5.0, vs like 0.5).

The ceiling is the *absence of every countervailing mechanism*: the author embedding is a persistent negative and nothing in the pipeline (except a single VF-free, DPP-surviving viral post) can override it. The one-escape observation (drdrew's 1.21M-view post) matches the design: a strong *post-level* prediction (post embedding + age + SID + multimodal) can outscore the author contribution because the candidate vector is post-heavy.

### 13.5 Finding 3: format effects — the ~48× dawn_com vs zerohedge gap

Unchanged interpretation; now with weight evidence: reply weight 5.0 (vs like 0.5) quantifies why R/V discriminates (~0.007% vs 0.027% vs 0.13%); dwell_time (0.004, capped ~0.12) modestly favors the article posts in the sample (collector flag `IsArticle`, bit 2 — median V/F 25.95% vs 9.65% baseline; n=37, 23 from one account; "longer dwell" is an inference, not a measured signal — see Section 1.F) while link posts' low dwell/click-through starve `cont_dwell_time` and `open_link` (0.2) and `share_via_copy_link` (20.0) requires *sharing* — rare. VQV gate confirmed at 10 s. **Sept:** the VQV *scoring* weight is now 0.0 and the dwell *probability* head is 0.05 — "watch time" now earns only the (unchanged) continuous dwell term plus the new dwell-prob term, not VQV.

### 13.6 Finding 4: engagement rates don't explain delivery — unchanged, weight-reinforced

mmpadellan (L/V 6.09%, RT/V 1.25%, medV/F 3.78%) vs nikitabier (0.76%, 0.05%, 81.64%): the code confirms aggregate counts enter the model only as **bucketed post features** and the author embedding is trained on *unobserved* per-viewer outcomes. The observable-rate inversion remains unexplained by any shipped feature — the causal state (per-viewer propensity, dwell patterns, velocity) is still unobservable in aggregate data.

### 13.7 Finding 5: verification has zero effect — still true for delivery, with a nuance

Unverified accounts outperform verified in every bucket from 1K to 1M (9.79% vs 18.76% at 1K-10K; 13.57% vs 17.11% at 10K-100K; 7.24% vs 10.08% at 100K-1M) — with one caveat: the only 1M+ unverified account (rt_com, 1.09%) trails the verified median (4.57%), so "every bucket" is three of four and the exception is a sample of one. The feed path still never reads `is_blue_verified` (gizmoduck hydrates no such field; model input has no verification feature). **Nuance**: user-cred-v2's teleport distribution is uniform over *premium* users (blue/gray/gold/org) — so verification influences *enforcement exemption* (cred score, high-PR ≥ 54), which is a removal-shield, not a delivery boost. The empirical delivery finding stands.

### 13.8 Finding 6: virality is OON distribution — updated

The OON layer now has three arms: **Phoenix retrieval** (two-tower over SID + embeddings; index windows per engagement threshold — 1fav_1day … evergreen_video_5yr, plus Sept's age-banded 1fav_video_* slices, rankall), **SimClusters** (min score 0.5, 800 max — cluster-similarity reach), and **PhoenixTopics/MoE**. The 50-2000×-followers viral tails match any of these. Note the **rankall windows**: a post needs ≥1 fav to enter the 1-day index and ≥32 favs for 32fav_1day — a minimum engagement threshold for OON candidacy (posts below 1 fav live only in post_creation_1day/video windows — and since Sept, fav processing emits 1fav unconditionally, closing the 0-fav race gap).

### 13.9 Finding 7: author-diversity decay — values now known

`AuthorDiversityDecay = 0.5`, `Floor = 0.25`, k-th post multiplier `(1−0.25)·0.5^k + 0.25` (2nd post ×0.625, 3rd ×0.4375). Still per-request, per-viewer slot allocation — **not observable in aggregate view counts**; unchanged conclusion. New related mechanisms also per-viewer/invisible in this dataset: DPP diversity (per-request embeddings), cold-start slot lift, bidirectional boost.

### 13.10-13.11 Observation → mechanism map (updated) + caveats — otherwise unchanged

New map entries:
- Small-account delivery advantage → **cold-start boost (≤1K followers, <1K Home-Timeline views → slot 15)** + collision dilution + no follower feature in model
- dawn_com ceiling → ineligible for cold start (1.38M followers); personal (undiluted) negative embedding; near-zero reply signal vs reply weight 5.0
- Articles best format → **no type signal exists** (Section 1.F): the sample's article posts over-deliver (25.95% vs 9.65% V/F) but that's 37 posts, 23 from one small account — plausibly cold-start + content dynamics, not a format boost; dwell bonus (0.004, capped ~0.12) + no VQV gate applies to all text-like posts
- Verification neutrality → no verification feature anywhere in feed path; premium only shapes cred teleport (enforcement shield)
- Viral tails → rankall index thresholds (≥1 fav 1-day; ≥32 fav; video windows to 5y) + SimClusters + Phoenix retrieval

Caveats: dataset unchanged; all timing/positional effects in 13.9 remain age artifacts; causality caveat in 13.6 stands.

---

*This analysis is based on the Aug 13, 2026 open-source release (commit a389166) plus the Aug 14, 2026 follow-up (commit c65aa17) and the Sept 2, 2026 state (commit 85ac72a, 13 commits Aug 17–Sept 2: weight moves to video_open 0.07 / vqv 0.0 / dwell-prob 0.05, MPN removal, slimmed VMRanker request, Home-Timeline cold-start views, TsTopK 2, FOLLOWING_MAX 110, LONG_DWELL aggregation, multi_risk ads blender, grox gates 30K→120K, Brazil list 2,388, rankall 17/15 windows, Muon training with enabled stale-post/follow/count features, VF refactor, panda-reports rule + generic actions, redacted bdsm count gate). Unlike the July release, the scoring weights, feed sizes, age limits, history windows, and visibility-filter rules are now open; the remaining unknowns are production feature-switch values (rust_home_mixer.yml), the VMRanker head-value model (redacted), bdsm thresholds **including the count gate**, and the Phoenix training data windows/cadence.*

<!-- REVIEW-BASELINE-FOOTER rev=85ac72a1bba41f21615e3f0bca56da75970a6633 short=85ac72a date=2026-09-02T19:46:48Z -->
