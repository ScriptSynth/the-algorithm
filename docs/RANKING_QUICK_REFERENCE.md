# Tweet Ranking Quick Reference

A concise guide to understanding and optimizing for X's (Twitter's) recommendation algorithm.

## 🎯 How Tweets Are Ranked

```
Candidate Generation → Feature Hydration → ML Scoring → Filtering → Your Timeline
(~1 billion tweets)   (~6,000 features)  (probability)  (diversity)   (~100 tweets)
```

### The 6-Stage Pipeline

1. **Candidate Generation**: Fetch ~thousands of tweets from multiple sources
2. **Feature Hydration**: Compute ~6,000 features per tweet
3. **ML Scoring**: Neural networks predict engagement probability
4. **Filtering**: Apply diversity, quality, and safety filters
5. **Re-ranking**: Optimize for diversity and freshness
6. **Mixing**: Combine with ads and recommendations

---

## 📊 Ranking Signals (What Matters Most)

### Engagement Signals (Primary)
| Action | Weight | Impact |
|--------|--------|--------|
| 👍 Likes | ⭐⭐⭐⭐⭐ | Very High |
| 🔄 Retweets | ⭐⭐⭐⭐⭐ | Very High |
| 💬 Quote Tweets | ⭐⭐⭐⭐⭐ | Very High |
| ↩️ Replies | ⭐⭐⭐⭐ | High |
| ▶️ Video Watch Time | ⭐⭐⭐⭐ | High |
| 🔍 Clicks | ⭐⭐⭐ | Medium |
| 🔖 Bookmarks | ⭐⭐⭐ | Medium |
| 👤 Profile Visits | ⭐⭐ | Low |
| ❌ "Not Interested" | ⭐⭐⭐⭐⭐ | Very Negative |
| 🚫 Reports | ⭐⭐⭐⭐⭐ | Very Negative |

### Content Features
- ✅ **High-quality media** (photos, videos)
- ✅ **Topic relevance** to user interests
- ✅ **Clear, compelling text**
- ✅ **Authentic, original content**
- ✅ **Recency** (newer = better)

### Author Features
- ✅ **Follower count** (medium impact)
- ✅ **Engagement rate** (high impact)
- ✅ **Account reputation** (TweepCred score)
- ✅ **Verification** (small boost)
- ✅ **Posting consistency**

### Graph Features
- ✅ **Direct follows** (very high impact)
- ✅ **Real Graph score** (interaction likelihood)
- ✅ **Two-hop connections** (friends of friends)
- ✅ **Mutual relationships**

---

## ✅ Best Practices: How to Rank Your Tweets

### Content Optimization

**📝 Tweet Structure:**
```
[Hook - Attention-grabbing first line]
↓
[Context - Brief setup]
↓
[Value - Main insight]
↓
[CTA - Call to action]
↓
[Media - Photo/video if relevant]
```

**✅ Do This:**
- Post 1-3 times per day
- Use 1-2 relevant hashtags (not 10)
- Include high-quality images or videos
- Write concise, valuable content
- Respond to replies quickly
- Post when your audience is active
- Create shareable insights
- Be authentic and consistent

**❌ Don't Do This:**
- Spam (>10 tweets/hour)
- Engagement bait ("RT if you agree!")
- Misleading clickbait
- Excessive hashtags
- All caps or excessive punctuation
- Copy content without credit
- Post low-quality content
- Ignore your community

### Media Guidelines

**📷 Images:**
- High resolution (1200x675px optimal)
- Clear, relevant visuals
- Include alt-text
- Avoid text-heavy images

**🎥 Videos:**
- Hook viewers in first 3 seconds
- Keep 30-60 seconds (sweet spot)
- Add captions (most watch muted)
- High completion rate boosts ranking

### Timing Strategy

**⏰ Optimal Timing:**
- First 30 minutes are CRITICAL
- Weekday mornings (9-11am)
- Weekday afternoons (1-3pm)
- Test your specific audience times

**📈 Velocity Matters:**
- Fast early engagement = big boost
- Share in relevant communities
- Engage with early responders
- Don't delete and repost

---

## 🔍 Understanding In-Network vs Out-of-Network

### In-Network (Following)
- ~50% of For You timeline
- Tweets from accounts you follow
- **No scoring penalty**
- Higher baseline ranking

### Out-of-Network (Recommendations)
- ~50% of For You timeline
- From accounts you don't follow
- **0.75x score multiplier** (25% penalty)
- Must be 33% better to compete
- Requires exceptional quality

**Key Insight:** To reach beyond your followers, your content must be significantly better than average.

---

## 🏗️ Building Your Own Ranking System

### Minimal Viable Ranker

```python
def rank_content(user_id, candidates):
    scored = []
    for item in candidates:
        # Simple scoring
        recency = 1.0 / (1 + hours_since_post(item))
        engagement = (item.likes + 2*item.retweets + 3*item.replies) / max(item.impressions, 1)
        relevance = compute_relevance(user_id, item)
        
        score = 0.3*recency + 0.4*engagement + 0.3*relevance
        scored.append((item, score))
    
    scored.sort(key=lambda x: x[1], reverse=True)
    return [item for item, score in scored[:100]]
```

### Essential Features to Track

**Author:**
- Follower count
- Account age
- Engagement rate
- Reputation score

**Content:**
- Has media (photo/video)
- Text length
- Topic/category
- Language

**Engagement:**
- Like count
- Retweet count
- Reply count
- Engagement velocity

**User-Content Affinity:**
- User follows author
- User topic interest
- Historical engagement

**Temporal:**
- Hours since post
- Engagement velocity
- Is trending

### Key Architecture Components

1. **Candidate Sources**: Multiple retrieval methods (search, graph, ML)
2. **Feature Store**: Fast access to user/content features
3. **ML Model**: Predict engagement probability
4. **Diversity Filters**: Avoid showing too much from one source
5. **A/B Testing**: Experiment and measure

---

## 📈 Key Metrics to Track

### For Creators:
- **Engagement Rate** = Total Engagements / Impressions
- **Reply Rate** = Replies / Impressions
- **Retweet Rate** = Retweets / Impressions
- **Video Completion Rate** = Watches to End / Total Watches
- **Profile Visit Rate** = Profile Visits / Impressions

### For System Builders:
- **Precision@K**: Relevant items in top K results
- **Engagement Rate**: User interactions / Impressions
- **Diversity**: Unique authors/topics in results
- **Latency**: Time to generate rankings
- **User Satisfaction**: Retention, time spent

---

## 🎓 Algorithm Components Reference

### Main Services
- **Home Mixer**: Main ranking orchestration
- **Earlybird**: In-network search index
- **UTEG**: User-tweet graph recommendations
- **TweetMixer**: Out-of-network coordination
- **FRS**: Follow recommendations

### ML Models
- **Heavy Ranker**: Multi-task neural network (main scorer)
- **Light Ranker**: Fast pre-ranking model
- **Navi**: High-performance model serving

### Feature Systems
- **SimClusters**: Community detection & embeddings
- **TwHIN**: Dense knowledge graph embeddings
- **Real Graph**: User interaction prediction
- **TweepCred**: PageRank reputation

---

## 💡 Quick Tips

### For Maximum Reach:
1. Create shareable, valuable content
2. Post consistently (same time, same quality)
3. Engage authentically with your community
4. Use media (especially video)
5. Optimize for likes and retweets
6. Monitor what works and iterate

### Common Mistakes to Avoid:
1. Too much self-promotion
2. Posting at random times
3. Ignoring replies and mentions
4. Using engagement bait tactics
5. Inconsistent posting schedule
6. Low-quality or irrelevant content
7. Copying without attribution
8. Overusing hashtags

---

## 📚 Learn More

- **Full Guide**: [TWEET_RANKING_GUIDE.md](./TWEET_RANKING_GUIDE.md)
- **Main README**: [../README.md](../README.md)
- **Retrieval Signals**: [../RETREIVAL_SIGNALS.md](../RETREIVAL_SIGNALS.md)
- **Home Mixer**: [../home-mixer/README.md](../home-mixer/README.md)

---

## 🎯 TL;DR

**The algorithm rewards:**
- 👍 Authentic engagement (likes, retweets, replies)
- 🎨 High-quality media
- 💎 Valuable, original content
- 🤝 Meaningful connections
- ⚡ Fast initial engagement
- 📊 Consistent quality

**The algorithm penalizes:**
- 🚫 Spam and engagement bait
- 👎 Low-quality content
- 😴 Negative feedback signals
- 📉 Inconsistent posting
- 🔇 Ignored community

**Bottom line:** Create authentic, valuable content that your audience wants to engage with. The algorithm will reward genuine quality.

---

*Last Updated: 2026*
