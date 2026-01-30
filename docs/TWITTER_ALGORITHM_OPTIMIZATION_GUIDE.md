# Twitter Algorithm Optimization Guide

## How to Write Tweets That Go Viral and Increase Your Account Visibility

This guide is based on an in-depth analysis of Twitter's (X's) open-source recommendation algorithm. It reveals the exact factors that determine tweet visibility and account reputation scores.

---

## Table of Contents
1. [Understanding the Algorithm](#understanding-the-algorithm)
2. [Tweet Ranking Factors](#tweet-ranking-factors)
3. [TweepCred: Your Reputation Score](#tweepcred-your-reputation-score)
4. [Engagement Metrics That Matter](#engagement-metrics-that-matter)
5. [Hidden Techniques from Source Code](#hidden-techniques-from-source-code)
6. [Actionable Tips for Maximum Visibility](#actionable-tips-for-maximum-visibility)
7. [What to Avoid](#what-to-avoid)

---

## Understanding the Algorithm

Twitter's For You timeline uses a multi-stage recommendation system:

1. **Candidate Sourcing** - Filters ~1 billion tweets down to a few thousand candidates
2. **Feature Hydration** - Fetches ~6,000 features for each candidate tweet
3. **ML Ranking** - Scores tweets using a neural network model
4. **Filtering & Heuristics** - Applies diversity, quality, and safety filters
5. **Final Mixing** - Combines tweets with ads, recommendations, and other content

Your goal: Optimize for **every stage** of this pipeline.

---

## Tweet Ranking Factors

### Primary Engagement Signals (Most Important)

The algorithm ranks tweets based on **4 core engagement types**:

#### 1. **Favorites (Likes)** ⭐
- **Weight**: Configurable (default: varies by experiment)
- **Impact**: High - Direct positive signal
- **Signal Name**: `PredictedFavoriteScoreFeature`

#### 2. **Retweets** 🔄
- **Weight**: Configurable
- **Impact**: Very High - Strongest amplification signal
- **Signal Name**: `PredictedRetweetScoreFeature`
- **Note**: Quote tweets are tracked separately and also valued

#### 3. **Replies** 💬
- **Weight**: Configurable
- **Impact**: High - Shows conversation value
- **Signal Name**: `PredictedReplyScoreFeature`

#### 4. **Dwell Time** ⏱️
- **Weight**: Configurable
- **Impact**: Medium-High - Time spent reading
- **Signal Name**: `DwellParam`
- **Details**: Measures how long users spend viewing your tweet

### Secondary Engagement Signals

#### Video Quality View (VQV)
- Immersive video engagement
- Measured by watch time and completion rate
- **Signal**: `VideoQualityViewScoreFeature`

#### Good Clicks
- Profile clicks leading to engagement
- Tweet detail views with dwell time
- Indicates genuine interest

#### Bookmarks & Shares
- Bookmark actions (save for later)
- Share via DM or external apps
- Shows high-value content

### Content & Author Features

1. **Media Presence**
   - Images boost engagement
   - Videos (especially high VQV videos) get priority
   - Multiple images can increase visibility

2. **Tweet Length**
   - Algorithm categorizes by length
   - Longer tweets may get dwell time boost
   - Too long can reduce engagement

3. **Author Verification Status**
   - Blue verified accounts
   - Gold verified (organizations)
   - Gray verified (governments)
   - Creator subscription status

4. **Recency**
   - Recent tweets get higher priority
   - Time decay factor: **0.95** (older tweets progressively lose visibility)

---

## TweepCred: Your Reputation Score

TweepCred is your **account reputation score** calculated using a **Weighted PageRank algorithm**. It's one of the most important factors for visibility.

### How TweepCred is Calculated

Based on source code analysis (`UserMass.scala`, `Reputation.scala`):

#### Account Age Factor
```
Age Weight Formula:
- < 1 day: 0.0
- 1-7 days: 0.1
- 7-30 days: Linear scaling (0.1 to 1.0)
- 30+ days: 1.0 (full weight)

Base Score = 0.1 + (0.5 × device_weight × age_normalization)
```

**Key Insight**: New accounts have severely reduced visibility for the first 30 days.

#### Safety Status Multipliers

From `UserMass.scala` lines 45-46:

- **Verified accounts**: Score = **100** (maximum)
- **Normal accounts**: Calculated via PageRank
- **Restricted accounts**: Score × **0.1** (90% penalty)
- **Suspended accounts**: Score = **0** (no visibility)

#### Follower-to-Following Ratio Penalty

**Critical Hidden Factor** from `adjustReputationsPostCalculation`:

If you follow >500 accounts AND your following/followers ratio >0.6:

```
Division Factor = exp(5.0 × (ratio - 0.6))
Adjusted Score = Original Score / Division Factor
```

**Example Penalties**:
- Ratio 0.6 (600 following, 1000 followers): 1.0× (no penalty)
- Ratio 0.8 (800 following, 1000 followers): 2.7× penalty
- Ratio 1.0 (1000 following, 1000 followers): 7.4× penalty
- Ratio 1.5 (1500 following, 1000 followers): 55× penalty
- Ratio 2.0 (2000 following, 1000 followers): 403× penalty

**Action Item**: Keep your following/followers ratio below 0.6 to avoid exponential penalties.

#### PageRank Network Effects

Your TweepCred is influenced by:
- Quality of accounts that follow you (PageRank scores propagate)
- Quality of accounts you interact with
- Network position in the social graph
- User interactions (mentions, retweets, etc. create weighted edges)

**Optimization**: Get followed by high TweepCred accounts (verified, influential users).

---

## Engagement Metrics That Matter

### In-Network vs Out-of-Network

The algorithm distinguishes between:

1. **In-Network Engagement**: Actions from your followers
   - `InNetworkFavoritesCount`
   - `InNetworkRetweetsCount`
   - `InNetworkRepliesCount`
   - **Higher weight** - Shows your existing audience finds content valuable

2. **Out-of-Network Engagement**: Actions from non-followers
   - Enables content discovery
   - Signals broad appeal
   - Required for viral growth

### User Signal Service (USS) Features

From `RETREIVAL_SIGNALS.md`, the algorithm tracks:

| Signal | Used For | Impact |
|--------|----------|--------|
| Author Follow | Features/Labels in multiple models | High |
| Tweet Favorite | Features/Labels across all systems | Very High |
| Retweet | Features/Labels in TwHIN, UTEG, FRS | Very High |
| Quote Tweet | Features/Labels | High |
| Tweet Reply | Features across systems | High |
| Tweet Click | Features/Labels for Light Ranking | Medium |
| Video Watch | Features/Labels | High (for video) |
| Tweet Bookmark | Features only | Medium |
| Tweet Share | Features for FRS | Medium |
| **Negative Signals** | | |
| Author Unfollow | Features | Negative |
| Tweet Unfavorite | Features | Negative |
| Tweet "Don't like" | Features | Strong Negative |
| Tweet Report | Features | Very Strong Negative |
| Author Mute | Features | Strong Negative |
| Author Block | Features | Very Strong Negative |

---

## Hidden Techniques from Source Code

### 1. Score Weight Configuration

From `ScoredTweetsProductScoringWeightRegistry.scala`:

The algorithm uses **configurable weights** ranging from **-10,000 to +10,000**:

```scala
// Positive Engagement Weights
FavWeight: Range(-10000, 10000)
RetweetWeight: Range(-10000, 10000)
ReplyWeight: Range(-10000, 10000)
DwellWeight: Range(-10000, 10000)
VideoQualityViewWeight: Range(-10000, 10000)

// Negative Engagement Weights
NegativeFeedbackV2Weight: Range(-10000, 10000)
ReportWeight: Range(-10000, 0)      // Default: -1000
BlockWeight: Range(-20000, 0)       // Default: -20000
MuteWeight: Range(-10000, 0)        // Default: -1000
```

**Key Insight**: Negative actions (reports, blocks) have **20× the weight** of single positive actions. Avoid content that triggers negative feedback.

### 2. Score Aggregation Formula

From `RerankerUtil.scala`:

```
Final Score = Σ(engagement_score × weight) + epsilon

Where:
- epsilon = 0.001 (smoothing factor)
- Scores normalized per request
- Rank decay factor = 0.95
```

### 3. Content Balance Heuristics

The For You timeline enforces:
- **In-network vs Out-of-network balance** (typically 50/50)
- **Author diversity** - Limits tweets from same author
- **Feedback fatigue** - Reduces visibility of similar content types
- **Deduplication** - Filters previously seen tweets

### 4. Candidate Source Distribution

From `home-mixer` documentation:
- ~50% from Search Index (In-Network)
- ~50% from Out-of-Network sources (Tweet Mixer, UTEG, FRS)

**Optimization**: To go viral, you need BOTH strong in-network engagement AND out-of-network appeal.

### 5. Feature Thresholds

From source analysis:
- **Negative feedback threshold**: 0.15 (normalized score)
- **Minimum engagement threshold**: 0.001 (constant)
- **Video quality minimum**: Varies by model

---

## Actionable Tips for Maximum Visibility

### For Tweet Creation

#### 1. **Optimize for All 4 Primary Metrics**

Don't just aim for likes. Design tweets to get:
- ❤️ **Likes**: Make it agreeable, relatable, funny, or insightful
- 🔄 **Retweets**: Make it shareable, quotable, useful
- 💬 **Replies**: Ask questions, spark discussion, be controversial (carefully)
- ⏱️ **Dwell Time**: Use thread formats, compelling stories, detailed insights

#### 2. **Use Rich Media**

- Include **images** on most tweets (higher engagement)
- Use **video** when possible (VQV scoring boost)
- For videos: Optimize for watch time and completion rate
- Multiple images can work but don't overdo it

#### 3. **Tweet Timing & Frequency**

- **Recency matters**: Tweet when your audience is active
- Don't spam: Author diversity filter will suppress multiple tweets
- Space out tweets to maintain visibility
- Thread continuation tweets can bypass some diversity filters

#### 4. **Length & Format**

- **Not too short**: Need enough content for dwell time
- **Not too long**: People won't read walls of text
- **Sweet spot**: 100-280 characters for engagement, or multi-tweet threads
- Use **line breaks** for readability (improves dwell time)

#### 5. **Engagement Bait (Use Carefully)**

Direct calls to action:
- "Retweet if you agree"
- "Drop a reply with your thoughts"
- Polls (high engagement)
- Questions that invite responses

**Warning**: Overuse can trigger spam detection. Use naturally.

#### 6. **Network Effects**

- Reply to high-engagement tweets in your niche
- Quote tweet with added value (not just "This!")
- Engage with accounts that have high TweepCred
- Build relationships with influential accounts

### For Account Optimization

#### 1. **Maintain Healthy Follower Ratio**

**CRITICAL**: Keep following/followers ratio below 0.6

- If ratio >0.6 with >500 following: **Exponential penalty**
- Unfollow inactive or low-value accounts
- Focus on quality followers over quantity following
- Example: 1000 followers? Don't follow more than 600

#### 2. **Age Your Account Properly**

- New accounts (<30 days): **Severely limited visibility**
- Build gradually: Don't spam when new
- Focus first 30 days on:
  - Building follower base
  - Establishing posting patterns
  - Creating quality content
  - Avoiding negative signals

#### 3. **Avoid Negative Signals**

Actions that hurt your TweepCred:
- Being muted (signals low-quality content)
- Being blocked (signals problematic behavior)
- Getting reported (very damaging)
- Being unfollowed frequently (churn signal)
- Having tweets marked "Don't like" (poor content)

**One report or block = -20,000 weight score**. Avoid at all costs.

#### 4. **Get Verification**

From source code: Verified accounts get **TweepCred = 100** (maximum score)

Benefits:
- Instant maximum reputation
- Higher baseline visibility
- Trust signals across all models
- Better For You placement

#### 5. **Build High-Quality Network**

- Get followed by verified accounts (PageRank boost)
- Get followed by high-engagement accounts
- Interact with quality accounts
- Build genuine community

#### 6. **Consistent Engagement Patterns**

- Regular posting schedule
- Consistent engagement with followers
- Authentic interactions (algorithm detects fake engagement)
- Build history of positive signals

### Content Strategy

#### 1. **Understand Your Niche**

The algorithm uses:
- SimClusters (community detection)
- Topic embeddings
- Interest classification

**Strategy**: 
- Stay consistent within topics
- Build authority in specific areas
- Algorithm will recommend you to interested users

#### 2. **Engagement Velocity Matters**

- Early engagement boosts visibility
- First hour is critical
- Seed tweets with engaged community
- Post when followers are active

#### 3. **Thread Strategy**

- Threads can bypass some diversity filters
- First tweet is most important (hook)
- Keep each tweet engaging (dwell time on each)
- End with CTA (call to action)

#### 4. **Test and Iterate**

Different weight configurations in A/B tests mean:
- What works can change over time
- Test different formats
- Analyze your best-performing tweets
- Double down on what works

---

## What to Avoid

### Content to Avoid

1. **Spam patterns**
   - Excessive hashtags
   - Repetitive content
   - Link spam
   - Mass mentions

2. **Negative feedback triggers**
   - NSFW content (without proper marking)
   - Harassment or abuse
   - Misinformation
   - Scams or manipulation

3. **Engagement bait overuse**
   - "Like and RT" spam
   - Follow-for-follow schemes
   - Artificial engagement
   - Bot-like behavior

### Behaviors to Avoid

1. **High Following/Followers Ratio**
   - Following >500 with ratio >0.6 = exponential penalty
   - Mass following strategies backfire

2. **Aggressive Automation**
   - Fake engagement (detected)
   - Bot-like posting patterns
   - Automated replies (low quality)

3. **Negative Interactions**
   - Getting into heated arguments
   - Attacking other users
   - Controversial for sake of controversy
   - Triggering reports or blocks

4. **Inconsistent Activity**
   - Long gaps in posting
   - Sudden spam bursts
   - Erratic behavior patterns

---

## Summary: The Viral Tweet Formula

Based on algorithm analysis, a viral tweet has:

1. ✅ **Strong Hook** (first line) → Stops scroll, increases dwell time
2. ✅ **Visual Element** (image/video) → Boosts engagement
3. ✅ **Shareability** (quotable, useful, funny) → Gets retweets
4. ✅ **Discussion Prompt** (question, hot take) → Generates replies
5. ✅ **Posted by High-TweepCred Account** → Better initial distribution
6. ✅ **Early Engagement Velocity** → Signals quality to algorithm
7. ✅ **Topic Relevance** → Matches user interests (SimClusters)
8. ✅ **Proper Timing** → When audience is active
9. ✅ **Clean Content** → No negative signals
10. ✅ **Network Effect** → Engagement from quality accounts

### The Account Reputation Formula

To maximize your TweepCred score:

1. ✅ Age account properly (30+ days for full weight)
2. ✅ Maintain following/followers ratio <0.6 (if >500 following)
3. ✅ Get verified if possible (instant max score = 100)
4. ✅ Build network with high-quality accounts
5. ✅ Avoid all negative signals (reports, blocks, mutes)
6. ✅ Create consistent, engaging content
7. ✅ Engage authentically with community

---

## Advanced Techniques

### 1. PageRank Optimization

Your TweepCred is literally your PageRank score in Twitter's social graph.

**Optimization strategies**:
- Get followed by verified/influential accounts (PageRank flows to you)
- Interact with high-PageRank accounts (creates weighted edges)
- Build reciprocal follows with quality accounts
- Create content that quality accounts want to share

### 2. Multi-Model Optimization

Your content is scored by multiple models:
- **Light Ranker** (Earlybird) - Initial candidate filtering
- **Heavy Ranker** (Neural network) - Deep ranking with 6000+ features
- **SimClusters** - Community/topic matching
- **TwHIN** - Knowledge graph embeddings

**Strategy**: Optimize for each stage
- Light Ranker: Keywords, recency, engagement velocity
- Heavy Ranker: All engagement signals, quality signals
- SimClusters: Topic consistency, community alignment
- TwHIN: Network structure, entity relationships

### 3. A/B Test Awareness

Twitter constantly runs A/B tests on weight configurations.

**Implications**:
- Algorithm behavior varies by user
- What works for one audience may differ for another
- Test your own content variations
- Adapt to what works for YOUR account

### 4. Real-Time Signals

The algorithm uses real-time streams:
- Unified User Actions (UUA) - Real-time engagement events
- GraphJet - In-memory graph for recent interactions

**Optimization**:
- Early engagement is amplified
- Recent interactions have more weight
- Velocity matters (engagement rate over time)

---

## Conclusion

Twitter's algorithm is complex but understandable. The key factors are:

1. **Engagement metrics** (likes, retweets, replies, dwell time)
2. **Account reputation** (TweepCred via PageRank)
3. **Content quality** (negative signals hurt massively)
4. **Network effects** (who engages with you matters)
5. **Timing & consistency** (when and how often you post)

By optimizing across all these dimensions, you can significantly increase your visibility and chances of going viral.

**Remember**: The algorithm rewards **genuine, engaging content** from **reputable accounts**. Focus on creating value, building community, and maintaining a healthy account profile.

---

## Technical References

This guide is based on analysis of Twitter's open-source algorithm repository:
- `home-mixer/` - Timeline construction and ranking
- `src/scala/com/twitter/graph/batch/job/tweepcred/` - TweepCred calculation
- `RETREIVAL_SIGNALS.md` - User signal tracking
- `src/python/twitter/deepbird/projects/timelines/` - ML ranking models
- `representation-scorer/` - Embedding and similarity scoring

For more details, explore the source code at: https://github.com/twitter/the-algorithm

---

*Last Updated: Based on source code as of 2023 release*
*Note: Twitter/X may update algorithms over time. These insights are based on the open-source release.*
