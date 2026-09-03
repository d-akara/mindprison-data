# Why Some Accounts Are Seen and Others Aren't

<!-- REVIEW-BASELINE repo=https://github.com/xai-org/x-algorithm.git rev=85ac72a1bba41f21615e3f0bca56da75970a6633 short=85ac72a date=2026-09-02T19:46:48Z prev=c65aa179db7bdd61e2c2821eac87f208a105c053 range_commits=13 -->

### Explaining the anomalies in "A Mostly Incorrect Analysis of X Social Media Algorithms and Reach" using the now-open X algorithm code

---

## What this document is

In August 2026, xAI published a much larger slice of the X recommendation algorithm than ever before (the "For You" feed: scoring weights, the visibility-filtering rules, the credibility system, the model training code, and more). Shortly before that, Dakara (Mind Prison) published [a data-driven article](https://www.mindprison.cc/p/mostly-incorrect-x-algorithm-reach-analysis) asking why some accounts — including several very large ones — get almost no reach, and why engagement seems to have little to do with it. On August 14th, 2026 a follow-up commit (`c65aa17`) added more: an official clarification of how the weights work, a Brazil 2026 election filter, new model input features, and threshold changes. Thirteen more commits through September 2nd, 2026 (`85ac72a`) moved weights and gates again (grox 30K→120K, VQV weight→0, dwell-prob head→0.05, MPN removal, Brazil 665→2,388, rankall 17/15 windows, Muon training, VF refactor, redacted bdsm count gate). Everything here reflects the code as of that commit; Part 24 summarizes the August updates, and **Part 25 the September ones**, with the affected earlier parts updated in place.

This document attempts to explain the article's anomalies with what the open code actually does. It is written for readers who do not want to read Rust source code. Where the article made specific numerical claims, I checked them against the [raw datasets](https://github.com/d-akara/mindprison-data/tree/main/x-algo-analysis) before using them; every number quoted here that comes from the data was independently reproduced.

**Where this fits:** the companion to the primary, code-citing reference [algo-analysis.md](algo-analysis.md). This document is the scenario addendum — it answers questions ("what happens if I reply to small accounts?", "can my likes hurt me?", "is my account recoverable?") with the same code. For the non-technical distilled playbook, see [user-guide.md](user-guide.md).

One honest caveat up front: the code can explain *mechanisms* — the forces that push accounts toward one outcome or another. It cannot tell us the actual production values of every setting (some are redacted, some are read from internal config files), and it certainly cannot tell us about *you* specifically. What follows is the best mechanical explanation the code supports, plus a clear statement of what it can't explain.

---

## Part 0 — A quick primer: how reach actually works

Five ideas make almost every anomaly in the article intelligible. The rest of this document is just these five ideas applied to different accounts.

**1. Every view is a separate decision.** Nothing on X is "broadcast" to your followers. When anyone opens the For You feed, the system assembles a batch of candidate posts — the posts competing for a place in your feed, about a thousand or more — and for each one asks the model a separate question: *"How likely is THIS specific person to like, reply to, re-post, or spend time on THIS specific post?"* Each viewer gets their own scores. Your "reach" is the sum of thousands of individual yes/no decisions, one per person.

**2. Follower count is never used in scoring.** The ranking model does not receive your follower count as an input. A follower list is treated as a list of people, not as an audience. Each follower is individually predicted and individually shown (or not shown) the post. This is the single most counterintuitive fact in this document, and it explains more than half of the article's anomalies.

**3. Every account carries a learned reputation that entrenches itself.** The model doesn't just look at the text of a post. Every account has a learned "impression" — a vector of numbers trained on millions of past outcomes — that roughly encodes *what happens when this account's posts are shown to people*. Accounts whose posts generate engagement get an impression that makes the model predict engagement for their future posts, which means they get shown more, which generates more engagement data, which reinforces the impression. Accounts whose posts don't generate engagement get the mirror image: shown less, less data, impression unchanged. This feedback loop is the engine behind most of the "unfairness" in the article. It is not a conspiracy; it is a self-fulfilling prediction.

**4. Every feed is a competition for a handful of seats.** A feed request pulls candidate posts from several sources (posts from people you follow; posts found by the discovery models; similar-content clusters; topic feeds; a cache), then runs them through a series of filters (too old, already seen, blocked author, muted keyword, and so on), then scores them, then applies a diversity step (a stage that removes posts too similar to ones already chosen) that can zero out near-duplicates, then a content-safety gate, and finally keeps roughly **35–47 slots** (the rest filled by ads and "Who to Follow" boxes). Most candidates lose every single request. Your post competes against the whole feed every time.

**5. The engagement numbers you see are symptoms, not causes.** The system shows your post mostly to the people it predicts will engage with it. So the engagement rate you see is measured *on the already-chosen audience*. A low engagement rate on your posts is partly the cause of low reach — but the two feed each other, and the direction of causality cannot be read from public numbers. This is why two accounts can have identical public engagement rates and wildly different reach.

---

## Part 1 — The invisible giants

> *"Dawn_com is the absolute worst large account I've found. At 1.4M followers, this account has a zero-engagement rate of 20%… JLMarchese111 has 122K followers with about a 70% zero-engagement rate!"*

**Verified.** dawn_com: 1.38M followers, median **~900 views per post** (0.07% of followers), 20% of posts with literally zero likes/replies/reposts/quotes/bookmarks, median engagement of 2 per post. JLMarchese111: 122K followers, median **~96 views per post** (0.08%), **68% zero-engagement rate**, median engagement of 0. Dr. Drew (2.5M followers, 0.4% delivery), Kirsten Gillibrand (1.3M followers, 0.5%), TechCrunch (10.6M, 0.5%), Entrepreneur (3.5M, 0.3%), digg (1.1M, 0.19%) are all in the same family.

**What's happening.** These are the trap from Idea 3, in its most extreme form. For each of these accounts the model has learned, from years of outcomes, that showing this account's posts to people produces almost nothing: no likes, no replies, no dwell time worth having. So per-viewer predictions stay low, the posts get shown to almost nobody, and the small number of views they do get generate little engagement, which confirms the impression. The account is not "punished" by any rule — no visible code flags these accounts. The system is simply, consistently *uninterested*.

Three details make the trap airtight:

- **Their size works against them.** The one deliberate small-account boost in the code — a "cold start" mechanism that lifts a low-performing post toward the top of the feed — only applies to accounts with **fewer than 1,000 followers** and posts with fewer than 1,000 **Home-Timeline** views (Sept). A 1.4M-follower account is 1,400× over the cap. The big invisible accounts have no counterweight at all.
- **Their audiences are full of people the model predicts won't engage.** A follower list accumulated over years contains inactive accounts, bots, and people who followed once and never engage. Since every follower is individually predicted, a large but passive list is worth less than a small engaged one. (This is why a 12,000-follower senator can out-deliver a 12-million-follower one — Part 3.)
- **The "zero engagement" rate is the signature.** When a post does get its ~900 views, those views are the people the model considered *most likely* to engage. For dawn_com, a 100% link-posting news account, even those people don't engage: headline links get a quick click-away, almost no time spent, almost no replies (their reply rate is ~0.007% of views — 8 replies on 4 posts across the whole 100-post sample). When even your best-audience converts at zero, the loop is complete.

**Is this "shadow-banning"?** Not in the administrative sense. A true suppression would come from labels or content-safety drops, which are binary and visible in the code's rulebook; these accounts show none of that — they still get thousands of views. This is *emergent* invisibility: no flag, no rule, just a learned equilibrium. To the person running the account the effect is identical to a shadow-ban, which is exactly why the article is worth taking seriously. But the fix is different: a flag can be appealed; an equilibrium has to be changed through outcomes (see Part 11).

---

## Part 2 — Same size, 7,000 times apart

> *"Dawn_com is essentially the same size as Nikita's account [nikitabier], but at 7,000 times less engagement."*

**Verified — the article is almost exactly right.** nikitabier: 1.25M followers, median **1.02 million views per post** (81.6% of followers), median engagement **13,894** per post, median likes ~10,400. dawn_com: 1.38M followers, median 900 views, median engagement **2**, median likes 1. The ratio is **6,947×** — the median-*engagement* ratio (13,893.5 / 2); the median-views ratio is ~1,130×.

**What's happening.** Two accounts of identical size, one a stone in a river, one carried by it. The difference is not the accounts' sizes but what the model has learned about what *happens when their posts are shown*. Nikita's posts generate replies, shares, and long dwell — and the scoring weights now published in the code show exactly which behaviors the system pays for. A **reply carries 10× a like's weight** (5.0 vs 0.5) — weight, not count-equivalence: every term is `weight × predicted probability`, and the large weights on rare actions (DM share 5, copy-link share 20) are inflated partly because those actions are rare, the same logic as report −234 (the platform's own clarification, and the VP of Product's Aug 15, 2026 post). A share via DM is 5, a share via copy-link is 20, a follow-after-seeing is worth 4, and every second of dwell time adds a small continuous bonus — each scaled by how likely the model thinks *this viewer* is to do it. Dawn's headline links generate none of these — their reply rate per view is ~4× lower than zerohedge's (0.007% vs 0.027%) and ~18× lower than Nikita's (0.007% vs 0.13%).

So the machine isn't unfair in a mysterious way; it is executing a very specific definition of value: *posts that make people reply, share, follow, and stay are worth showing*. One account triggers that definition every time; the other triggers it never. Both learned their states the same way — from their own history. That is also why the article's observation that dawn_com has a *flat* 0.07% delivery (no spikes, nothing) while Nikita's posts almost all do well is meaningful: flatness is the signature of an account-level state, not per-post luck. Per-post luck would produce occasional hits; an entrenched negative impression produces none.

---

## Part 3 — The small accounts that crush giants

> *Fetterman: 54% views/follower, "nobody else in this sample is close." Bernie Sanders at ~12M followers: 1.8%. Pete Ricketts, the smallest account: 15%. OrwellDay (20K followers) outperforming Alex Jones (4.5M).*

**Verified.** Fetterman (538K followers): **54.4%** of followers per post, median 293K views. Sanders (11.6M): 1.78%. Gillibrand (1.3M): 0.53%. Ricketts (12.1K): 15.5%. OrwellDay (20.3K): 37.4% vs Alex Jones (4.47M): 3.1%.

**What's happening.** This is Idea 2 in action. The model never asks "how big is this account?" It asks, per person, "would *this person* engage?" Fetterman's 538K followers are overwhelmingly real, active, and engaged with his specific style — so the model shows his posts to a huge fraction of them, every time. Sanders' 11.6M followers contain a long tail of dormant, casual, and inflated accounts — the model predicts little engagement from most of them, so most never see his posts. The 54% vs 1.8% gap is not bias; it is the difference between an audience and a list.

Two further mechanisms help small accounts specifically:

- **The cold-start boost.** The code contains an explicit small-account favoritism: for accounts with **fewer than 1,000 followers**, if a post is less than 24 hours old and has fewer than 1,000 **Home-Timeline** views (Sept: `view_count_on_home`), the system will take one such post per request and **promote it toward the top of the feed** (its score is lifted to the level of a slot-15/16 post). This is the only deliberate "help the little guy" mechanism in the entire published pipeline, and it exists precisely because the model would otherwise ignore small accounts entirely (not enough data to learn anything about them). Two precisions from the code: the ≤24-hour age check applies in the experiment's *treatment* arm (the default holdout arm applies the follower/view/type conditions without the age gate), and the promoted post must already rank within the top 85% of the scored pool — the boost is "lift the best eligible candidate," not "rescue the worst" (author_cold_start.rs). The August 14th update added a **Thompson-sampling variant** (`EnableColdStartThompsonSampling`, ships off): instead of always picking the highest-scoring eligible post, it draws a Beta(0.75 + favs, 49.25 + non-favs)-distributed reward per eligible candidate, keeps the **top-2** by sampled reward (**Sept**: `ColdStartTsTopK` 5→2), and promotes the highest-scoring one of those — i.e., among candidates with near-zero engagement data, the *least-proven* one gets a shot (author_cold_start.rs, param.rs).
- **Being unknown is treated as "average," not "bad."** Because of how the model stores account information (a hash-based lookup shared among many accounts), a small account with little history is effectively judged as *an average author* — its posts are evaluated mostly on their own text, format, and appeal, not on a deeply-learned negative reputation. The worst fate in this system is not obscurity; it's a *learned* negative reputation (Part 1). Small accounts are mostly spared that.

**The >100% accounts.** celestepoasts (2,871 followers) has a median of 110% of its follower count in views per post; atmoio (72K followers) has a median of 105%. This is mathematically only possible if the system is showing their posts to **people who don't follow them** — the out-of-network discovery layer (retrieval models, similar-content clusters, topic feeds). The algorithm reaches beyond your followers when the model predicts outsiders would like your content. A small account with a good learned reputation gets a double engine: decent follower delivery plus an out-of-network tail.

**Where the initial views come from — the follower seed.** The code never restricts a new post to followers, but the first phase of distribution is *effectively* follower-only, and it is structural, not accidental. The in-network channel (Thunder) enters your post into every follower's candidate pool instantly and unconditionally — in-memory, 48-hour window, no engagement threshold, no index membership (whether it's *shown* to a given follower is still decided by that follower's per-viewer prediction, filters, and ranking — "unconditional" means unconditional *candidacy*). Every out-of-network channel is gated: the main discovery index admits posts only after they earn **≥ 1 favorite** (the training pipeline's home retrieval corpus is literally the 1-fav pool — `1fav_1day`), the 0-fav fresh window (`post_creation_1day`) arrives with a ~2-minute delay and carries only content-based predictions, and OON candidates score with the 0.75 discount on top. So the typical lifecycle is: **followers see it first; their favorites expand who sees it (the first favorite opens the base OON pool, the 32nd opens the hot pool, and every favorite in between raises the live engagement-count features — now through both the buckets and a dedicated log2-MLP path in `home_direct_packed` — that improve predictions for the next request — the momentum loop; the exact index-trigger cadence is maintenance plumbing, not audience mechanics; since Sept, fav processing emits `1fav` unconditionally, closing the 0-fav race gap); discovery then expands — or doesn't, if the seed audience doesn't engage.** dawn_com's 0.07% delivery is follower-only with zero discovery lift; the >100% accounts above are the mirror. A post that never earns its first favorite from its seed audience never leaves the seed — which is the mechanical reason your followers are the *entry ticket*, not the audience (algo-analysis §1.G). The seed size itself scales with followers — more followers means a larger raw pool and faster gate-crossings (more early favorites → sooner discovery expansion) — but the advantage is *velocity*, never per-viewer delivery probability: the model reads no follower count, so a follower's chance of being shown the post is the same for a 100-follower account and a 100-million-follower one. (One seed-stage asymmetry: ≥10-second videos enter their out-of-network pools at creation with zero favorites; text and photos need at least one.)

---

## Part 4 — Why engagement rate doesn't predict reach

> *"Senator Warnock has an engagement rate of 10%, also the highest in the sample, but with a view rate that is 1/5 of Fetterman's."*

**Verified.** Warnock: 10.0% engagement rate but 9.5% delivery; Fetterman: 6.7% engagement rate but 54.4% delivery. The same inversion exists in the first sample: mmpadellan (8.1% engagement rate, 3.8% delivery) vs nikitabier (1.3% engagement rate, 81.6% delivery).

**What's happening.** This is Idea 5, and it's the most misunderstood part of the whole system. The engagement rate you can see — likes divided by views — is measured *after* the system already decided who to show the post to. If the model shows a post only to its most devoted fans, the engagement rate will look excellent while the reach is tiny (Warnock). If the model shows a post to everyone, the engagement rate gets diluted by millions of indifferent views while the reach is enormous (Fetterman, nikitabier).

The model predicts things the public totals don't include:

- **Predicted dwell time** — will this person spend 20 seconds or 2 seconds on this post? (The scoring pays 0.004 per second of predicted dwell — capped at ~30 seconds, so worth at most ~0.12 points — but it is a real, continuous signal that applies to every viewer.)
- **Per-person reply and share probability** — not the aggregate rate, but the individual prediction for each viewer.
- **Click-through and follow-through** — does the post pull people to the profile, and do they follow?
- **Generalization** — "people similar to those who engaged with this author historically also engage with similar content" — the invisible cross-user patterns.

So when the article asks how the platform "cannot find a single person interested in your content," the mechanical answer is: it did find the people it predicts are most interested — that's who your views come from. Whether that prediction is a good one is a separate question, and the aggregate data can't answer it.

**Why your own account *feels* shadow-banned when it isn't.** Dakara's own account, Mind Prison (1,244 followers), measures 9.2% views/follower and 3.5% engagement — right at the sample median, arguably above it. Yet it feels broken. Three honest reasons:

1. **The For You feed is a skewed window.** It shows you accounts with good reach — that's what "For You" means — so your reference class is survivorship-biased. You compare yourself to the accounts the algorithm picked to show you, not to the median account.
2. **Hit concentration** — the share of an account's engagement that comes from its few biggest posts. Of the last 100 posts, one did well. With a median of ~114 views per post, most posts sit near zero, and the distribution is heavily skewed: most posts underperform, a rare few overperform. That shape is the same shape as everyone's, just at different scales — a 240M-follower account has the same pattern, with different numbers.
3. **Per-request competition.** Each of your posts enters a fresh tournament against a thousand candidates and loses most of them. Nothing about you individually is being singled out; that's the baseline experience of the vast majority of accounts.

---

## Part 5 — Blue checks don't matter (for delivery)

> *"There doesn't seem to be any evidence of a significant penalty or even any penalty at all [for non-premium accounts]."*

**Verified, twice.** In the data: unverified accounts *outperform* verified accounts in every follower bucket from 1K to 1M (9.79% vs 18.76% at 1K–10K; 13.57% vs 17.11% at 10K–100K; 7.24% vs 10.08% at 100K–1M) — with one caveat: the single unverified account in the 1M+ bucket (rt_com, 1.09%) trails the verified median (4.57%), so "every bucket" is three of four, and the exception is a sample of one. In the code: the feed path **never reads verification status**. The account-information fetch that supplies the ranking pipeline pulls follower count, username, and NSFW flags — nothing about a blue check. There is no verification feature in the model's inputs, no weight, no filter.

The one place verification *does* appear in the released code is outside the feed: the credibility system (a PageRank-style score over the follow and engagement graph) gives verified/premium accounts extra weight when distributing its "trust mass," and that credibility score shields accounts from *enforcement* (spam labels, suspensions, challenges). So a blue check can protect you from removal actions — it does nothing for your delivery. (Sept: the feed gained one new NSFW-author model signal, `nsfw_author_phoenix` — verification is still absent everywhere.) This matches the article's finding exactly.

---

## Part 6 — The AI-content accounts

> *"AI content often seems to perform better than everything else… very unnatural-looking profiles… high engagement and view ratios combined with unusually low hit concentration."* And: *"a sample post… with zero reposts, gets 28K views for a 33K account. What is going on here?"*

**What's happening.** First, the honest part: **there is no "AI content" detector on the feed path itself.** Nothing in the ranking code reads "was this written by an LLM" or penalizes automated posting. (Two qualifications: the released enforcement layer *does* contain LLM-content detection — the abuse-enforcement-service's `llm_slop_user`/`llm_slop_post` rules, which can label accounts SpamHighRecall and posts RiskyHighVizReply (**Sept**: joined by a new `panda_reports_embedding_v10_rough_spam → suspend` rule and a GrowthBook-gated generic-action path) — and the anti-bot behavior model (bdsm, eight classifications including "bot-like" patterns) ships with redacted thresholds (**Sept**: the action-count gate is redacted too). Both are labeling/enforcement systems: they are not part of the feed's scoring, their thresholds are hidden, and none of the sampled accounts show their labels.) So whatever these accounts are, their success is not being *suppressed* by the visible algorithm, and if their engagement is real, they are amplified by exactly the same machinery as everyone else: consistent content → consistent engagement → consistently positive learned reputation → the model confidently predicts engagement for every post → uniform, high delivery.

The **"low hit concentration"** observation is the tell. Real-people accounts have spikes: most posts land at baseline, a rare one catches fire. These accounts have no spikes because they don't need any — their delivery is carried by account-level state, not per-post luck (the mirror image of dawn_com's flatness at the other extreme). Content that reliably generates replies and dwell (poll-like posts, hot-topic commentary, consistent formats) produces exactly this pattern.

The **zero-reposts / 28K views** case is easy to explain once the machinery is clear: **views don't require reposts, and often don't require followers.** 28K views on a 33K-follower account is ~85% delivery — plausibly mostly followers — and even beyond that, the out-of-network layer (retrieval + similar-content clusters + topic feeds) serves posts to non-followers whenever the model predicts interest. Reposts are one distribution channel; they are not the only one, and for most accounts they aren't even the main one.

The uncomfortable truth the article gestures at: the system doesn't care where content came from, only what it predicts people will do with it. If an audience engages with formulaic content, the algorithm treats it as valuable content. That's a defensible critique of the *platform's values* — but it is not an anomaly in the algorithm; it is the algorithm working as designed.

---

## Part 7 — Why a repost from a big account often changes nothing

> *"I've had my posts occasionally reposted by large accounts and still performed poorly. I am not sure that reposts always ensure access to the full reach of those who repost them."*

**What's happening.** The article's suspicion is correct, and the code explains seven separate reasons why a repost doesn't inherit the reposter's audience:

1. **Reposts are scored per viewer too.** The repost appears as a candidate in the feeds of people who follow the reposter, and each of those people gets an individual prediction. The model decides which of the big account's followers will be shown it.
2. **Dedup filters.** If you've already been shown the original post (or a previous repost of it), the repost is removed — the system doesn't show people the same content twice in a session.
3. **The out-of-network filter.** Reposts from accounts you don't follow are filtered out entirely, as are replies from accounts you don't follow.
4. **The in-network repost discount.** Even reposts from accounts you *do* follow are explicitly discounted in the scoring — the code deboosts reposts and replies from followed accounts (a deliberate reduction, presumably to stop feed domination by reposters).
5. **The diversity step.** If the repost is similar to other posts already selected for the feed, it can be zeroed out entirely.
6. **The original's own reputation still applies.** The scored content is the original post, including the original author's learned reputation. A repost carries the original's baggage.
7. **Cascade drops.** If the original post is dropped by the content-safety gate for a given viewer (spam, safety labels, legal reasons), the repost of it is dropped for that viewer too — the drop propagates per viewer, and the repost never appears to anyone whose original was itself dropped.

In short: a big-account repost *adds your post as a candidate* in more feeds — it doesn't guarantee a seat.

---

## Part 8 — Posting frequency doesn't matter (much)

> *"There does not seem to be any strong correlation between frequency and reach."*

**Verified — and consistent with the code.** There is no posting-frequency feature anywhere in the scoring. Each post is scored on its own merits, per viewer, independently. The mechanisms that sound like "posting penalties" don't work the way people assume:

- The **author-diversity decay** (repeat authors get their 2nd, 3rd, 4th posts in a feed multiplied down toward a floor) is a *per-request, per-feed* mechanism. It resets every time anyone opens the app. It is not a lifetime posting limit.
- There **are** per-request caps on how many posts from a single author can even enter a feed's candidate pool (e.g., ~50 original posts, ~30 replies, ~100 videos). These are ceilings, not penalties — beyond them, extra posts simply don't add candidates.
- Posts are only candidates for **48 hours** anyway. Posting frequency mostly matters for keeping fresh posts in that window; it doesn't accumulate goodwill or penalties.

So the article's scatter chart makes sense: frequency is largely irrelevant to per-post reach; what matters is what each post does.

---

## Part 9 — What the content-type patterns mean

The article's content findings all trace to specific, visible scoring decisions:

**Videos get more reach.** Three mechanisms: dedicated video signals in the model (video-open probability — **0.07** since Sept — video-quality-view probability — **weight now 0.0**, so discovery-signal only), a rule that only videos **10+ seconds** long generate the "video view" engagement signal that feeds discovery, and — most importantly — video posts stay eligible for out-of-network distribution **far longer** than text posts (text posts effectively age out in days; the code maintains "evergreen" video pools measured in *years*, plus Sept's new age-banded `1fav_video_*` slices to 30 days). Text is judged fresh-then-dead; video can keep circulating.

**Photos get more engagement but fewer views.** The photo-expand signal is real but weakly weighted, and photos get no longevity bonus and no dedicated discovery pool. Photos please the people who already see them; they don't attract new ones.

**URLs underperform — and the article is right that there's no link penalty.** The old Twitter code's news-link boost is gone; the current code's only link-related signal is a small open-link click weight (0.2) — no penalty, no boost. URL posts underperform for content-dynamics reasons: people click away (no dwell time, and dwell is paid), links rarely generate replies (and replies are the highest-value engagement), and the model judges the post by its text, not the destination. dawn_com (100% links, 0.07% delivery) vs zerohedge (75% links, 3.1%) vs nikitabier (1% links, 81.6%) shows the same format producing a ~48× gap — format isn't the differentiator; the learned account state is.

**Articles get no special treatment — but they may deserve one.** X's native long-form Articles are *not* a distinct post type in the algorithm. There is no article flag anywhere in the feed path: the model's per-post inputs (tweet bools: reply/retweet/quote/media only; content embedding; SID; author) contain nothing article-specific, and there is no article weight, filter, retrieval pool, visibility rule, or even an Article bit in the served-metrics analytics bitset (the only "article" reference in the visibility service is an unused `is_article` method in a test mock). The one place articles appear is content understanding: grox feeds the title, cover and body media into the post's multimodal embedding (and therefore its SID) and tells its LLM classifiers the post is an article. So an Article is scored on its *content* — which is also why the 48-hour clock applies to it (no video-style evergreen pools) and why it can't earn video-only signals. The dataset's article posts (collector flag `IsArticle`) do over-deliver in aggregate — median views/follower 25.95% vs 9.65% for everything else — but that's 37 posts out of 10,000, 23 of them from one small account, so the cold-start boost and content quality explain it without any format boost. If long-form content engages, it gets amplified like anything else; the algorithm simply has no opinion about the format itself.

**Quotes transfer attention.** When you quote a post, the code explicitly scores "quoted-post clicks" — people click through to the original. Your quote gets the traffic; the original gets the engagement. And if the original is dropped by the safety gate, your quote can be dropped too (cascade rule). Quoting is a gift to the quoted.

**Bookmarks and likes are structurally special.** The discovery/retrieval model is *trained on favorites* — its entire objective is to find posts a user will like — which is why like-rates correlate most tightly with views: they're the same signal the machine is optimizing. Bookmarks feed the "recent interests" signals that drive the similar-content discovery source. Engaging with bookmarks and likes isn't just social; it literally trains the discovery system to send you more like that.

---

## Part 10 — Winner-take-all: why most of us don't exist

> *"Fetterman's engagement is equivalent to that of 81 other senators combined."*

**Verified (approximately).** Using median engagement per post, Fetterman's median (21,189 engagements) exceeds the sum of the median engagements of the **bottom 80 senators** (20,778). He beats all 99 other senators individually, and his total engagement over 100 posts is about a fifth of the other 99 senators *combined*. The article's "81 senators" is one senator too generous under the sum-of-medians computation — the bottom 80 sum to 20,778 (below 21,189) while the bottom 81 sum to 22,151 (above) — but the winner-take-all point is beyond dispute.

**What's happening.** This is the mathematical destination of Idea 3 + Idea 4. The feed has 35–47 seats per viewer. The model spends those seats on the posts it is most confident about, and the accounts with entrenched positive reputations collect the seats. Engagement concentrates → those accounts get more engagement → their reputations strengthen → they collect more seats. This is a **rich-get-richer dynamical system**, not a policy. Every engagement-optimizing feed in existence produces this shape, which is why "all social media would graph similarly," as the article says.

There is a real philosophical objection buried in this: the system's *goal* — "show people what they'll engage with" — is what produces the outcome the article finds objectionable, and optimizing for engagement is not the same as optimizing for good discourse. But the asymmetry between accounts is the mechanism doing its job, not a shadow agenda. A system designed to concentrate attention on what best predicts engagement will, by construction, concentrate attention.

---

## Part 11 — What this actually means for you

If the code is the truth, here are the levers that exist — and the ones that don't.

**Levers that exist (in rough order of strength):**

1. **Make people reply.** Reply carries the highest *weight* of the common engagements (10× a like's weight — which means "10× a like's weight at equal predicted probability," not "10 likes' worth of score"; Part 2). Conversation-starting posts — questions, provocations, polls, things people want to argue with — are what the model is paid to find.
2. **Make people stay — but know it's a small lever.** Dwell is a continuous head, but it's normalized and capped: the model's dwell prediction tops out around 30 seconds (a fixed ceiling in the model's output function), worth at most ~0.12 points — less than a quarter of a like. "Long dwell" is a real signal in the data, but the scoring weight is modest; replies and shares dwarf it.
3. **Make people share — especially via DM or copy-link.** The highest positive weights in the entire scoring table are on shares: copy-link shares (20), DM shares (5), regular shares (2). "This is worth sending to someone" is the strongest single signal in the system.
4. **Get follow-backs.** Original posts from **mutual follows** get a huge bonus (the reply-weight boost) — and as of the August 14th update, the model itself also receives *both directions* of the follow relationship as per-token features (does the viewer follow this author, and does this author follow the viewer), so mutual-follow status shapes prediction directly, not just the post-hoc boost. If someone you follow follows you back, your original posts become worth far more to them — and to everyone, via the learned reputation.
5. **Stay fresh and video-capable.** Posts older than 48 hours stop being candidates entirely; videos of 10+ seconds open up the video signal and the long-lived video distribution pools.
6. **If you're small (under ~1,000 followers), post low-Home-Timeline-view-count content deliberately.** The cold-start boost promotes exactly these posts — it's the one mechanism that actively rescues small accounts. (It also means the "quicksand" for tiny accounts is shallower than it feels.)
7. **Avoid the penalties.** The negative weights are enormous: a *report* prediction is worth −234 (vs +20 for a copy-link share). Content that generates reports, mutes, or blocks trains the account's learned impression downhill — and that state is what determines everything else.
8. **Don't post near-duplicates.** The diversity step actively removes posts that look like other posts already in the feed. Distinct formats and topics survive; repeated formulas get zeroed.

**Levers that don't exist:**

- Verification/blue check (delivery), pinned/highlights (the article is right — nothing in the code reads them), posting frequency, follower count, and "engagement rate" as such. None of these are inputs to the scoring.

**The honest, uncomfortable part:** the account-level learned state changes through **outcomes, over time**, and mostly on the platform's own schedule (retraining cadence is not published). One good post won't rewrite it; a consistent change in what your posts make people *do* will. And if an audience for your content genuinely doesn't engage — not because the algorithm hid it, but because the behavior data says so — then no setting in the published code will fix that. The system is a mirror of aggregate behavior. The fair criticism is not that the mirror lies, but that we have no right of appeal to it, no transparency about its state, and no way to know which side of the mirror we're on.

---

## Part 12 — What we still can't know

This explanation has limits, and honesty requires listing them:

1. **Production values may differ from the published defaults.** The repo ships the code and defaults; live weights are read from an internal configuration file that is not published. The article's data is from the same period as the release, so the published defaults are likely close — but "likely" is doing work.
2. **The value-model ranker's actual model is redacted.** The open "reranking" service contains only the diversity step; the model that originally combined the scores is not in the release. If that model has account-specific effects, we can't see them.
3. **The bot-detection thresholds are redacted.** The behavior-classification system (eight classes including "bot-like") ships with its operating thresholds hidden. It is not consumed by the feed code we can see, but we cannot rule out that production runs additional suppression outside the published path.
4. **The LLM prompts (spam classification, quality scoring) are not published.** Content-level judgments happen partly inside systems whose exact prompts are excluded.
5. **The data itself is limited.** Two convenience samples, point-in-time view counts (newer posts undercounted), follower counts that may include inflation, and no per-viewer data. The causality question — is low engagement the *cause* of low reach or the *symptom*? — cannot be settled with aggregate numbers, because the system's internal predictions are invisible.
6. **Timing.** The article's data and this code release are contemporaneous (July–August 2026), but the algorithm's learned state changes continuously. A snapshot explains mechanisms, not your account's future.

---

## Part 13 — What the system remembers about your account, and for how long

*"Are there permanent attributes not in that window? For example, number of times you've been blocked or muted? Is an account like dawn_com actually not recoverable?"*

The honest headline: **the system's memory is a stack of different windows, and the most important one — the learned reputation — has no expiration at all.** It only changes when new outcomes re-teach it. Here is the full inventory of what the code shows the system remembers about an account, and for how long.

### 13.1 The inventory: every learned attribute and its lifetime

| What's remembered | Where it lives | How long it persists |
|---|---|---|
| **The learned author reputation** (the vector of numbers encoding "what happens when this account's posts are shown") | The model itself — ranking and retrieval checkpoints | **No expiration.** No author-side time feature exists anywhere in the model. It persists across checkpoints until new training data moves it, on the training schedule (see 13.3) |
| **The same, for retrieval** (whether your posts get served to non-followers) | Retrieval model checkpoint | Same — no expiration, moves with retraining |
| **Your block/mute/report *history*** | No explicit counter exists. The events live only in training data (UAS) and in what they taught the embedding | As long as the events remain in the training data window (window not published) and the checkpoint they shaped is live |
| **Active blocks and mutes** (yours of others, others' of you) | Per-viewer filter lists, hydrated fresh every request | Persist while in place (until unblocked); binary effect while active |
| **Account-level credibility score** (user-cred) | Daily batch, Manhattan snapshot | Re-computed daily; **warm-started** — each day's score carries forward the previous day's mass for users who still exist, and the snapshot used may be up to 7 days old. Its *engagement* component only looks at the last **7 days** of favorites/reposts; the follow-graph backbone has no window |
| **Labels** (spam, NSFW, abusive, impersonation, compromised, read-only…) | Label stores + gizmoduck | Varies by labeler: enforcement labels **30 days**; the real-time spam-label store **26 hours**; post→account label aggregation **7 days**; suspensions **permanent** (until action); several user labels ship with no visible TTL — they live until their owner removes them |
| **The viewer's recent behavior** (UAS) | The action sequence fed to the model | **5 minutes** (rolling), truncated to the last ~1,000 actions |
| **Your follower/following counts, account age** | Stored account data | Permanent while true |
| **Post age** | Feature + filter | Posts stop being candidates after **48 hours**; the model buckets age up to ~80 hours; a **14-day staleness flag** zeroes engagement counts (**enabled in `home_direct_packed` since Sept**) |
| **Aggregate engagement ratios** (e.g., blocks-per-fav, reports-per-fav in the agatha modeling pipeline) | Snapshot datasets → user-health labels | These are *recomputed* periodically (windows not visible in the repo) and feed the labeling systems — their reach-limiting effect lasts as long as the resulting label does |

### 13.2 The block/mute question, answered precisely

**There is no feature anywhere in the ranking or retrieval model that says "this author has been blocked X times."** We checked every input the model actually consumes: per-post content (SID, multimodal embedding), the author's ID, engagement counts, media flags, language, post age, and the viewer's own context (country, language, gender, age, apps, follower/following counts). One relationship *is* now a model input: as of the August 14th update, the serving engine stamps **both directions of the follow relationship** as per-token boolean features — whether the viewer follows the candidate's author (`isAuthorFollowedByUser` → `IS_AUTHOR_FOLLOWED_BY_VIEWER_SEQ`) and whether the author follows the viewer (`author_info.is_following_user` → `IS_AUTHOR_FOLLOWING_VIEWER_SEQ`), on both candidate and history tokens (util.rs). **Sept**: the indices are now shipped (`feature_config.py:42-44`) and `home_direct_packed` enables both flags (xrecsys.py:289-290); on retweets, however, the model receives neither flag nor any follower count (candidate.rs). The viewer's own blocker/muter counts (`user_blockers_count`/`user_muters_count`) remain defined in the proto and **unconsumed**. Nothing author-side counts blocks.

Blocks and mutes still matter — through three indirect paths, none of them a counter:

1. **Per-viewer, immediate, binary:** if the *viewer* blocks or mutes the author, the author's posts are removed from that viewer's feed on the spot (both directions — if the author blocks the viewer, same result). This is a filter, not a score, and it persists while the block does.
2. **Per-viewer, learned, per-request:** the model *predicts* for each (viewer, post) pair the probability that the viewer will block, mute, or report the author. Those predictions carry the largest negative weights in the entire scoring table (report −234, mute −58.8, not-interested −43.2, block −31.2). An author whose posts routinely provoke these actions gets high predictions and low scores. How *broad* that damage is, though, is contested by the platform's own August 14th comment in the code (param.rs): it argues the predictions are personalized — reports from bad actors "will primarily affect recommendations for users who are similar to the bad actors, rather than having the same effect on the post's ranking to everyone" — that only actions on posts *served in the Home Timeline* count toward ranking (directly navigating to a post has no ranking impact), and that the baseline predicted probability of a report is more than 1,000× lower than a like, which is why the report weight is so high. The mechanism is real either way; the *scope* — author-level suppression vs. per-user suppression — is exactly the kind of thing aggregate public data cannot resolve.
3. **Training, permanent-until-relearned:** every actual block/mute/report/unfollow is a training example. The model is built with an explicit rule — when a "negative feedback" action occurs on a post (block, mute, report, not-interested, see-fewer, unfollow, not-relevant, mute-conversation, notification-see-less-often), the loss **zeroes all the non-negative predictions for that post** (recsys_model.py:2781-2823), making the event a hard negative. One important caveat about this mechanism: it is gated behind a flag (`mask_candidate_positive_when_negative_action_present`) that **ships disabled** — no released config turns it on (recsys_model.py:511, xrecsys.py:554-556). If production enables it, negative events are doubly potent; as shipped, negative outcomes teach through the ordinary loss instead. Over checkpoints, either way, the author's learned reputation absorbs them.

So "number of times blocked" exists as *a shape in the learned reputation*, not as a number. Its persistence is the persistence of the embedding itself — which is the part of this answer that deserves emphasis:

### 13.3 The one attribute with no expiration

The learned author reputation is **static in time by construction**. The model has no author-age feature, no "recent activity" feature, no decay on the author's vector, no expiration date. No author-side time feature exists anywhere in the model; the only *post-time* features are *post* age — and a **14-day staleness flag** (`IS_STALE_POST14D` — **enabled in `home_direct_packed` since Sept**, xrecsys.py:272): candidates older than ~14 days have their engagement counts zeroed and the flag set (util.rs:464-477, `STALE_POST_14D_TTL_SEC = 1,213,200`). (Timezone and local hour-of-day features do exist, but they encode the viewer's local time on every token — they say nothing about the author or when the post was created.) This means:

- An author's reputation is exactly as fresh as the **latest checkpoint** containing it.
- It changes **only when retraining re-learns it from new outcomes**. The training code ships with Kafka-streaming data feeds — it is built to consume training examples off live topics (kafkaconsumer.py, data_feeds.py) — and the production model configs checkpoint every 150–300 steps. But the public launcher is an offline harness over a finite synthetic dump (TRAINING.md:8-10), and the two numbers that actually matter are not published: the *data window* each checkpoint learns from, and the *rollout cadence* to production. Those two numbers are the actual answer to "how long does a bad reputation persist," and the repo deliberately doesn't say.
- Nothing resets it. There is no "clean slate," no periodic reputation wipe, no expiry on the embedding.

By contrast, the **credibility score** (the only explicit reputation number) has the famous **7-day window** — but note what that window covers: only the *engagement* component (favorites and reposts feeding the PageRank teleport). The follow graph beneath it has no window, and the daily recompute is warm-started from the prior day's mass, so the score evolves gradually rather than resetting. The credibility score also doesn't affect feed delivery at all — it gates *enforcement* (spam labels, suspensions) and spam screening. Recovery of the credibility score can be roughly a matter of weeks (7-day engagement window, daily snapshots). Recovery of the *feed* reputation has no published schedule.

### 13.4 Is dawn_com actually not recoverable? (corrected)

*An earlier draft of this section said dawn_com "has no negatives, no labels, no enforcement — just an absence of positives." That sentence overclaimed, and this subsection is the correction.*

Short answer: **not permanently condemned — but part of what we'd need to be sure of that is unverifiable from the available data.**

**What we can actually verify about dawn_com:**

1. **It is not suspended and its posts are served.** All 100 sampled posts have views (minimum 432, median ~900). This is the strongest test available, and it rules out the harsh end of suppression: a suspended account would produce no posts, and *any all-surface drop* — a suspension, or a post-level label like SPAM, PDNA or BOUNCE (the tweet-label drops in the VF base rules) — would remove its posts for every viewer and produce roughly **zero views**, not 900. None of those are currently active on the sampled posts. (Note that the *user-level* labels — SPAM_HIGH_RECALL, READ_ONLY, COMPROMISED, IMPERSONATION, the NSFW family — are all out-of-network-only: they exempt followers, so they could never produce zero views anyway; an OON-only restriction and a mere low score look identical in aggregate view counts, which is item 2 below. **Sept precision:** those OON user labels drop followers too when OON — only ABUSIVE_HIGH_RECALL and DO_NOT_AMPLIFY exempt followers — so the "could never produce zero views" reasoning holds via the in-network allowance, not via follower exemption.)
2. **The audience that does get served doesn't engage.** Median 1 like per post, ~0.007% reply rate, 0.17% engagement overall (median of per-post engagement rates), 20% of posts with zero interactions of any kind — and this is the audience the model selected as *most likely* to engage. A label doesn't do that; labels remove posts for whole classes of viewers, they don't choose low-propensity viewers. The pattern is the signature of a low-propensity prediction state.
3. **The follower path is a scoring story.** 900 views is ~0.07% of its 1.38M followers. Follower delivery isn't touched by out-of-network-only restrictions, and the same low band is occupied by senators with no plausible label story (Gillibrand 1.33M followers at 0.5%, Durbin 684K at 1.1%, Blumenthal 655K at 0.8%). Per-viewer scoring alone produces this magnitude.

**What we cannot verify — and why the original sentence was wrong:**

1. **Negative feedback that is invisible in public data.** Blocks, mutes, reports, "not interested," "show fewer," and unfollows are not public, and the dataset (which only records likes, replies, reposts, quotes, bookmarks, views, flags) does not contain them. The model's training absolutely includes these events — any of the nine negative-feedback actions zeroes the positive training signal for that post. A link-spamming news account can absolutely accumulate "not interested" / "show fewer" / block feedback from annoyed users. If dawn_com's history contains that, its learned reputation is *actively negative*, and the heavy negative prediction weights (−31.2 to −234) are live against it — not dormant as the earlier draft assumed.
2. **Out-of-network-only restriction labels.** Two user labels (DO_NOT_AMPLIFY, ABUSIVE_HIGH_RECALL) exempt followers; the other OON user labels (SPAM_HIGH_RECALL, READ_ONLY, COMPROMISED, IMPERSONATION, the NSFW family) drop followers too when OON — but all of them allow in-network, so no user label can zero out follower delivery. Several out-of-network-only (OON-only — applying only to people who do not follow the author) tweet rules (e.g., SPAM_HIGH_RECALL) drop a post only for non-followers. In aggregate view counts, an OON-restricted account and a merely low-scored account look identical. dawn_com could carry one of these and the 900-view number would look exactly the same.
3. **Past enforcement and expired labels.** Enforcement labels carry 30-day TTLs; the real-time spam-label store holds labels for ~26 hours. Anything applied and lapsed before the sampling window is invisible to us.

**Corrected verdict.** "Its problem is absence of positives" should read: **absence of positives in the public record.** The private record could be quiet or actively negative, and we cannot tell which — which matters for recovery speed, because an indifferent reputation re-learns faster than one trained on blocks/mutes/reports. Either way the strategy conclusion is unchanged: the loop runs on outcomes, nothing in the code permanently condemns the account, and only changed engagement outcomes move the state. But the honest timeframe spans from "weeks to months" (indifference) to "considerably longer" (active negatives), with both schedules unpublished. The epistemic rule of thumb this correction establishes: **with the data we have, "not administratively punished" is verifiable; "no negative history" is not.**

### 13.5 A table for the impatient

| Question | Answer |
|---|---|
| Do we know how long learned account attributes persist? | Persistence = training data window + checkpoint rollout cadence. **Both unpublished.** What is known: no author-side expiration exists in the model |
| Are there permanent attributes? | Yes, structurally: the learned author reputation (until re-learned); suspensions; and any user label with no TTL. Nothing has a *scheduled* reset |
| Is there a "times blocked/muted" counter? | No — the events act via the learned reputation (which absorbs them) and via per-viewer binary filters, not via a count feature |
| What's the 7-day window, exactly? | The credibility score's engagement-teleport component (7 days of favorites/reposts), plus warm-start snapshots ≤7 days old. It does **not** govern the feed model's learned reputation |
| Is dawn_com recoverable? | In principle yes — and "not administratively punished" is verifiable (posts are served, not suspended; any all-surface drop label would yield ~0 views). "No negative history" is **not** verifiable — blocks/mutes/reports and OON-only labels are private. Recovery requires sustained positive outcomes either way, over an unpublished timeline |

---

## Part 14 — The weight of negative history: is there a limit, and what does recovery actually take?

*"Without knowing the private negative feedback, do we not know the effort required to recover a poorly performing account? Is there no limit to negative feedback that can be accumulated, and how heavy might it be weighted?"*

Two separate questions hide in there — how heavy the penalty can be *per post*, and how much bad history can *accumulate* over time. The code answers both partially, and one of the answers is a genuinely surprising finding about the training optimizer.

### 14.1 How heavy can negative predictions weigh? (The serving-time math, now open)

The scoring weights are published, and the predictions they multiply are probabilities (0 to 1) — per-viewer predicted probabilities, not raw engagement counts. The August 14th update added an explicit official clarification of this in the code itself (param.rs, ranking_scorer.rs, README): the weights "do not multiply raw engagement counts," so reading weight ratios as count equivalences — e.g. "one report cancels 468 likes" — is incorrect; the report head is weighted −234 because the *baseline probability* of a report is more than 1,000× lower than a like, not because one report outweighs 468 likes. The same compensation runs through the positive side: DM shares (5) and copy-link shares (20) are rare actions, so their weights are inflated relative to the common like (0.5) — every term contributes `weight × P(action)`, which is why "reply is 10× a like" is a weight comparison, not a value one (the platform's VP of Product popularized exactly this framing on Aug 15, 2026). That framing matters for every bound below: these are *upper bounds on a per-viewer prediction*, not a ledger of engagement counts. With that said, the bounds are staggeringly asymmetric:

- **All positive predictions at maximum confidence** (favorite + reply + repost + all shares + quote + follow + dwell-prob (0.05 since Sept) + dwell at its 30-second cap + everything, all = 1.0) sum to **≈ 43.5 points** (≈ 58.5 for a mutual-follow original post, with the +15 boost).
- **All negative predictions at maximum confidence** (report + mute + not-interested + block + not-dwelled, all = 1.0) sum to **≈ −367.2 points**.

So the negative side has **≈ 8.4× the raw weight of the entire positive side**. The single report head (−234) alone outweighs every positive prediction at full confidence. More usefully: a **~19% predicted probability of report** (−44.5) outweighs *everything* positive a post can earn. For a *given viewer*, a post the model thinks has even a one-in-five chance of being reported by that viewer is mathematically unable to win that viewer's seat, regardless of how loved it is otherwise. The platform's own commentary adds two boundary conditions: (1) these are per-viewer predictions, so the "unable to win" verdict is per-viewer — a post can be dead for viewers predicted to report it and alive for everyone else; and (2) only actions on posts *served in the Home Timeline* are counted at all (direct navigation has no ranking impact), and the model cannot be made to serve a post to a viewer who would not be predicted to see it anyway.

And there's a structural twist in the scoring formula that makes this even more binary. Scores are computed as *positives minus negatives*; then:

- net-positive score → keeps its value (plus a tiny offset)
- **net-negative score → compressed into the range [0, 0.001)** — a floor band

Every post that ends up net-negative — even barely — is squashed into a sliver at the bottom where it loses to every net-positive post by a factor of a thousand or more, and where *degrees* of negativity barely matter anymore (a mildly negative post and a catastrophically negative post rank almost identically: both dead). The ranking consequence is stark: **a post is either competitive or in the grave; there is almost no "slightly punished" state.** For an account in recovery, this means progress is not gradual in feed terms — the model's predictions must actually cross the sign before anything visibly changes.

### 14.2 Can negative feedback accumulate without limit?

There are **three hard limits** on accumulation in the code — and none of them is a "reputation reset":

**Limit 1 — probability saturation.** The learned predictions are probabilities. Once the model is certain a post will be reported (P ≈ 1), additional reports teach it nothing new; the penalty is already maxed. More negative events change nothing beyond saturation.

**Limit 2 — the rolling training window.** The embedding is re-learned from a window of recent training data (window length unpublished). Events older than the window stop contributing entirely. Accumulation is rolling, not total — old history expires by construction.

**Limit 3 — the optimizer forgets (the surprising finding).** The account embedding rows are updated with a specialized optimizer — the math that updates the learned values during training — and its configuration ships with **an exponential decay on the accumulator (its running memory of past updates): a half-life of 5,000 training steps by default** (rowwise_adagrad.py:37 — **Sept**: the production configs now set it explicitly, 2500 on H100 / 5000 on GB300, with lazy decay plus a small weight decay, and dense training moved from AdamW to **Muon**). Plain AdaGrad has a known failure mode: its accumulated gradient history grows forever, so a frequently-updated row's effective step size shrinks toward zero and the embedding *freezes* — old, busy accounts would become permanently immovable. This implementation explicitly prevents that: **old gradient history decays, so the embedding stays as movable on day ten thousand as on day one.** Long history does not lock the account in. (If this were vanilla AdaGrad, "accumulated negative feedback" would indeed be effectively permanent — the fact that the decay is there, by default, is the strongest evidence in the code that the platform does not want reputations to be permanent.)

There is also an aggravating asymmetry in *training* worth knowing: when any of the nine negative-feedback actions occurs on a post, the loss **zeroes out all the non-negative learning signal for that post** (recsys_model.py:2781-2823) — *when the flag enabling it is on*. The caveat matters: **the released configs ship with that flag off** (`mask_candidate_positive_when_negative_action_present` defaults to false and no shipped config sets it true), so in the code as published a block or "not interested" adds a negative example but does not erase the positive one. If production enables the mask, negative events are doubly potent — they both add negative evidence and *erase the positive example that impression might otherwise have been* — and the trap deepens accordingly.

And the non-learned paths are all TTL-bounded: enforcement labels 30 days, the real-time spam-label store ~26 hours, post→account label aggregation 7 days. The only permanent negatives in the entire system are suspensions.

### 14.3 So what do we actually know about recovery effort?

**The honest summary: we know the shape of the effort, not the amount.** The amount is set by four numbers, three of them hidden:

| Number | Status |
|---|---|
| How negative the private history actually is | **Unknowable from outside** (lives in the checkpoint) |
| The training data window length | **Unpublished** |
| The embedding learning rate | Published (**0.28** since Sept; was 0.2) — but its interaction with window is what matters |
| Checkpoint rollout cadence | **Unpublished** |

What the code does establish, regardless of those:

1. **Recovery is structurally finite.** Nothing in the model is permanent: the window rolls, the accumulator decays, predictions saturate and re-learn. The minimum recovery time is roughly *one training window + one rollout lag* — whatever those numbers are. It is not a life sentence.
2. **The effort is proportional to the balance of outcomes in the window, not to a ledger of sins.** The embedding converges to whatever the recent statistics say. An account needs its recent window to contain enough positive outcomes to outweigh the negative ones — and note the double potency: it also needs the negative *events* to stop, because each one both adds negative signal and (if production enables the negative-masking flag from Part 14.2) erases a positive example; the negative signal itself stands either way.
3. **The serving-time lag is real.** Even after behavior changes, the model keeps predicting the old pattern until a new checkpoint rolls out — and during that lag, the offset compression keeps the account's posts in the bottom band (see 14.1). Recovery has a "shadow penalty" phase where the account is doing the right things and still seeing nothing change.
4. **For a specific account, the volume math is the computable part.** dawn_com: ~900 views per post, median ~2 engagements per post (mean 3.2 — a trickle by any measure). Whatever the training window length, their posts generate engagement examples only at the rate of ~2 per post against a history of millions of impressions with near-zero engagement. That's the real reason their recovery would be slow: not a penalty, but the sheer arithmetic of needing enough positive examples to re-educate the model — at a trickle of ~2 per post. The trajectory is deterministic; the timeline is hidden.
5. **The answer to the original question, precisely:** yes — without knowing the private negative feedback (and the two hidden training numbers), we cannot know the *amount* of effort required. But we can know the *kind* of effort: sustained positive outcomes, at sufficient volume to dominate the training window, persisted through the rollout lag, with the penalty phase understood as expected and temporary. And the code contains no mechanism by which old negative feedback becomes permanently locked in.

### 14.4 A correction to earlier parts of this document

While verifying the math for this section, a magnitude error surfaced in Parts 11 and 13's description of dwell time. The model's dwell prediction is sigmoid-activated and de-normalized by its norm scale (30), so predicted dwell is **capped at ~30 seconds** on the For You feed surface (the gallery-page surface uses a separate tweedie head with a 300-second cap that does not feed this score), making the maximum dwell contribution to a score **≈ 0.12 points** (30 × 0.004) — less than a quarter of a like (0.5), and ~40× weaker than a reply (5.0). "Make people stay" is a real but small lever; it was previously overstated in this document and is corrected here (and in the underlying [algo-analysis.md](algo-analysis.md)). This correction *strengthens* the 14.1 asymmetry: there is no large continuous positive that could ever offset a meaningful negative prediction.

---

## Part 15 — The quicksand hypothesis: is it right, and would the proposed fixes work?

*"Many of the negative aspects people complain about are due to account-level learned attributes that affect engagement prediction. These can become self-reinforcing, creating quicksand that's difficult to escape. Improvements might be definitive windows or cooldown periods that let accounts escape, or removing more account-level attributes and letting individual posts weigh more in prediction."*

This is a testable hypothesis against the open code. The verdict, in one line: **the mechanism is real and the code shows exactly how it self-reinforces; of the two proposed fixes, one is already partially implemented, and the other would amputate the discovery system — but there are code-native versions of both that the platform has conspicuously not built.**

### 15.1 How much of a prediction is actually account, and how much is post?

The candidate representation fed to the model is a concatenation of four components (recsys_model.py:1018-1105):

1. **The post's own ID embedding** (unique to this post, learned)
2. **The post's semantic ID (SID)** — content-based, derived from the post's multimedia
3. **The author's embedding** — *the only component that is persistent, personalized, and per-account*
4. **Product surface** — plus multimodal content embeddings added on top

In raw dimensions the post/content side outweighs the author side. That matters for the *healthy* accounts: it's why good posts from good accounts spike, and why even good accounts' posts vary. But it also hides the real problem. For an account in the trap, the content components change with every post while the author component never does — and the model's **prediction** for each new post inherits the account state no matter what the content is. The empirical signature of this is the flat ceiling: dawn_com's delivery is 0.07% across all 100 posts (near-zero hit concentration), and its escape posts — the ones where content temporarily won — are rare and small. Dr. Drew's single 1.21M-view post is the proof that content *can* win; the account state is strong, not absolute.

Two further facts sharpen the picture:

- **The author component appears in *both* ranking and retrieval.** Retrieval's candidate tower is built from SID + author, and its user tower matches against the *authors you've engaged with* in your history. So the account state controls not just "how high does this rank" but "does this post get proposed to new viewers at all." The quicksand is doubly sealed: trapped accounts stop reaching their existing viewers *and* stop being discovered by anyone new.
- **The account state is personalized exactly where it hurts most.** Small accounts' author embeddings are diluted by hash-bucket collisions (they're judged as "average author"); large accounts own their bucket, so a large account's negative state is uniquely its own — no dilution to soften it, and no cold-start boost (capped at 1,000 followers) to counter it.

So the hypothesis's first half is **confirmed with one correction**: it's not that account attributes dominate the *input* — it's that the account attribute is the *only persistent* input, and it propagates into every prediction through a loop.

### 15.2 The reinforcement loops — the actual quicksand mechanics

Four loops in the code, in order of importance:

**Loop 1 — the serving loop.** Low predicted engagement → few serves → few impressions → few engagement examples → the embedding continues to encode "low engagement" → low predictions. This is the main trap, and it's why the trap is *flat*: every post inherits the same state and produces the same nothing.

**Loop 2 — the discovery loop.** Low engagement also weakens the retrieval representation → the post is rarely proposed to non-followers → no new-viewer engagement data → the retrieval state stays weak. A trapped account cannot escape through out-of-network discovery even with a good post, because retrieval only proposes what it already predicts.

**Loop 3 — the double-potency of negatives.** When any negative-feedback action occurs (block, mute, report, "not interested", "show fewer", unfollow…), the training loss **zeroes the non-negative learning signal for that post** (recsys_model.py:2781-2823) — *when the enabling flag is on*. The caveat from Part 14.2 applies here: the released configs ship with that flag **off**, so in the shipped code a negative event adds negative evidence but does not erase the positive example. If production enables the mask, the trap deepens faster than equivalent positives would lift it.

**Loop 4 — the floor band.** Net-negative posts all compress into the score range [0, 0.001) (Part 14.1). Small improvements are invisible: a post that goes from "catastrophically negative" to "barely negative" still ranks dead last, indistinguishable from the worst. Progress on the way up is unmeasurable from the outside, which is why accounts in the trap report "nothing I do matters."

**The floor.** The quicksand is not a vacuum: the feed must fill ~35–47 slots, previously-seen filters force fresh candidates, and pool exhaustion means low-score posts occasionally surface (dawn_com's ~900 views per post are this trickle). But a trickle of impressions at 0.17% engagement is exactly what *sustains* the trap — it provides just enough data to confirm the low predictions and never enough to change them.

### 15.3 Fix A — definitive windows or cooldowns

What already exists, and what doesn't:

| Mechanism in code | What it is | Is it "definitive escape"? |
|---|---|---|
| Rolling training window (length unpublished) | Old events age out of the learned state | Soft and hidden; not account-level |
| Optimizer accumulator decay (5000-step half-life default; **Sept**: 2500 H100 / 5000 GB300 + lazy decay + weight decay, dense on Muon) | Keeps the embedding movable forever | Prevents *permanent* locks; sets no escape time |
| 5-minute viewer history window (UAS) | Viewers forget quickly | Viewer-side only, not account-side |
| Label TTLs (30 days / 26 hours / 7 days) | Label-based penalties expire | Bounded, but labels aren't the main trap |
| **Nothing** | An account-level "clean slate," reset, or cooldown after sustained good behavior | **Does not exist in the code** |

A definitive cooldown would need to be one of two things:

**(a) An external reset** — "after N days of clean behavior, the account's learned state is reinitialized." Nothing like this exists, and there's a design reason it probably never will: **it is gameable, and the README explicitly says some internals are withheld to reduce gaming risk** (README.md:416). A spammer's calendar is exactly as long as the cooldown; a public reset window is an arbitrage opportunity. The platform's shipped machinery — hidden windows, soft decay, saturation, rolling re-learning — reads as a deliberate architecture of *no hard promises*.

**(b) A clock the model can learn.** This is the code-native version of the user's proposal and the one the architecture would actually support. Today the model has **no time dimension for authors at all** — no "days since the state last changed," no "days since last negative outcome," nothing. The only time dimension that exists is *post age* (plus viewer-local timezone/hour-of-day context). Adding an author-state clock (e.g., an embedding input for "days since this author's last negative feedback action" or "age of the current learned state") would let the model *itself* learn the recovery curve — the model could decide that an account with 60 clean days deserves a reset-like treatment, and do it differently per account, per viewer, per post. This preserves anti-gaming opacity (no public rule to wait out) while giving trapped accounts a real escape schedule.

### 15.4 Fix B — remove account attributes, weight posts more

This one needs a careful cost accounting, because the code shows what would break:

**What the author component is actually for.** It is the *relationship* signal. Retrieval finds new posts for you partly by matching "authors similar to authors you've engaged with"; ranking scores a candidate partly by how your history generalizes to its author. Removing or heavily de-weighting the author component doesn't just weaken the trap — it removes the "posts by people like the people you like" mechanism entirely, collapsing discovery into pure content similarity (SID/multimedia). Small accounts would lose the one personalized signal they have (diluted as it is). The Fetterman-vs-Sanders effect would survive (that's viewer-side, not author-side). The dawn_com-type ceilings would weaken — but so would prediction quality across the board, because author identity is *predictive*; that's why the network learned to use it.

**The interesting nuance:** in raw input terms, posts already outweigh the account — and healthy accounts show exactly that (viral tails, hit concentration, content-type effects). The trap isn't caused by the *input* weighting; it's caused by the persistent state poisoning the *prediction* for every post, plus the floor band making partial progress invisible. So reweighting inputs is aiming at the wrong layer. The precise version of the user's proposal is: **scale the author component's influence down at prediction time when the state is stale** (few or no recent outcomes, old negatives) — a "staleness discount" — and scale it back up as recent outcomes accumulate. Same idea as Fix A(b), but applied to the component rather than the whole score.

**One version of Fix B already exists, in miniature.** The post-age buckets give every post a fresh-start window where age is the dominant post feature; the cold-start boost (≤1,000 followers) exploits the fresh window for small accounts; `post_unexplored` rewards never-seen posts (weight 0.02, in-network only). What's missing is the generalization: a **"new-post honeymoon"** — during the first hours of a post's life, de-weight the author component and up-weight the post components at prediction time, for *all* accounts. That is exactly "individual posts weighing more vs. account attributes," it is implementable with the age feature that already exists, and it is the closest the code comes to a viable version of the proposal.

### 15.5 What the platform has already chosen (and what it hasn't)

The inventory of shipped anti-trap mechanisms is revealing: cold-start boost (real, but capped at 1,000 followers and one post per request; a Thompson-sampling variant shipped Aug 14 but is off by default, top-2 since Sept), `post_unexplored` (real, but 0.02 and in-network only), the accumulator decay (real, but hidden in the optimizer — **Sept**: now explicit per-hardware plus weight decay), the rolling window (real, but length hidden), the 5-minute viewer forgetting (real, but viewer-side). Every one is **soft, time-delayed, and hidden**. The pattern is a consistent design philosophy: *bounded memory plus minimal guaranteed exploration, with no hard promises and no public schedules.*

What the platform has conspicuously **not** built: an account-level escape clock, guaranteed exploration slots for all accounts (the cold-start slot exists only for ≤1,000 followers), or any prediction-time staleness discount. The quicksand the article's data documents is therefore not a bug — it's the designed-in floor: the trap is real, the escape hatches are small, and the definitive escape is left to the slow re-learning loop whose schedule nobody outside the company knows.

### 15.6 Verdict

1. **Hypothesis confirmed with precision**: the negative aspects are substantially driven by persistent account-level learned attributes; they self-reinforce through four identifiable loops (serving, discovery, negative double-potency, floor band); the empirical signatures — flat ceilings, near-zero hit concentration, single rare escape posts — match the mechanics.
2. **Cooldowns/windows**: directionally sound; the hard-reset version conflicts with the platform's evident anti-gaming philosophy and doesn't exist; the code-native version is an **author-state clock feature** that lets the model learn recovery schedules itself.
3. **Less account weighting**: removing author attributes would amputate the discovery mechanism; the code-native version is a **prediction-time staleness discount** on the author component and a **new-post honeymoon** using the existing age feature. Note the nuance: posts already outweigh accounts in the input; the problem is the state poisoning the prediction, which is a features problem (staleness clock, exploration slots) more than a weighting problem.
4. **The minimal change that would actually break the loop**: guaranteed exploration slots for *all* accounts (not just ≤1,000-follower ones) — a reserved seat in the feed for underexplored posts, which guarantees every account a floor of impressions, which guarantees the training loop a flow of engagement data, which is the one thing that can move a flat state. The cold-start boost proves the platform knows how to build this; it just hasn't extended it beyond tiny accounts.

---

## Part 16 — Author-centric vs. content-centric: what the architecture actually is

*"I'm not looking for authors, I'm looking for posts on topics. I don't want to be fed something just because it came from an author. IMO, this is a big part of what is wrong with social media."*

This deserves a straight answer, because it's the deepest product decision in the entire system — and the code shows both that the complaint is architecturally true, and that the alternative already exists in the codebase, unused at the main stage.

### 16.1 The feed is author-centric by inheritance

The entire skeleton of X's recommendation system is built on the follow graph — a pre-ML inheritance from Twitter:

- **In-network content (Thunder)** *is* the follow graph: posts from people you follow.
- **The OON discount** (0.75) and the **mutual-follow boost** (+15 on reply weight; a parallel +2.0 dwell-bonus mechanism exists but ships at 0) are author-relationship rules — and as of the Aug 14 commit, both follow directions are also explicit *model features* (`IS_AUTHOR_FOLLOWED_BY_VIEWER`, `IS_AUTHOR_FOLLOWING_VIEWER` on candidate and history tokens — enabled in `home_direct_packed` since Sept), so author relationship enters at two layers: the score multiplier and the prediction itself.
- **The credibility score** (user-cred) is a PageRank over follow + engagement edges — a graph of people.
- **Every candidate representation, in both ranking and retrieval, contains the author's learned embedding.** In the retrieval tower, the author hashes are unconditional — *always* present — while the content (SID) is a toggled option (enabled in production configs). In the ranking model, the candidate is post-id + SID + author + surface.
- **The viewer profile is built from a history of *who* they engaged with as much as *what*.** The model's history tokens carry author embeddings of the accounts you interacted with, and the retrieval user-tower matches candidates against "authors like the authors in your history." That is exactly the mechanism you're objecting to, and it is structural, not incidental.

So: you are correct that the system will feed you something because of its author. The author component is the one persistent, predictive, relationship-carrying signal in the representation, and the model uses it because it works — people do engage more with authors they know, and the training data says so.

### 16.2 But the content pathway exists — and is substantial

The same codebase contains a genuine, content-first machinery:

- **Semantic IDs (SID)**: every post gets a six-layer code that compresses its content — the layers get progressively finer, so same-topic posts share early layers (the *prefixes*). The code is derived from the post's multimedia embedding (a representation of its images, video and text together). This is literally a learned topic taxonomy, built from content, not from authors. Both the ranking model and the retrieval tower consume it.
- **Topic feeds**: a whole parallel path — followed topics, inferred topics, topic filtering (`TopicIdsFilter`), topic-specific OON weight (0.5), and a topics candidate source.
- **Content-type signals**: video-open, photo-expand, open-link, dwell — all content-driven heads. (**Sept**: the VQV head's weight went to 0.0 — "video watched" no longer scores directly, only via the implicit discovery signal and dwell.)
- **Two experimental content-first retrieval models ship in the repo**: a model that *generates the next post's SID* (the prediction target is pure content — "what topic will you engage with next"; no author is predicted, though the input history still carries author tokens), and a generative-retrieval model that predicts the next post's *embedding* directly. The platform has already built content-first discovery; it just isn't the main path yet.

### 16.3 Why the author signal is entangled — and why that's the real problem

The model cannot tell *why* you engaged with something. The training data records "this viewer engaged with this post by this author," not "because of the topic" vs. "because of the author." So the learned representation conflates them, and the author correlation is strong — which is precisely what creates the negative aspects documented in Part 15:

- **The quicksand is an author-state phenomenon.** A content-only system couldn't entrench dawn_com — its posts would be judged on SID and content alone, where a genuinely good post could win.
- **Reach concentration is author-concentration.** Fetterman's engagement, the 7,000× gap, the flat ceilings — all ride on author state propagating across every post the author makes.
- **The author echo chamber is real.** If you engage with politics from one set of authors, the author-matching path feeds you *those authors' peers*, not the topic. The topic path exists, but it's one component competing with the author path.

### 16.4 What a content-first version would look like (all parts already exist)

- **Retrieval**: candidate tower = SID + multimodal only (drop the author hashes — the code already supports `use_post_sid` with toggles). User tower = content history (post/SID embeddings) only. Discovery becomes "posts whose content matches what you engage with," regardless of author.
- **Ranking**: de-weight or gate the author component at prediction time — especially for accounts whose author state is stale (Part 15.4).
- **Keep the follow graph for in-network** (following is a deliberate human choice; it should stay), but make out-of-network discovery purely topical.
- The repo's own SID-generation and gen-recs models are proof-of-concept for exactly this design.

### 16.5 The honest tradeoff

- **Prediction quality drops.** Author identity is statistically predictive of engagement. A content-first feed shows you more of what you've *already engaged with the content of* and less of what you'd *grow into* — and it would likely retain fewer users, because relationship-based feeds are sticky. This is why it's a product decision, not a technical one.
- **"Topic" is engagement-shaped.** SID clusters are learned from engagement data; a "topic" in this system means "content that people who engage like this also engage with" — not a human-defined category. Content-first would still be optimizing engagement, just along a different axis.
- **The complaint and the fix are aligned with the code's own direction.** The author-entrenchment effects (quicksand, concentration, echo chambers) are the strongest *technical* arguments for content-first; the platform's shipped experiments suggest the company is at least exploring it; the absence of it from the main path suggests the engagement-retention tradeoff is currently winning.

Your instinct — that author-based discovery is a large part of what's wrong — is consistent with everything the data and code show: the worst pathologies in this analysis (flat ceilings, entrenched reputations, winner-take-all) are all author-state phenomena, and the content pathway is the one place the system already knows how to look past them.

---

## Part 17 — Does the algorithm boost viral trends?

Short answer: **there is no trend detector in the feed path, but there are four velocity mechanisms that effectively boost posts that are already winning — and a shipped "popular topics" module that is switched off.**

### 17.1 The explicit trend mechanisms — both disabled or offline

- **PopularTopicsSource** (home-mixer/sources/popular_topics_source.rs): a dedicated source that fetches "popular topic tweets" per Grok topic (22 hardcoded topics: News, Sports, Music, Memes—), up to 5 per topic, language-filtered, tagged `ForYouPopularTopics`, with its own author-dedup filter. **Its `enable()` hardcodes `false` — shipped but not wired into the pipeline.** Dead code, like the liveness hydrator.
- **GeoPopularTopicsApp** (simclusters/scalding): an offline batch job that ranks topics per country by raw tweet *impression* counts — the closest thing to literal trend detection in the repo. But it's a batch data product; no consumer in the For You feed path is visible in the released code (likely feeds Explore/trends and topic embeddings, not feed ranking).

### 17.2 The four velocity mechanisms that DO boost winners

1. **Live engagement counts are model inputs.** Every candidate's current favorite/reply/repost/quote counts are hydrated (60-second cache) and fed to the model as bucketed features. A post that is rapidly accumulating engagement has high counts → higher predicted engagement → served more → more counts. That's a momentum loop with ~60-second freshness — the engine of "posts that catch fire keep burning."
2. **Engagement-velocity gates in the retrieval index** — the pools of posts the discovery model searches, maintained by the "rankall" system. Index membership is tiered by thresholds-within-a-day: `1fav_1day` (any favorite → 1-day pool), **`32fav_1day` (32 favorites within a day → the "hot" pool)**, `1fav_topic_1day` (favorite + topic membership → topic pool), and video windows out to 5 years. Crossing a threshold inside a day opens more candidate pools — more discovery chances. This is the closest thing to an explicit virality rule in the code: *velocity earns eligibility*.
3. **Viewer-side momentum.** The model's history is the viewer's last 5 minutes of actions; the explicit/implicit engagement signals (15 per type, recency-sorted) drive the SimClusters and Phoenix retrieval sources. Engage with a topic now and similar content is proposed to you immediately — topical momentum, per viewer.
4. **Propagation through similarity.** SimClusters ("people who engage with X engage with Y"), SID topic codes, and followed-topic feeds spread hot content through the engagement-similarity graph; `evergreen_video_*` pools keep hot videos eligible for years.

### 17.3 The anti-trend forces

The same pipeline suppresses some virality channels: out-of-network retweets and replies are filtered out entirely (OONRetweetReplyFilter), repeated reposts deduplicate to the first, posts die at 48 hours, the diversity step zeroes near-duplicate viral posts, and author-diversity decay limits per-author domination.

### 17.4 Net effect

The system boosts **posts with engagement momentum, not topics that are trending**. There is no "this topic is hot right now" signal anywhere in the ranking model — no news-awareness, no trend feature, no hashtag velocity input. When a topic is genuinely hot, the way posts about it win is the unglamorous way: they accumulate engagement fast, the live count features see it within a minute, the 32-favs-in-a-day gate admits them to more pools, and the momentum loop takes over. The platform's one explicit attempt at topic-level trending (PopularTopicsSource) ships disabled — which is itself a statement about where the product believes the value is.

---
## Part 18 - Do replies and posts rank separately?

*"If your replies often do very well or poorly, does that affect the predictions of your posts? Or are they mostly separate?"*

The short answer: **the learned state is shared, but the serving and scoring paths are separate - which is why reply engagement and post reach can look completely disconnected in practice.**

### 18.1 The shared part: one learned state per author, for everything

Every account has exactly one learned reputation (the author embedding), and it is trained on **all** of the account's outcomes - original posts, replies, everything. A reply that gets engaged with is a positive training example for the same author row that conditions your next post's predictions; a reply that is served and ignored (including "recap" notifications - X's notification cards, pushed to people who engaged with the thread - shown and not dwelled on, a dedicated negative training signal in the retrieval model) is a negative example for the same row.

So in the strictest sense: **yes, reply performance affects post predictions** - through the embedding, over training cycles, in both directions. For an account in the trap, replies that generate real engagement are one of the few reliable sources of positive training examples (Part 13.4's recovery lever), because replies can reach audiences that the account's own posts can't (below).

### 18.2 The separate part: four mechanisms decouple observable reply engagement from post reach

1. **Replies ride conversations, not authors.** Replies enter the feed through a different door than posts: an in-network reply (someone you follow replying) is collected **with its ancestors - the parent post and the conversation root** - so a reply by an author you don't follow reaches you when it's part of a conversation with someone you do follow (thunder_source.rs:92-98). Out-of-network replies are removed entirely, and replies with missing parents are dropped (oon_retweet_reply_filter.rs:13-18). The audience a reply reaches is therefore the *conversation's* audience, not the reply author's audience. This is the mechanical reason an account can have "normal" reply engagement while its posts are invisible: the replies are being shown to someone else's followers.

2. **The model has no idea the post is a reply.** The ranking model's entire feature pipeline contains **zero reply-status or ancestor features**. (The only "in-reply" matches in the whole training code are data-loading internals that use the in-reply flag to compute the post-unexplored training label; no reply or ancestor information is ever fed to the model as an input.) To the model, a reply is just another post by its author, scored on its own content (SID, multimedia), its author's embedding, and the viewer's history. The conversation context lives only in *which candidates get proposed* (serving), never in *how they're scored*. So a reply's ranking is not boosted or penalized for being in a big conversation - the conversation only guarantees it a seat at the table.

3. **Scoring is deliberately originals-biased.** Even when replies do reach the pool, they are de-weighted: in-network replies (and reposts) from followed accounts receive the out-of-network discount (0.75) - the system explicitly values originals over conversation participation - and the mutual-follow boost (+15 on reply weight) applies **only to original posts**, never replies or reposts. A reply must outscore an original by a third just to tie.

4. **The recap pathway is a separate channel with its own negative signal.** Replies are pushed through notifications/recaps; "shown but not dwelled" recaps are a dedicated soft-negative (a mildly negative training example) in retrieval training. Reply performance therefore includes a distinct class of negative outcomes (recap fatigue) that original posts don't accumulate.

### 18.3 What "transfer" does and doesn't mean - three levels

*"Does reply performance help future predictions about post engagement?"* The answer depends on the level, and the distinction is worth being exact about:

**Level 1 - Immediate (per request): no coupling.** The score your next post receives is computed from the author embedding *as it currently exists* plus the post's own features. Nothing in the request path reads "this author's recent replies did well" and boosts the post. This is what "your next post still faces your own author state" meant: no same-day, per-post transfer.

**Level 2 - Viewer-side (minutes): a narrow, real coupling.** If a specific viewer engages with your reply, that reply (and your author embedding) enters that viewer's history - and the model's history window is 5 minutes. For the next few minutes, that viewer's predictions about your content are conditioned on having just engaged with you, and your reply also sits in their recent-engagement signals that drive the similar-content discovery source. So there is a real but minutes-scale, per-viewer transfer.

**Level 3 - Learned (days/weeks): yes - the only way the account state ever changes.** Reply outcomes are training examples on the *same* author row as post outcomes, and the row is re-learned from the rolling window of all outcomes. Good replies therefore *do* improve future post predictions - not immediately, but they move the exact state that gates future posts. For a trapped account, replies may even be the *dominant* mover of that row: reply serving doesn't depend on the author embedding (conversations carry them), so a trapped account can accumulate more training examples through replies than through its own invisible posts. The asymmetry that limits the reply lever: the row also accumulates recap not-dwelled negatives, and the learned blend moves at the hidden retraining cadence.

So the precise answer to "better replies won't change post predictions?": **better replies change post predictions exactly as much as better posts do - through the shared learned state - but only at Level 3, which is slow, invisible, and governed by the training schedule. They do not change anything at Level 1.** "The only way to improve post engagement prediction is better posts" is wrong: any outcome that produces positive training signal on your author row improves it. What is true is that no *single* post or reply changes the state today; only sustained outcomes, absorbed across retraining cycles, do.

### 18.4 Practical consequence

- **"My replies get engagement but my posts don't"** is not a mystery in this architecture: the replies are being served to *other people's* audiences through conversation collection, while the posts must be served on the account's own reputation. The reply engagement still feeds the shared embedding (slowly, positively), but it does not transfer as any serving or scoring preference - your next post still faces your own author state, the deboosted reply path, and the 48-hour clock.
- **The reverse is also true**: weak posts do not stop your replies from reaching conversations, because reply serving doesn't depend on your embedding.
- **Caveat on measurement**: the article's datasets measure replies *received* per post, not the account's own reply performance - so the "replies do fine, posts invisible" observation cannot be directly verified from the raw data. It is a serving-path prediction of this architecture, not yet an empirical measurement.

### 18.5 The recovery lever (for accounts in a poor state)

For an account trying to escape a poor state (Part 13/14): replies in large conversations are the most reliable route to (a) get views at all, and (b) accumulate positive training examples on the shared embedding - with two costs to manage: recap not-dwelled negatives, and the fact that reply outcomes re-learn the *same* author row that gates your posts. Replies are not a separate reputation; they are a side door into the shared one.

---

## Part 19 - Does replying to very small accounts hurt you?

*"If you reply to very small accounts, where the only likely engagement is from the author, is that potentially harmful if done a lot? Instead of engaging mostly with larger accounts where the odds of multiple engagements are higher?"*

Short answer: **there is no counter that penalizes "replies to small accounts" by name, but the code routes your replies through different machinery depending on the conversation's size - and the small-conversation path is lower-value, higher-risk, and at volume can genuinely drag on your learned state.** The system's own name for the big-thread path is "blast radius" — the size of the audience a reply could reach — and it literally gates its processing on it. These are the **blast-radius gates**: small threads get LLM spam screening and no reply ranking; large threads get the reverse.

### 19.1 The blast-radius gates (grox, reply_spam/task_filter.py)

Every reply you make is routed through the reply-processing pipeline, and the filters gate on the **follower counts of the person you're replying to (the "in-reply target") and the thread root**:

| Conversation size | Spam screening (automated checking of replies for spam) | Coordinated-spam detection (detection of many accounts acting together) | LLM reply ranking (an AI model ordering replies under a post) |
|---|---|---|---|
| Target AND root both ≤ 120,000 followers (**Sept**; stepped up from 30K) | **Runs** (LLM screens your reply) | Only if root ≥ 1,000 and thread ≥ 2 deep | **Does not run** (reason: `low_blast_radius`) |
| Target or root > 120,000 followers | **Skipped** (reason: `reply_ranking_target`) | May run (root ≥ 1K) | **Runs** - an LLM scores and positions your reply on the thread |

*(The blast-radius thresholds were 15,000 → 30,000 in the August 14th update, then stepped to **120,000** across Aug 17–25 (40K→60K→80K→100K→120K) — task_filter.py, `FOLLOWER_COUNT_THRESHOLD_FOR_SPAM_DETECTION` and `FOLLOWER_COUNT_THRESHOLD_FOR_REPLY_RANKING`. The direction of every effect is unchanged; the small-conversation lane is now much larger.)*

Consequences for you:

1. **Replying to small accounts = your replies get LLM-spam-screened.** The same reply posted in a thread with a big account would be exempt. A false positive publishes a spam label on that reply (suppression). Volume increases exposure: every reply to a small account is one more screening event. This is the sharpest asymmetry in the code: **the follower count of the person you reply to determines whether your reply is treated as suspicious.** (**Sept**: the screened lane now covers everything up to 120K followers — the "big-thread" exemption is 4× narrower than the 30K era.)
2. **Replying to big accounts = your reply gets ranked, not screened.** On threads with a target or root above 120,000 followers, the spam screen is skipped entirely (your "reputation" substitutes for screening) and an LLM reply-ranker decides your reply's position - where a good reply can earn placement in front of a large audience. (**Sept**: reply scores now only move *down* — the first/lowest score sticks, low scores fan out to regional caches faster — and the per-post dedupe was removed, so re-scores land faster.)
3. **Coordinated-spam detection only targets threads with blast radius** (root ≥ 1,000 followers, ≥ 2 levels deep). Tiny threads are below its radar; mid-size threads (root 1K-120K) are the ones watched for coordinated behavior.

### 19.2 The training-signal economics (why volume matters)

Beyond the gates, the expected *value* of each reply as training data on your shared author row (Part 18) scales with the conversation's audience:

- A reply to a small account is served to a handful of people (the conversation's participants, the small account's few followers, recaps to a small set). Outcomes: usually one author engagement and several "shown and not dwelled" recaps.
- A reply to a big post is recap-served to everyone who engaged with the original post - often thousands. Outcomes: many impressions, many dwells/engagements at a similar rate, and proportionally more positive examples.
- Every reply also generates recap not-dwelled events (a dedicated negative training signal). At volume, a habit of replying into small conversations produces a steady stream of low-value outcomes on your row - **the signal-to-noise of your own training data degrades the more you reply into dead-end threads.** Nothing counts the replies; the row just learns "this author's replies get mostly ignored."

### 19.3 The pattern risk (redacted, but real)

The behavior-detection model (bdsm — the inauthentic-behavior classifier, which watches account behavior patterns) has a dedicated **ReplySpamBot head** (one of its behavior classifications) - high-volume replying with low engagement is precisely the signature it is built to recognize (its thresholds are not shipped; **Sept**: the action-count gate is now redacted too — `min_actions_for_enforcement: 999999`, so only the "> 0.5 dominant head" pick logic is verifiable). Combined with the spam-screening exposure above, "lots of replies into tiny conversations" is the one pattern in this analysis that reads as an actual reply-spam profile in the platform's own machinery.

### 19.4 The honest summary

- Not directly penalized, but **structurally disadvantaged**: smaller conversation = smaller recap audience = lower expected positive signal per reply + a real screening risk that big-thread replies don't carry. (**Sept**: "small" now means anything under 120K followers.)
- **Bigger conversations are the better bet on every axis the code exposes**: no spam screening (above 120K), LLM-ranked positioning, larger recap audiences, more positive training examples, and the small chance of an outsized engagement event.
- The nuance that cuts the other way: a single genuine engagement from a small account's author is still a positive example, and replying to small accounts is how you build relationships that may later move you into their (or their peers') in-network reach - the code simply prices that path as low-blast-radius, and the price is screening + weak training signal + the (redacted) pattern risk.
- The strategic conclusion from the code: **if you reply for reach or training value, prefer threads with blast radius; if you reply for relationship, know that the system is screening you in the small-conversation lane it calls low-blast-radius.**

---

## Part 20 - The missing baseline: how the model judges engagement without context

*"The code doesn't seem to take into account the size of the account you engage with? All it sees is: reply to a tiny account = almost no engagement; reply to a large account = lots of engagement. It doesn't compare to a baseline expectation? Essentially you might be penalizing yourself with repeated small-account engagement, even if not done rapidly like spamming?"*

**Confirmed. This is exactly right, and it is one of the most consequential facts in the entire codebase.**

### 20.1 What the model does and doesn't see

For every served impression, the training example is: (viewer, candidate, outcomes). The candidate's features are its content (SID, multimedia), its author's embedding, its own engagement counts, age, and the viewer's features. There is **no conversation-size feature, no parent-author feature, no thread-audience feature, and no reply-status feature at all** (verified: a grep across the entire model code for reply/ancestor/in-reply features returns nothing).

The outcome label is absolute: the viewer engaged (favorite/reply/dwell/etc.) or did not. A "served and ignored" impression is a full-strength zero-target example (a training example where the viewer took no action, so the correct answer for every action is zero) - the released training pipeline assigns example weight **1.0** by default, and the only weight-adjustment mechanisms in the code ship disabled: an NSFW safety filter (`safety_filter_mode = "off"`) and a negative-downsampling reweighting (`negative_downsample_rate = 1`; when set > 1, sampled-away no-action sequences are up-weighted by the rate). There is **no baseline normalization anywhere**: no "this conversation is small, so low engagement is expected," no per-context expected-value adjustment, no no-action down-weighting.

### 20.2 Why this penalizes small-conversation replying

Without the ability to know *why* an outcome happened, the model attributes the outcome to the features it does have: the author embedding, the content, the post's own id, and the viewer. Consider the two cases:

- **Reply to a big post:** served to thousands of diverse viewers. Engagement varies a lot, the viewer features vary a lot, so the model has plenty of viewer-side variance to attribute the outcome to. The author row gets the *average* - which includes a steady stream of positive examples (dozens per thread).
- **Reply to a tiny account:** served to a handful of homogeneous viewers (the conversation's participants). There is almost no viewer-side variance to attribute the outcome to. The model's only explanation for "1 engagement, 4 nothings" is the author (and the content/post-id). **The author row absorbs nearly the entire outcome.**

So small-conversation replies are not merely low-value training examples - they are the most *author-attributed* examples in the system: low absolute engagement, minimal viewer variance to blame, full example weight. Repeated small-account replying therefore produces a consistent stream of "this author's replies produce almost nothing" signal on the shared author row - the same row that gates your posts. The drag is **volume/consistency-based, not speed-based**: it has nothing to do with spam behavior; a patient person replying daily into small threads is building the same statistical profile as a bot would.

### 20.3 The baseline that would fix it (and what exists instead)

A fair system would compare outcomes to an expectation: "1 engagement out of 5 impressions in a 200-follower conversation" is a 20% engagement rate - excellent - and should be a positive signal. This system cannot compute that, because it never receives the conversation size. The only context-normalization that exists in the entire released pipeline is:

- **Log-Q correction (retrieval only):** candidate-popularity correction that down-weights examples of posts shown very frequently. It normalizes by *post frequency*, not by conversation size, and does not apply to the ranking model's main loss.
- **The experimental dwell-regret scoring:** normalizes engagement against the *candidate pool's mean* at scoring time - pool-relative, not context-relative, and off by default.
- **A `sample_weights` column** exists in the data pipeline (defaulting to 1.0). The streaming consumer *does* set it when negative downsampling is enabled (weight = the downsample rate on sequences that are kept, kafkaconsumer.py:318-339) — but the shipped defaults leave both that and the safety-filter weighting off, so out of the box training runs at uniform weight 1.0.

### 20.4 The honest summary

- The user's understanding is correct: the model sees absolute outcomes and attributes them without a baseline, and the attribution lands hardest on the author for small-conversation outcomes.
- The harmful effect is real but *slow and diluted*: the per-post id embedding absorbs some of it (and expires with the 48-80h window), the author's own engagement back is a positive example, and the row is shared across all your outcomes. The effect is a rate, not a switch - consistency over volume.
- The reverse is equally true: big-thread replies generate the positive examples that lift the same row, which is why the strategic conclusion from Part 19 stands - if you reply for reach or for the learned state, prefer threads with blast radius; and know that every reply into a tiny thread is a full-weight training example in which the model can only blame you.

---

## Part 21 - Can liking posts hurt your own predictions?

*"Are there any potentially negative signals based on which posts you like? Can liking a post ever result in some kind of negative result to your account's predictions for posts being shown?"*

Short answer: **no - liking is never a negative signal by itself. A like is the single most positive signal in the system (it is literally the training target of the discovery model). But there are four indirect ways liking can shape your feed negatively, and one important difference from the reply case in Part 20.**

### 21.1 What a like is, everywhere it appears

A favorite appears in five places in the pipeline, all positive:

1. **The retrieval model is trained to predict it.** The discovery model's primary positive target is "the viewer will favorite this post" - the system is literally optimized to find you posts you will like. This is why like-rates correlate most tightly with views (Part 9): they are the same signal.
2. **Your history.** Your likes are action tokens in your 5-minute history window - the model's main input about who you are right now.
3. **The discovery signals.** Your recent favorites (up to 15, recency-sorted) drive the SimClusters source - "find similar content" is fed by what you liked.
4. **The author-similarity path.** Liking an account's posts puts that author's embedding in your history, strengthening the "show me more from authors like this" signal.
5. **The dwell-regret gate.** Your 28-day favorite count is one of 19 features deciding which experimental scoring variant you receive - its coefficient is small and negative (-0.082), but this only selects a scoring mode; it is not a reach adjustment. (**Sept**: the model's dwell *probability* head is separately live at 0.05, so dwell now earns through two terms — the capped continuous one and the probability one.)

### 21.2 The four indirect ways liking can shape your feed negatively

1. **Contradictory actions erase the like.** If you like a post AND also block, mute, report, unfollow, or click "not interested"/"show fewer" on it, the training loss **zeroes every non-negative signal for that post** (recsys_model.py:2781-2813) - the like teaches nothing; the negative dominates. (Same caveat as Part 14.2: this masking ships **off** in the released configs. With the shipped defaults the like still counts as a training signal — but it is one positive example against a negative-heavy outcome, and the retrieval model's hard-negative handling of block/mute/report/see-fewer/unfollow/not-relevant applies regardless.) The most common real-world case: liking an account and then unfollowing it. The like you gave before the unfollow is erased as a training signal, and the unfollow (a negative-feedback action) teaches the model that this author/content predicts negative outcomes for you.

2. **Subsequent negative feedback outweighs likes.** The model learns your propensity from the *net* of your actions per candidate and across similar candidates. Ten "not interested" clicks on meme posts will outweigh one meme like in what the model believes about your meme propensity. Likes are votes; negative feedback is a veto.

3. **The echo-chamber narrowing.** Every like strengthens the "more like this" direction - same authors, same content, same topics. This is not a penalty; it is a diversity loss. Your feed becomes narrower in exactly the directions you like, which is why "liking one thing" can quietly become "seeing that one thing." The explicit/implicit engagement signals feeding discovery have the same effect at the source level.

4. **"Garbage in, garbage out" discovery.** A like carries no quality information - it says "I want more like this," not "this was good." Liking low-quality or engagement-bait content feeds "more like this" into your discovery sources, and the model will happily serve you more of it. The content-safety layer filters rule-breaking content, not low-quality content; your likes decide which low-quality *direction* you get.

### 21.3 The key difference from the reply case

In Part 20, replying to small accounts harmed you *as an author*: the low-engagement outcomes were attributed to your **author row** - the same learned state that gates your posts. Liking has **no author-side effect at all**: when you like something, you are the viewer, not the author, and viewer signals never touch your author row. Your likes cannot hurt your own posts' predictions.

The one caveat is the redacted pattern layer: the inauthentic-behavior model (bdsm) has a **LikeBot head** - high-volume, rapid, or mechanically patterned liking is one of the behavior signatures it classifies (thresholds not published). That path feeds enforcement (challenges, labels), not feed scoring - but it is the only place in the released ecosystem where "liking" is associated with anything negative, and it concerns *how* you like, not *what* you like.

### 21.4 The honest summary

- Liking is never a negative signal. No mechanism exists in which a like itself reduces your predictions or your reach.
- The real risks are contextual: likes combined with negative actions (the negative wins), likes outweighed by later negative feedback (the veto), likes narrowing your feed (the echo chamber), and likes steering discovery toward low-quality content (garbage in, garbage out).
- The 5-minute history window keeps the direct effect of any single like short-lived; the durable effects flow through retraining, where likes are the positive target - which is why, over time, what you like is the single biggest thing you control about what you are served.

---

## Part 22 - How do reposts count toward YOUR engagement?

*"If you repost something and the repost gets engagement or not, is it the same as your own posts?"*

Short answer: **no - a repost is not your post. Engagement on a repost is attributed to the original post and the original author, not to you.** You are the *distribution agent*, not the beneficiary. The code is unambiguous about this.

### 22.1 Whose post the algorithm thinks a repost is

For a repost candidate, home-mixer resolves it to the original before anything else happens (models/candidate.rs:143-149, 167-177):

- The **tweet id sent to the model is the original post's id** (`get_original_tweet_id()` = the retweeted tweet).
- The **author id sent to the model is the original author's id** (`get_original_author_id()` = the retweeted user).
- The **engagement counts hydrated** (favorites, replies, views) are the original post's counts.
- The **Phoenix scores are applied to the original tweet id** (phoenix_scorer.rs:109).
- The reposter appears only as a metadata field (`retweeting_author_id`) alongside the original.

The model therefore builds the candidate's representation from the **original post id and the original author's embedding**. Every prediction for the repost is conditioned on the original author's learned reputation, not yours.

### 22.2 What this means for engagement - and its absence

- **Engagement on your repost trains the original author's row, not yours.** When someone likes, replies to, or dwells on the repost you shared, the training example is (viewer, original post, original author, outcome). Your author embedding receives nothing from it. The original author's reputation is what improves.
- **A repost that gets ignored is also the original author's signal.** The "served and no action" examples are attributed to the original author as well - and, per the Part 20 mechanism, without conversation-size context the model attributes the outcome to the features it sees, which are the original author's.
- **Your own author row is untouched in both directions.** Reposting neither helps nor harms your learned reputation directly. The one thing the repost does for you as an author: nothing. As a viewer, plenty (below).

### 22.3 What the reposter does get (viewer-side only)

1. **The repost action is in your history.** Reposting is an action in your user action sequence - the model learns "this viewer reposts" and shapes your feed accordingly (you are treated as someone who shares content, which correlates with engagement propensity as a viewer).
2. **The repost feeds your discovery signals.** Reposting is an explicit engagement signal that drives the SimClusters source - "find me content like what I share." The original author also enters your history via the shared post, strengthening the "show me more like this author" path.
3. **Nothing on the author side.** The asymmetry with quotes is the cleanest way to see it: a **quote post is yours** - you wrote text, the candidate author is you, and quote engagement trains your row. A **repost is the original author's** - you added nothing, and the credit goes to them.

### 22.4 The serving-side treatment of reposts (what you can observe)

- **In-network reposts are deboosted**: reposts (and replies) from accounts you follow get the 0.75 out-of-network discount - the system explicitly values originals over shares (Part 18.2).
- **Out-of-network reposts are removed entirely** (OONRetweetReplyFilter) - a repost only reaches viewers who follow the reposter.
- **Repeated reposts deduplicate to the first** (RetweetDeduplicationFilter) - only one copy of a viral original appears per feed.
- **The diversity step can zero near-duplicate reposts** of the same original across different accounts.

This is also why the article's observation in Part 7 holds: a big account reposting you doesn't transfer their audience to you - the repost carries *your* original's reputation and is scored per viewer anyway; the reposter's role is amplification, not ownership.

### 22.5 The pattern risk (the one place reposting is "yours")

The inauthentic-behavior model (bdsm) has an **RTBot head** - high-volume, mechanical reposting is one of its behavior classifications (thresholds redacted). As with the LikeBot head (Part 21.3), this path feeds enforcement, not feed scoring, and it concerns *how* you repost, not *what*.

### 22.6 The honest summary

- Reposts are not "your engagement" in any training or scoring sense: the credit - positive and negative - belongs to the original author, and your author row is unaffected by what happens to your reposts.
- The repost is a viewer-side signal: it shapes what you are shown (history, discovery signals, author-similarity) and nothing about what others are shown of you.
- If you want engagement to count toward your own learned reputation, it must happen on your own posts or your quotes. Reposts are a gift to the original author - which is why the platform treats them as a deboosted, deduplicated, filtered distribution channel rather than an authorship act.

### 22.7 Self-reposts: the one repost that IS yours — and why it's still usually a mistake

A repost of your own post is the exception to everything above: the original author is you, so the engagement (and its absence) trains your own row. The mechanical question is whether the repost changes anything about the second chance, and the code says mostly no:

- **The model scores the original.** The repost resolves to the original tweet id before scoring (candidate.rs:143-149), so the model sees the same post embedding, the same content, and the same author state - and post-age features are computed from the original's snowflake creation time (xai-recsys util.rs:392, 472), not the repost's. A self-repost of an old post is scored as an old post.
- **The 48-hour filter passes the repost but the freshness features don't reset.** `AgeFilter` computes age from the candidate's own id (age_filter.rs:16-20) - the fresh repost passes - but the model still sees the original's age, at or past the worst freshness buckets.
- **Seen filters remove it for everyone who saw the original.** PreviouslySeenPostsFilter checks the retweeted id via `related_post_ids_iter` (previously_seen_posts_filter.rs:24-30, candidates_util.rs:13-17), and RetweetDeduplicationFilter keeps only one copy of any original per request (retweet_deduplication_filter.rs:19-26) - so the repost reaches only viewers who missed the first run, and even collides with the original itself if both are candidates.
- **It is deboosted in-network.** The 0.75 OON factor applies to reposts from followed accounts - yours included (ranking_scorer.rs:776-783). Reposting your own post prices it 25% below an original on arrival.
- **The expected outcome is the original outcome.** Same predictions → the same best-predicted audience is re-served → another round of near-zero outcomes at full example weight on your row: the serving loop's second iteration. There is no second-chance mechanism - no freshness reset, no new embedding, no honeymoon, no unexplored credit for the viewers who saw it.
- **The narrow case where it is defensible:** if the original was essentially never served (dead-hour posting, pre-audience, cold-start), the flop carries no information - and the repost does re-enter the retrieval index as a fresh id (`post_creation_1day`, 0-fav window) for 48 hours, reaching new viewers and new followers. That benefit is real but small: the model still scores the original (old-age features, your state) and the OON 0.75 still applies when it is discovered.
- **Even then, two strictly better moves exist in the code:** a **new original** (fresh id → fresh age features, unexplored credit, cold-start eligibility if ≤1K followers, no 0.75 deboost, no dedup interaction) or a **quote of your own post** with added context (a new post, your own text, and quotes are NOT deboosted - the in-network discount applies only to replies and reposts, ranking_scorer.rs:776-783). Both beat the self-repost on every axis the pipeline exposes.
- **Pattern risk:** repeated self-reposts read as mechanical reposting - the RTBot head's signature (Part 21.3's caveat applies; thresholds redacted).

---

## Part 23 - The diversity step, in detail

*"How does it measure the level of diversity? Which post wins in this contest? Older posts? Posts from larger accounts?"*

The "diversity step" is a real algorithm: a **determinantal point process (DPP)** running in the vm-ranker service, after scoring and before the final top-50 selection. It is the last thing that decides which posts survive into a feed, and it can remove posts entirely — including in-network ones.

### 23.1 The inputs: score + content representation

For each candidate, the DPP receives exactly two things:

1. **The post's final score** — the output of RankingScorer (predicted engagement × weights).
2. **The post's content embedding** — a 1,024-number representation of the post's images, video and text together (the same "multimodal embedding" family the model uses). For a repost, the original post's embedding is used. If an embedding is missing, the post gets a **random unit vector** — which in 1,024-dimensional space is effectively orthogonal to everything (see 23.4).

Notably absent: post age, follower counts, author identity, engagement counts. The DPP measures only *score* and *content similarity*. Everything else has already shaped the score by this stage.

### 23.2 How it measures diversity: the geometry

Think of every post as a point in a 1,024-dimensional space, where its position is its content representation. Two posts are "similar" when their points point in nearly the same direction — measured by **cosine similarity** (a number from -1 to 1; 1 = identical content, 0 = unrelated, -1 = opposites). A photo of a sunset and a photo of a sunset at a different beach have high cosine; a sunset photo and a text thread about politics have near-zero.

The DPP builds a matrix from two ingredients:

- **Quality (the diagonal):** each post's score is normalized relative to the best post in the pool, then converted with an exponential curve — the highest-scoring post gets the biggest quality number, and quality falls off smoothly with score.
- **Similarity (the off-diagonal):** the pairwise cosine between every pair of posts' embeddings.

The algorithm then chooses the subset that maximizes a geometric quantity (the *volume* spanned by the chosen points, computed via matrix determinant). Maximizing volume means: **pick high-quality points that are as far apart in content space as possible.** That is the entire definition of diversity here: mutual content dissimilarity, weighted by score.

### 23.3 Which post wins — the exact rule

The selection is greedy, one pick at a time:

1. **The highest-scoring post is always picked first.** Score is king at the start.
2. **Each subsequent pick goes to the post that adds the most *new* information** — the one with the highest "conditional variance" given the posts already chosen (computed by a matrix decomposition). A post whose content is nearly identical to an already-chosen post has almost no remaining variance — its information is already "used up" — so it is skipped even if its score is high. A post orthogonal to everything chosen keeps its full variance and wins.
3. Selection stops at 50 posts, or earlier if every remaining post is effectively a duplicate of something chosen (variance below a tiny threshold).
4. **The losers get score zero** — they are removed from the feed as thoroughly as if a filter had dropped them. This is why the diversity step is a *hard* gate, not a soft re-ordering.

So the winner in a contested pair is determined by the product of both factors: score differences are exponential (a big score gap can beat similarity), but similarity is also exponential — a near-duplicate of a chosen post loses even against a substantially lower-scoring post that is genuinely different. **The feed's second copy of any story is always a distinct-content post, never a same-content post with a slightly different score.**

### 23.4 The three questions, answered directly

**Older posts?** Age is not an input to the DPP. An older post wins exactly when its *score* is high enough — and by this stage, age has already done its work: the 48-hour filter removed everything older, and the model's freshness signals lower the predicted engagement of aging posts. Within the DPP, an older post is treated purely on its score and content distinctness. The DPP has no concept of freshness.

**Posts from larger accounts?** Follower count is not an input either. A large account's post wins only through its score — which comes from per-viewer predicted engagement and the learned author reputation, never from follower count (Part 0, Idea 2). The DPP is the most "meritocratic" stage in the entire pipeline: score plus content, nothing else.

**The wildcard case:** posts missing a content embedding get a random unit vector — effectively orthogonal to every other post — so they are never penalized for similarity and will win on score alone if their score is high. The system deliberately prefers "unknown content" over "similar content" in the diversity contest.

### 23.5 Why it exists, and what it means for you

The DPP exists to prevent the feed from showing five versions of the same thing (the same screenshot from five accounts, five posts about the same breaking story, five similar videos). Its practical consequences:

- **Being the first or most distinctive post on a topic matters** — once the feed picks one version of a story, similar versions are zeroed, regardless of whose they are.
- **Distinct content is structurally promoted** — mixing formats and topics in your posting increases the chance each post survives the diversity contest against your other posts in the same request.
- **The author-diversity decay (Part 8) and the DPP are different tools**: the decay weakens *repeat authors* (per request, by author count); the DPP removes *similar content* (by embedding cosine). A prolific author posting diverse content is only hit by the decay; a single author posting five near-identical posts is hit by both.
- **Score gaps are surmountable but expensive**: because quality is exponential in relative score, outscoring a chosen post's *content twin* by a small margin does not save it — the twin is zeroed for similarity, not for losing on points.

---

## Part 24 — The August 14th, 2026 update: what changed and what it means

A day after the main release, xAI pushed commit `c65aa17`. None of it overturns the analysis above — but four pieces of it are worth knowing, and two of them are the platform's own commentary on exactly the questions this document asks.

### 24.1 The official weights clarification (the most important piece)

The new commit adds comments to `param.rs`, `ranking_scorer.rs`, and the README directly addressing the *interpretation* of the weights:

1. **Weights scale predicted probabilities, not counts.** "One common misinterpretation is that you can read these weight ratios as count equivalences, e.g. the incorrect statement that 'one report cancels 468 likes' — this is incorrect because the weights apply to the predicted probabilities rather than raw counts."
2. **Why report is −234.** "The baseline probability of a Report is more than 1000× lower than a Like, so it's weighted more to allow the prediction to affect the final ranking at all."
3. **On mass blocking/reporting campaigns**: "It's predicting your likelihood of the action, not summing up raw weights on counts. Also, recommendations are personalized, so reports from bad actors will primarily affect recommendations for users who are similar to the bad actors, rather than having the same effect on the post's ranking to everyone." And: "For an account to count in the algorithm's recommendation system, it must take place on a post served in Home Timeline. Directly navigating to a post (i.e., coordinating via groupchat) has no ranking impact."

What this does to this document: Part 14.1's math is unchanged (it was always framed as probability bounds), but the *reading* of it must carry the per-viewer caveat — a "−44.5 report penalty" is not a subtraction from some global score; it is the per-viewer prediction weighted, and the platform's position is that a coordinated report campaign moves *similar users'* rankings, not everyone's. Part 13.2 #2 now carries this counterpoint explicitly. The 14.1 bound "a post with a 19% predicted report probability cannot win" is still correct *for that viewer*; whether it also dies for everyone else is precisely the disputed scope question, and the aggregate data cannot answer it.

### 24.2 The Brazil 2026 election filter (a new hard drop rule)

`home-mixer/filters/brazil_2026_election_filter.rs` (**Sept**: 5,082 lines, **2,388** IDs — test asserts `len() == 2388`; README notes "Account list updated August 27, 2026"): a **pre-scoring filter** wired into the For You pipeline between `AuthorSocialgraphFilter` and `VideoFilter` (phoenix_candidate_pipeline.rs:360-369), implementing Brazilian electoral law (as announced by X on August 14th). It drops:

- posts **by** any account in a hardcoded list of **2,388 user IDs** reported to Brazil's Electoral Court for the 2026 election (IDs obfuscated, usernames in comments — e.g. @FlavioBolsonaro, @CarlosBolsonaro, @geraldoalckmin)
- **retweets of** and **quotes of** listed accounts
- **replies whose conversation ancestors include** a listed account

…with one exemption: **viewers who explicitly follow the listed account still see its posts** (`is_excluded_author = listed && !followed`). There is no feature switch — the filter is unconditionally in the pipeline. This is the first example in the open code of a jurisdiction-specific, legally mandated visibility rule, and it is a reminder that the "binary drops" layer (Part 0, Idea 4) can change without any change to the scoring model. (Note: the README's filter table was not updated to include it in this commit — the code has 18 pre-scoring filters, the README lists 17.)

### 24.3 New model input features (relationship flags, and a stale-post flag)

The serving engine (`phoenix/crates/common/xai-recsys/src/util.rs`) now stamps three new boolean features:

- **`IS_AUTHOR_FOLLOWED_BY_VIEWER`** — does the viewer follow this candidate's author? (candidate and history tokens)
- **`IS_AUTHOR_FOLLOWING_VIEWER`** — does the author follow the viewer? (candidate and history tokens) — the mutual-follow signal, now a *model input* in addition to the +15 scoring boost
- **`IS_STALE_POST14D`** — candidates older than ~14 days (`STALE_POST_14D_TTL_SEC = 1,213,200`, shipped since Sept) get their engagement counts **zeroed** and this flag set (**enabled in `home_direct_packed` since Sept**, xrecsys.py:272)

Two transparency wrinkles, both **closed in Sept**: the feature *indices* are now shipped (`feature_config.py:42-44`, `STALE_POST_14D_TTL_SEC = 1,213,200` at :74) and `home_direct_packed` enables `enable_stale_post` (xrecsys.py:272) — only the production on/off decision beyond the shipped config remains unpublished.

What this means for earlier parts: Part 13.2's "we checked every input" list is updated — the follow-direction flags are real model inputs now (relevant to Part 0 Idea 2's "follower count is never used in scoring": follower *counts* still aren't, but follower *relationships* now are). Part 13.3's "the only post-time feature is post age" gains the 14-day staleness qualifier. For the mutual-follow lever (Part 11 #4), the news is good: the boost is no longer only a post-hoc multiplier; the model itself now sees who follows whom.

### 24.4 Smaller changes

- **Blast-radius thresholds raised, then raised again**: grox's reply-spam-screening and reply-ranking gates moved from 15,000 to **30,000 followers** in the Aug 14 commit, then stepped to **120,000** across Aug 17–25 (`FOLLOWER_COUNT_THRESHOLD_FOR_SPAM_DETECTION` and `FOLLOWER_COUNT_THRESHOLD_FOR_REPLY_RANKING`, task_filter.py:17,185). Part 19's numbers are updated; the *direction* of every effect is unchanged (small-conversation replies are still screened and unranked; big-thread replies still ranked and unscreened) — the small-conversation lane is now much larger.
- **Cold-start Thompson sampling** (`EnableColdStartThompsonSampling`, off by default): an alternative picker for the cold-start slot that draws a Beta reward per eligible post and promotes the highest-scoring post among the **top-2** sampled rewards (**Sept**: `ColdStartTsTopK` 5→2) — i.e., among statistically indistinguishable low-data posts, the least-proven one occasionally gets the slot. When off (the default), behavior is exactly the deterministic best-score pick this document describes. **Sept**: eligibility now counts **Home-Timeline** views (`view_count_on_home`).
- **Retrieval index**: the SID pipeline gained a `video_14day` window in Aug (now 10 SID windows); **Sept rebuilt the video indexing** — Main 14→17 and SID 10→15 windows with age-banded `1fav_video_*` slices, grok pool removed, min-age gating with prefloor sidecars, unconditional `1fav` emission, and the training EVERGREEN corpus moved to `video_4to14day`.
- **bdsm**: the enforcement-note mechanism is now visible in structure — a note fires when the dominant bot head scores > 0.5 over a policy-defined action count, with the head named; **Sept: the count gate (`min_actions_for_enforcement`) is itself redacted** — it ships as `999999` (was 30 in the Aug tree), so the "> 0.5" pick logic is verifiable but the real action-count gate is withheld. The note *templates* themselves are redacted (sentinel `"<redacted>"`). Other thresholds remain hidden.

**Net effect on this document's conclusions: none overturned.** The weights clarification reframes Part 14.1's interpretation (per-viewer, not per-count); the Brazil filter adds a genuinely new, jurisdiction-specific drop rule outside the scoring model; the follow-direction features strengthen the mutual-follow lever; the grox threshold change enlarges the small-conversation lane. The mechanisms, the data, and the equilibrium story all stand.

---

## Part 25 — The September 2nd, 2026 update: what changed and what it means

Thirteen commits between Aug 17 and Sept 2 (`b089ce6`…`85ac72a`) moved the code substantially further. As with the August update, **none of it overturns the analysis above** — but it changes numbers, removes one scoring mode, enables three model features, rebuilds video indexing, and redacts one previously-visible threshold. The affected earlier parts are updated in place; this part collects the changes and their meaning.

### 25.1 Scoring weights moved (Aug 25 commit)

`home-mixer/params/param.rs` (sync stamp `2026-09-02`): `video_open` 0.05→**0.07**, `vqv` 0.05→**0.0**, dwell **probability** head 0.0→**0.05** (continuous `cont_dwell_time` unchanged at 0.004). Net effect on the Part 14.1 bounds: the all-positive max moves ≈43.4→**≈43.5** (≈58.5 mutual) — the 19%-report / 8.4× asymmetry is essentially unchanged. Conceptually larger: the VQV head no longer scores at all (the 10-second gate survives only as an implicit discovery signal), while "watch time" now earns through *two* terms. Parts 1.G, 2, 9, 11, 14.1, 16.2 updated.

### 25.2 MPN scoring removed; VMRanker slimmed (Aug 25 commit)

`EnableMpnScoring`, `MpnParts`, per-head weight sending, and the VMRanker fold/author-diversity override are **deleted**; the replacement is a narrower `MultiplierPreOffset` gate (default off), and the VMRanker request is now just tweet id + retweeted id + score with hardcoded `value_model_id="dpp"` (vm_ranker.rs, 118 lines; no follower counts, head weights, slate context, or author fields). Consequences: one fewer "off by default" mode to track; the DPP hard-cap behavior itself is unchanged (non-selected still score 0.0); and follower count now reaches the ranker through *no* path at all — the old "`author_followers_count` → VMRanker" line in Parts 0.2, 13.3, and 16 is gone. On retweets the model additionally no longer receives `is_following_user`/`followers` (candidate.rs). The reverse-chron/Following pipeline, by contrast, *gained* capability: quoted/ancestor-text mute matching, author-blocks-viewer enforcement, its own VF hydrator, and `retweeted_user_id` plumbing from night-owl sources.

### 25.3 The in-network reply/retweet deboost is gated (verified, not new)

Reading the current scorer closely: the 0.75 discount hits in-network replies/reposts **only when `EnableOonRescoreForInNetworkRepliesRetweets` is on** (ranking_scorer.rs:677-687). The README states it unconditionally; prod's value is unpublished. Parts 1.A, 5.3, 6, 18, 22 updated to carry the gate.

### 25.4 Grox blast-radius gates stepped to 120K (Aug 17–25 commits)

`FOLLOWER_COUNT_THRESHOLD_FOR_SPAM_DETECTION` and `FOLLOWER_COUNT_THRESHOLD_FOR_REPLY_RANKING` (task_filter.py:17,185) stepped 30K→40K→60K→80K→100K→**120K**. The "big-thread" exemption is 4× narrower than the 30K era — the screened lane now covers everything up to 120K followers. Same commits: reply scores only move *down* (first/lowest sticks, fanned out regionally), the per-post annotation dedupe was removed (more LLM volume), the pipeline re-sourced to the v3 unified-posts topic, and reply/PTOS LLMs now see follower counts in prompt context. PTOS safety was overhauled in parallel: deluxe adult-Hard now needs safemodel+PTOS agreement (or a Grok-4.5 judge), ChildSafety fails closed to NoViolation, ViolentMedia fails soft to GraphicMedia, high-fav video safety runs a dedicated motion-reveal-gated lane (currently off — `enable_motion_reveal=False`), and a priority-post lane feeds banger scoring. Parts 19, 12.3, and the grox constants updated.

### 25.5 Brazil list 665 → 2,388 (Aug 25 / 28 / Sept 1 commits)

The filter logic is unchanged (follower-exempt author/retweet/quote/ancestor drops, no feature switch), but the list grew 665→2,328→2,315→**2,388** (test asserts `len() == 2388`; file now 5,082 lines with an Art. 28 §1-A legal citation, "ids obfuscated" note, exclusions, and "no live account found" notes). The README's only Sept change is *"(Account list updated August 27, 2026.)"* — its 17-filter table still misses the filter entirely. Part 24.2's numbers updated.

### 25.6 Rankall rebuilt for video (Aug 25 commit)

Main 14→**17** / SID 10→**15** windows: four age-banded `1fav_video` windows each (fav'd immersive videos fan out to both streams), `evergreen_video_grok_30day` deleted from both, bounded `video_4to14day`/`nsfw_video_4to14day` in SID, min-age gating with `prefloor_*` sidecars (young posts held back, restart-safe), unconditional `1fav` emission (0-fav race closed), 0-fav multimodal-metadata coverage, and the v3 fresh-post topic. The training EVERGREEN corpus moved `evergreen_video_1825day`→`video_4to14day` (retrieval_dataset.py:255-259). Net for Parts 1.G, 3, 9, 13.8, 17: the video-maturity ladder is finer-grained; the "0-fav" hole is smaller; nothing about the 1-fav/32-fav gates changes.

### 25.7 Training: Muon + enabled count/follow/stale features (Aug 19–Sept 1 commits)

Dense training moved AdamW→**Muon** (H100 lr 7.1e-4, GB300 5e-4); embeddings lr 0.2→**0.28**, half-lives explicit (2500 H100 / 5000 GB300) with lazy decay + small weight decay — rows now also *shrink*, which is recovery-friendly on top of the accumulator decay. `home_direct_packed` now learns from **live engagement counts** (log2(1+c)/32 → MLP-64 embedding path, recsys_feature_prep.py:628-656), **both follow directions**, and the **stale-post flag** (`enable_stale_post: True`, xrecsys.py:272; `STALE_POST_14D_TTL_SEC = 1,213,200`, feature_config.py:74) — the bool indices are now shipped, closing the Aug "internal config" wrinkle. Retrieval attention normalized + uncapped (`qk_norm` on, logit cap off); a separate `xrecsys_search` model config exists (search, not For You). The user-facing reading: velocity now shapes *predictions* directly (strengthening the Part 17 momentum loop), the mutual-follow lever is now three layers deep (boost + dwell + model feature), and posts older than ~14 days present zeroed counts to the model. Parts 5.1, 11, 13–15 updated.

### 25.8 Enforcement + VF: one new rule, one new path, one redaction

Enforcement gained **`panda_reports_embedding_v10_rough_spam` → suspend** and a GrowthBook-gated **`requested_actions` generic-action path** (model-requested suspend/label/challenge; inert until allowlisted; shields still win), a `decisions_json` mirror, and scarecrow's **XReview human-review reporting** — while the VF `rules/` consolidation (`tweet_rules.rs` + `author_rules.rs`, same 28+26 rules/order, golden-corpus pinned) is behavior-preserving except that the 16 NSFW-gating countries are now **runtime-configurable**. And **the bdsm action-count gate is now redacted**: `min_actions_for_enforcement: 999999` (was 30) — the "> 0.5 dominant head" logic is still verifiable, the count is withheld. Parts 6, 12.4–12.5, 13.4, and the Part 13.2 follower-exemption precision updated.

**Net effect on this document's conclusions: none overturned.** The weight moves are second-decimal; the removed MPN mode shipped off; the 120K gates enlarge the small-conversation lane without changing its direction; the Brazil growth is list maintenance, not new logic; the rankall rebuild refines video maturity without touching the 1/32-fav gates; the training changes strengthen the momentum loop and the recovery story (explicit decay + weight shrink + enabled stale flag). The mechanisms, the data, and the equilibrium story all stand.

---

## Glossary - terms used in this document, in plain English

The document deliberately uses the platform's own terminology where the code does. This glossary defines every technical term used here, without assuming knowledge of X or of recommendation-system code.

- **Action**: one of the measurable things a viewer can do with a post (like, reply, repost, quote, share, bookmark, follow the author, report, block, mute, spend time on it, and more). The model predicts a probability for each action.
- **Ancestors**: in a reply chain, the posts your reply is answering - the parent post and the conversation root (the first post in the thread).
- **Author embedding**: an account's learned reputation - a list of numbers summarizing what has historically happened when this account's posts were shown to people. Shared by everything the account posts; changes only through retraining (see Part 0, Idea 3, and Part 13).
- **Blast-radius gates**: mechanisms that treat replies differently depending on how large an audience the conversation could reach (measured by the follower counts of the person you reply to and the thread's root author). Small threads: your replies get LLM spam screening and no reply ranking. Large threads: no spam screening, and an LLM ranks your reply's position instead. "Blast radius" is the code's own term for the audience a reply could reach.
- **bdsm**: the inauthentic-behavior detection system - a model that watches account behavior patterns (bot-like following, liking, replying) and classifies accounts into types including bot patterns. Its operating thresholds are not published.
- **Bookmark**: saving a post for later. One of the measured actions; bookmarks also feed the discovery signals that find you similar content.
- **Candidate**: a post being considered for a viewer's feed in a single request. Each request assembles roughly a thousand or more candidates and keeps about 35-47.
- **Checkpoint**: a saved snapshot of everything the model has learned. Production predictions come from the latest checkpoint until the next one is trained and rolled out.
- **Coordinated-spam detection**: automated detection of many accounts acting together (for example, amplification networks); the code runs it only on threads with a large root author.
- **Credibility score (user-cred)**: a daily-computed 0-100 "trust" number per account, derived from PageRank-style math over the follow and engagement graph. It does not affect feed delivery; it gates enforcement (spam labels, suspensions, challenges) - high scores shield accounts.
- **Diversity step (DPP)**: a post-scoring stage that prevents the feed from filling with similar items. It compares posts by the similarity of their content representations (multimodal embeddings), always keeps the highest-scoring post, then picks the highest-scoring posts that are most *different* from the ones already chosen; posts too similar to chosen ones are removed entirely (their score becomes zero). See Part 23.
- **Dwell**: time spent viewing a post - a measured quantity. "Not dwelled" means it was shown but no time was spent on it.
- **Embedding**: a list of numbers that represents something (an account, a post, an image) to the model. Similarities and patterns in these numbers drive predictions.
- **Engagement**: any measurable action on a post - in this document's datasets: like, reply, repost, quote, bookmark.
- **Engagement rate**: interactions divided by views, measured on the audience the system chose to show the post to. A symptom of who was served, not a cause of reach (Part 0, Idea 5).
- **Example weight**: a multiplier controlling how strongly one training example influences the model. The released pipeline runs at 1.0; the only reweighting mechanisms (an NSFW safety filter and negative-downsampling) ship disabled.
- **Evergreen video pools**: retrieval candidate lists that keep certain videos eligible for distribution for years, versus roughly 48 hours for normal posts.
- **Filter**: a pipeline step that removes posts from consideration (too old, already seen, blocked author, muted keyword, and so on).
- **For You feed**: X's main algorithmic timeline, assembled per request.
- **grox**: the content-understanding service (LLM-based classifiers) - for example, spam screening of replies and post-quality scoring.
- **Head**: one of the model's outputs - a prediction (a probability or a quantity) for a specific action. The "reply head" predicts the probability the viewer will reply; the "report head" predicts a report; the "dwell time head" predicts seconds spent.
- **Hit concentration**: the share of an account's total engagement that comes from its few biggest posts. High hit concentration means spikes; low means uniform performance.
- **In-network / out-of-network (OON)**: in-network means posts from accounts you follow; out-of-network means everyone else. Out-of-network posts receive a score discount. (**Sept**: in-network replies/reposts are discounted too, but only when a dedicated switch is on.)
- **Label**: a marker applied to a post or account (spam, NSFW, abusive, and so on) that triggers visibility rules - drops, interstitials, or restrictions. Labels expire on their TTL (below).
- **Momentum loop**: the self-reinforcing cycle where a post's engagement raises its predicted engagement, which gets it served more, which raises its engagement.
- **Multimodal embedding**: a representation of a post derived from its images, video, and text together.
- **Negative-feedback actions**: viewer actions that express dislike - block, mute, report, "not interested," "show fewer," unfollow, mute conversation, "not relevant."
- **PageRank**: a classic algorithm that assigns importance to nodes in a graph by how much other important nodes point to them (originally web pages; used here over the follow graph).
- **Probability**: the model's outputs are probabilities from 0 to 1 - how likely the viewer is to take each action on a given post.
- **Recap**: a notification card X shows you (for example, "X replied to a post you engaged with"). Replies are pushed to conversation participants this way. "Shown but not dwelled" recaps are a negative training signal.
- **Ranking model**: the model that scores candidate posts for a viewer - predicts the per-action probabilities - and orders them.
- **Retrieval model**: the discovery model that finds out-of-network candidates - it searches large pools of posts using content and author similarity.
- **Retrieval index**: the pools of posts the retrieval model searches; membership is tiered by engagement thresholds reached within a day.
- **SID (semantic ID)**: a compact code for a post's content; posts with similar content share code prefixes - a learned topic taxonomy.
- **Serve / served**: shown to a viewer. "Served impressions" are the times a post was actually shown.
- **SimClusters**: a discovery system built on "people who engage with X also engage with Y" clusters.
- **Slot**: one of the roughly 35-47 positions in a feed response.
- **Soft-negative**: a training example treated as mildly negative (for example, a recap shown but not opened), versus a hard negative (for example, a block).
- **Spam screening**: automated, LLM-based checking of replies for spam before they are allowed normal visibility.
- **TTL (time-to-live)**: how long something lasts - a label, a stored value, a window.
- **Training example**: one record of "this viewer was shown this post; here is what they did."
- **Training window**: the recent slice of history the model learns from when it is retrained; older events age out. Its length is not published.
- **user-cred**: see credibility score.
- **Views per follower (V/F)**: median views divided by followers - the efficiency measure used throughout the article.
- **Visibility filtering (VF)**: the separate service that decides, per post and viewer, whether a post can be shown at all (allow / show behind an interstitial / drop), using labels, blocks and mutes, and viewer settings.
- **Weights**: numbers multiplying each predicted probability in the final score (for example, reply 5.0, report -234). The values are published in the code.
- **Zero-target example**: a training example where the viewer took no action, so the correct answer for every action is zero.

---

## The bottom line

The article's anomalies are real, its data is reproducible, and its skepticism is warranted. The open code explains them without needing to invoke malice: **reach is the product of per-person predictions; follower count doesn't enter the equation; every account carries a self-reinforcing learned reputation; the feed is a competition for a handful of seats; and the engagement you see is a symptom of who was shown your post, not a cause of who is.** The winner-take-all distribution the article documents is the expected output of a system engineered to concentrate attention on predicted engagement.

Whether that system is wise, fair, or good for public discourse is a separate question — and the code's answer to that question is: it was not designed with that question in mind.

*Current as of the Sept 2, 2026 tree (commit `85ac72a`); Part 25 collects the September changes.*

<!-- REVIEW-BASELINE-FOOTER rev=85ac72a1bba41f21615e3f0bca56da75970a6633 short=85ac72a date=2026-09-02T19:46:48Z -->

