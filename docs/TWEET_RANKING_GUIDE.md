# Tweet Ranking Algorithm: Complete Guide

## Overview

This guide explains how X's (formerly Twitter) recommendation algorithm ranks tweets in the "For You" timeline and provides practical guidance on how similar ranking principles can be applied to your own tweets and content recommendation systems.

## Table of Contents

1. [How the Algorithm Works](#how-the-algorithm-works)
2. [Key Components](#key-components)
3. [Ranking Signals and Features](#ranking-signals-and-features)
4. [Applying These Principles to Your Tweets](#applying-these-principles-to-your-tweets)
5. [Building Your Own Ranking System](#building-your-own-ranking-system)

---

## How the Algorithm Works

X's tweet ranking algorithm is a sophisticated multi-stage pipeline that processes approximately 1 billion potential tweets down to the few thousand that appear in your "For You" timeline.

### High-Level Process Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. CANDIDATE GENERATION                                         │
│    Fetch ~few thousand tweets from multiple sources             │
│    (~1 billion → ~thousands)                                    │
└──────────────────┬──────────────────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. FEATURE HYDRATION                                            │
│    Fetch ~6,000 features for each candidate tweet               │
│    (author, content, engagement, graph features)                │
└──────────────────┬──────────────────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. ML SCORING                                                   │
│    Neural network models predict engagement probability         │
│    (likes, retweets, clicks, watch time)                        │
└──────────────────┬──────────────────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. FILTERING & HEURISTICS                                       │
│    Apply diversity, quality, safety filters                     │
│    (deduplication, author diversity, content balance)           │
└──────────────────┬──────────────────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. MIXING & PRESENTATION                                        │
│    Combine with ads, who-to-follow, apply final ranking         │
│    (~thousands → ~hundreds shown)                               │
└─────────────────────────────────────────────────────────────────┘
```

### The Six Stages in Detail

#### Stage 1: Candidate Generation
Multiple specialized systems fetch tweet candidates:

- **In-Network Source** (~50% of tweets): Tweets from accounts you follow, sourced from the Earlybird search index
- **Out-of-Network Sources** (~50% of tweets):
  - **UTEG (User Tweet Entity Graph)**: Finds tweets based on in-memory graph traversals of user-tweet interactions
  - **TweetMixer**: Coordinates fetching from multiple candidate services
  - **FRS (Follow Recommendation Service)**: Suggests tweets from accounts you might want to follow
  - **SimClusters**: Community-based recommendations using sparse embeddings
  - **TwHIN**: Dense knowledge graph embeddings for users and tweets

#### Stage 2: Feature Hydration
For each candidate tweet, approximately **6,000 features** are fetched and computed:

- Author features (reputation, follower count, verification)
- Tweet content features (text embeddings, media, topics)
- Engagement features (likes, retweets, replies - both real-time and historical)
- Graph features (social connections, interaction likelihood)
- User-specific features (interests, language, past behavior)

#### Stage 3: ML Model Scoring
Neural network models (primarily **Navi** and **Phoenix**) predict:
- Probability of like
- Probability of retweet
- Probability of reply
- Probability of engagement (click, video watch)
- Probability of negative feedback (report, "not interested")

These probabilities are combined into a single relevance score.

#### Stage 4: Filtering and Heuristics
Multiple filters ensure quality and diversity:
- **Author Diversity**: Avoid showing too many tweets from the same author
- **Content Balance**: Mix in-network and out-of-network content (typically 50/50)
- **Feedback Fatigue**: Reduce tweets similar to ones you've indicated disinterest in
- **Deduplication**: Remove duplicates and tweets you've already seen
- **Visibility Filtering**: Block/mute enforcement, NSFW filtering, safety policies
- **OON Scaling**: Out-of-network tweets get a 0.75x score multiplier

#### Stage 5: Re-ranking with Listwise Diversity
Additional diversity and quality adjustments:
- Diversity discount for similar content
- Author-based listwise reranking
- Candidate source diversity
- Impression fatigue decay

#### Stage 6: Final Mixing
The final timeline is assembled:
- Tweets are mixed with ads
- Who-to-follow recommendations inserted
- Social context added (e.g., "liked by people you follow")
- Conversation modules for replies

---

## Key Components

### 1. Home Mixer Service
**Location**: `home-mixer/`

Main orchestration service that:
- Coordinates all ranking pipelines
- Manages feature hydration
- Applies final filtering and mixing
- Built on Product Mixer framework

### 2. Candidate Sources

#### In-Network: Earlybird Search Index
**Location**: `src/java/com/twitter/search/`
- Real-time search index of recent tweets
- Powers ~50% of For You timeline
- Efficient retrieval of tweets from followed accounts

#### Out-of-Network: UTEG
**Location**: `src/scala/com/twitter/recos/user_tweet_entity_graph/`
- GraphJet-based in-memory graph
- Traverses user-tweet interactions
- Finds similar tweets based on engagement patterns

#### TweetMixer
**Location**: `tweet-mixer/`
- Coordinates multiple OON sources
- Blends different recommendation types

#### Follow Recommendation Service (FRS)
**Location**: `follow-recommendations-service/`
- Recommends accounts to follow
- Surfaces tweets from those accounts

### 3. Machine Learning Models

#### Heavy Ranker
**External**: [See GitHub ML repo](https://github.com/twitter/the-algorithm-ml/blob/main/projects/home/recap/README.md)
- Multi-task neural network
- Predicts multiple engagement types
- Primary signal for tweet selection
- Uses ~6,000 input features

#### Light Ranker
**Location**: `src/python/twitter/deepbird/projects/timelines/scripts/models/earlybird/`
- Fast, lightweight model
- Used in Earlybird for pre-ranking
- Reduces candidate set before heavy ranking

### 4. Feature Systems

#### SimClusters
**Location**: `src/scala/com/twitter/simclusters_v2/`
- Community detection algorithm
- Creates sparse embeddings for users and tweets
- Finds similar users and content

#### Real Graph
**Location**: `src/scala/com/twitter/interaction_graph/`
- Predicts likelihood of user interaction
- Based on historical engagement patterns

#### TweepCred
**Location**: `src/scala/com/twitter/graph/batch/job/tweepcred/`
- PageRank-based user reputation score
- Identifies authoritative accounts

### 5. Safety and Quality

#### Visibility Filters
**Location**: `visibilitylib/`
- Enforces block/mute lists
- NSFW content filtering
- Compliance and safety rules
- Downranking of low-quality content

#### Trust and Safety Models
**Location**: `trust_and_safety_models/`
- Detects NSFW content
- Identifies abusive content
- Protects user experience

---

## Ranking Signals and Features

Understanding the signals used for ranking is crucial for optimizing your tweets' performance.

### User Engagement Signals (Primary Training Labels)

These are the strongest signals and directly train the ML models:

| Signal | Strength | Description |
|--------|----------|-------------|
| **Likes/Favorites** | Very High | Explicit positive signal; widely used across all models |
| **Retweets** | Very High | Strong sharing signal; indicates high-quality content |
| **Quote Tweets** | Very High | Engagement with commentary; shows thought-provoking content |
| **Replies** | High | Conversation starter; indicates engaging content |
| **Video Watch Time** | High | Completion rate matters; longer watch = better signal |
| **Click-through** | Medium | User viewed tweet details; interest indicator |
| **Bookmarks** | Medium | Save for later; indicates valuable content |
| **Shares** | Medium | External sharing signal |
| **Profile Visits** | Low | Indirect engagement; interest in author |
| **"Not Interested"** | Very Negative | Strong negative signal; reduces similar content |
| **Report** | Very Negative | Strongest negative signal; indicates problematic content |

### Author Features

| Feature | Impact | Description |
|---------|--------|-------------|
| **Follower Count** | Medium-High | More followers = potentially wider reach |
| **Verification Status** | Medium | Verified accounts may get slight boost |
| **Account Age** | Low-Medium | Older accounts may have trust advantage |
| **TweepCred Score** | Medium | PageRank-based reputation; identifies authoritative users |
| **Posting Frequency** | Variable | Consistency matters; too frequent can hurt |
| **Engagement Rate** | High | Historical engagement patterns on author's content |

### Content Features

| Feature | Impact | Description |
|---------|--------|-------------|
| **Text Quality** | High | Well-written, informative content ranks better |
| **Media Presence** | High | Photos/videos generally perform better |
| **Video Completion Rate** | Very High | Historical completion rate for videos |
| **Topic Relevance** | High | Alignment with user interests |
| **Language** | High | Matches user's preferred language |
| **Link Quality** | Variable | High-quality links boost; low-quality links hurt |
| **Hashtag Usage** | Low-Medium | Moderate use okay; overuse may hurt |
| **Text Length** | Variable | Medium-length often optimal (not too short, not too long) |

### Timing Features

| Feature | Impact | Description |
|---------|--------|-------------|
| **Recency** | High | Recent tweets preferred, especially for in-network |
| **Velocity** | Very High | Fast initial engagement = strong boost |
| **Half-life** | Medium | Rate of engagement decay over time |

### Graph Features

| Feature | Impact | Description |
|---------|--------|-------------|
| **Real Graph Score** | High | Likelihood of interaction between users |
| **Two-hop Connections** | Medium | Friends of friends engagement |
| **Author-User Relationship** | Very High | Direct follow relationship strongly matters |
| **Mutual Follows** | Medium | Bidirectional relationships |

### Aggregate Features

| Feature | Impact | Description |
|---------|--------|-------------|
| **Topic Engagement** | High | User's historical engagement with topic |
| **Author Engagement** | High | User's past engagement with this author |
| **Similar Content Engagement** | Medium | Performance of similar tweets |
| **Country/Language Aggregates** | Medium | Regional performance signals |

---

## Applying These Principles to Your Tweets

Now that you understand how the algorithm works, here's how to optimize your tweets for better ranking:

### 1. Optimize for Primary Engagement Signals

**Focus on Likes and Retweets:**
- Create content that people want to share
- Ask questions that prompt responses
- Share valuable, actionable insights
- Use emotional hooks (inspiration, humor, surprise)

**Encourage Replies:**
- End with questions
- Take controversial (but thoughtful) positions
- Create discussion-worthy content
- Respond to replies to keep conversations going

**Video Best Practices:**
- Hook viewers in first 3 seconds
- Keep videos concise (30-60 seconds often optimal)
- Add captions (most watch without sound)
- Create content worth watching to completion

### 2. Build Your Author Reputation

**Consistency is Key:**
- Post regularly (1-3 times per day optimal for most)
- Maintain a consistent voice and topic focus
- Build expertise in specific areas

**Grow Thoughtfully:**
- Focus on quality followers, not just quantity
- Engage meaningfully with your community
- Collaborate with others in your niche

**Establish Authority:**
- Share original insights and research
- Cite sources and be factually accurate
- Demonstrate expertise through consistent quality

### 3. Content Optimization

**Use Media Effectively:**
- Include images or videos when relevant
- Ensure high-quality visuals
- Use alt-text for accessibility

**Write Compelling Text:**
- Start with a strong hook
- Use clear, concise language
- Break up text with line breaks for readability
- Use bold claims that are backed by evidence

**Topic Alignment:**
- Stay focused on topics your audience cares about
- Use relevant hashtags (1-2, not 10)
- Engage with trending topics when appropriate

**Optimal Tweet Structure:**
```
[HOOK - First line grabs attention]

[CONTEXT - Brief setup or background]

[VALUE - Main insight or information]

[CALL TO ACTION - Encourage engagement]

[OPTIONAL: Media/Link]
```

### 4. Timing and Velocity

**Post at Optimal Times:**
- Test different times to find when your audience is active
- Generally: weekday mornings and early afternoons perform well
- Consider your audience's time zones

**Maximize Early Engagement:**
- The first 30 minutes are crucial
- Share in relevant communities
- Engage with early responders
- Don't delete and repost (resets engagement signals)

**Build Momentum:**
- Follow up on successful tweets
- Create threads to maintain attention
- Cross-promote your best content

### 5. Avoid Negative Signals

**Don't Do These:**
- ❌ Spam or excessive posting (>10 tweets/hour)
- ❌ Engagement bait ("RT if you agree!")
- ❌ Misleading clickbait
- ❌ Low-quality or broken links
- ❌ Excessive hashtags (#like #this #with #ten #hashtags)
- ❌ All caps or excessive punctuation
- ❌ Controversial content just for engagement
- ❌ Copying others' content without credit

**Safety and Quality:**
- Follow community guidelines
- Be respectful even when disagreeing
- Fact-check before sharing
- Give credit to original sources

### 6. Leverage Graph Effects

**Build Meaningful Connections:**
- Follow and engage with users in your niche
- Reply thoughtfully to others' tweets
- Quote tweet with added value
- Collaborate on content

**Tap Into Networks:**
- Engage with users who have engaged audiences
- Get mentioned or retweeted by larger accounts
- Participate in relevant communities

### 7. Understand Content Balance

**In-Network vs. Out-of-Network:**
- In-network (your followers) get preference
- Out-of-network reach requires exceptional quality
- To reach beyond your followers: create shareable, valuable content
- Remember: OON tweets get 0.75x scoring penalty, so must be 33% better to compete

### 8. Analyze and Iterate

**Track Your Performance:**
- Monitor which tweets perform well
- Identify patterns in your best content
- Learn from both successes and failures

**Key Metrics to Watch:**
- Engagement rate (engagements / impressions)
- Reply rate
- Retweet rate
- Video completion rate
- Profile visits from tweets

**A/B Test:**
- Try different formats
- Test various topics
- Experiment with posting times
- Compare media vs. text-only

---

## Building Your Own Ranking System

If you're building a content recommendation system, here's how to apply X's architecture:

### 1. Multi-Stage Pipeline Architecture

```python
# Pseudo-code for a basic ranking pipeline

def rank_content(user_id, timestamp):
    # Stage 1: Candidate Generation
    candidates = []
    candidates += fetch_from_follows(user_id, limit=1000)
    candidates += fetch_similar_content(user_id, limit=1000)
    candidates += fetch_trending(limit=500)
    
    # Stage 2: Feature Hydration
    features = hydrate_features(candidates, user_id)
    
    # Stage 3: ML Scoring
    scores = ml_model.predict(features)
    
    # Stage 4: Filtering
    filtered = apply_filters(candidates, scores, user_id)
    
    # Stage 5: Re-ranking for Diversity
    reranked = apply_diversity_rules(filtered)
    
    # Stage 6: Final Selection
    return reranked[:100]
```

### 2. Essential Features to Collect

**Start with these core features:**

```python
# Author Features
author_features = {
    'follower_count': int,
    'account_age_days': int,
    'avg_engagement_rate': float,
    'posting_frequency': float,
    'reputation_score': float
}

# Content Features
content_features = {
    'has_media': bool,
    'has_video': bool,
    'text_length': int,
    'sentiment_score': float,
    'topic_categories': list,
    'language': str,
    'readability_score': float
}

# Engagement Features (historical)
engagement_features = {
    'total_likes': int,
    'total_retweets': int,
    'total_replies': int,
    'engagement_velocity': float,  # engagements per hour
    'similar_content_performance': float
}

# User-Content Affinity
affinity_features = {
    'user_follows_author': bool,
    'user_topic_interest': float,
    'user_language_match': bool,
    'historical_engagement_with_author': float,
    'social_graph_distance': int
}

# Temporal Features
temporal_features = {
    'hours_since_post': float,
    'engagement_velocity_last_hour': float,
    'is_trending': bool
}
```

### 3. Building a Simple ML Model

**Start with a gradient boosting model (LightGBM/XGBoost):**

```python
import lightgbm as lgb
from sklearn.model_selection import train_test_split

# Prepare training data
# Target: did user engage with content? (1=yes, 0=no)
X_train, X_test, y_train, y_test = prepare_training_data()

# Train model
model = lgb.LGBMClassifier(
    objective='binary',
    n_estimators=100,
    learning_rate=0.05,
    max_depth=8
)

model.fit(
    X_train, y_train,
    eval_set=[(X_test, y_test)],
    early_stopping_rounds=10
)

# Predict engagement probability
scores = model.predict_proba(features)[:, 1]
```

**For production systems, use multi-task learning:**

```python
# Predict multiple engagement types simultaneously
def multi_task_model(features):
    """
    Predict probabilities for:
    - Like
    - Share
    - Reply
    - Click
    - Negative feedback
    """
    base_network = create_shared_layers(features)
    
    like_pred = dense_layer(base_network, name='like')
    share_pred = dense_layer(base_network, name='share')
    reply_pred = dense_layer(base_network, name='reply')
    click_pred = dense_layer(base_network, name='click')
    negative_pred = dense_layer(base_network, name='negative')
    
    # Weighted combination
    final_score = (
        2.0 * like_pred +
        3.0 * share_pred +
        4.0 * reply_pred +
        1.0 * click_pred -
        10.0 * negative_pred
    )
    
    return final_score
```

### 4. Implementing Diversity and Quality Filters

```python
def apply_diversity_filters(ranked_content, user_id):
    """Apply post-scoring diversity and quality rules"""
    
    filtered = []
    author_counts = {}
    topic_counts = {}
    
    for item in ranked_content:
        # Author diversity: max 3 items per author
        if author_counts.get(item.author_id, 0) >= 3:
            continue
        
        # Topic diversity: max 5 items per topic
        if topic_counts.get(item.topic, 0) >= 5:
            continue
        
        # Quality threshold
        if item.score < 0.1:
            continue
        
        # Remove seen content
        if is_already_seen(user_id, item.id):
            continue
        
        filtered.append(item)
        author_counts[item.author_id] = author_counts.get(item.author_id, 0) + 1
        topic_counts[item.topic] = topic_counts.get(item.topic, 0) + 1
    
    return filtered

def apply_content_balance(in_network, out_network, target_ratio=0.5):
    """Balance in-network vs out-of-network content"""
    
    total_items = 100
    in_network_count = int(total_items * target_ratio)
    out_network_count = total_items - in_network_count
    
    # Out-of-network items need higher scores to compete
    # Apply scaling factor
    for item in out_network:
        item.score *= 0.75
    
    # Merge and re-sort
    combined = in_network[:in_network_count] + out_network[:out_network_count]
    combined.sort(key=lambda x: x.score, reverse=True)
    
    return combined[:total_items]
```

### 5. Real-Time Feature Computation

**Use stream processing for real-time features:**

```python
# Using Apache Kafka/Flink for real-time aggregates
def compute_realtime_features(tweet_id):
    """
    Compute real-time engagement features
    """
    # Get engagement events from last hour
    events = kafka_consumer.get_events(
        topic='tweet_engagements',
        key=tweet_id,
        time_window='1h'
    )
    
    features = {
        'likes_last_hour': count(events, type='like'),
        'retweets_last_hour': count(events, type='retweet'),
        'replies_last_hour': count(events, type='reply'),
        'engagement_velocity': len(events) / hours_since_post(tweet_id),
        'engagement_acceleration': compute_acceleration(events)
    }
    
    return features
```

### 6. A/B Testing and Experimentation

```python
def select_ranking_algorithm(user_id):
    """
    A/B test different ranking approaches
    """
    experiment_group = hash(user_id) % 100
    
    if experiment_group < 10:  # 10% in test group
        return rank_with_new_algorithm(user_id)
    else:  # 90% in control group
        return rank_with_current_algorithm(user_id)

def track_metrics(user_id, shown_content, experiment_group):
    """
    Track key metrics for each experiment group
    """
    metrics = {
        'engagement_rate': compute_engagement_rate(user_id, shown_content),
        'time_spent': compute_time_spent(user_id),
        'user_satisfaction': get_user_satisfaction_signals(user_id)
    }
    
    log_experiment_metrics(experiment_group, metrics)
```

### 7. Infrastructure Considerations

**For a production ranking system:**

1. **Candidate Generation**:
   - Use Elasticsearch/Solr for search-based retrieval
   - Redis for caching recent content
   - Graph databases (Neo4j) for graph-based recommendations

2. **Feature Storage**:
   - Real-time features: Redis/Memcached
   - Batch features: Cassandra/HBase
   - Embeddings: Vector databases (Pinecone, Milvus)

3. **Model Serving**:
   - TensorFlow Serving, TorchServe, or custom serving layer
   - Model versioning and A/B testing
   - Feature stores (Feast, Tecton)

4. **Monitoring**:
   - Track model performance metrics
   - Monitor latency at each stage
   - Alert on score distribution shifts

### 8. Simplified Starting Point

If you're just getting started, here's a minimal viable ranking system:

```python
class SimpleRankingSystem:
    def __init__(self):
        self.weights = {
            'recency': 0.3,
            'engagement': 0.4,
            'relevance': 0.3
        }
    
    def rank(self, user_id, candidates):
        scored = []
        
        for item in candidates:
            # Simple scoring function
            recency_score = 1.0 / (1 + hours_since_post(item))
            engagement_score = (
                item.likes + 2*item.retweets + 3*item.replies
            ) / (1 + item.impressions)
            relevance_score = self.compute_relevance(user_id, item)
            
            final_score = (
                self.weights['recency'] * recency_score +
                self.weights['engagement'] * engagement_score +
                self.weights['relevance'] * relevance_score
            )
            
            scored.append((item, final_score))
        
        # Sort by score and return top N
        scored.sort(key=lambda x: x[1], reverse=True)
        return [item for item, score in scored[:100]]
    
    def compute_relevance(self, user_id, item):
        # Simple relevance based on user interests
        user_interests = get_user_interests(user_id)
        item_topics = get_item_topics(item)
        
        overlap = len(set(user_interests) & set(item_topics))
        return overlap / max(len(user_interests), 1)
```

---

## Key Takeaways

### For Tweet Creators:
1. **Quality over quantity**: Focus on creating engaging, valuable content
2. **Optimize for early engagement**: First 30 minutes are critical
3. **Build your network**: Meaningful connections amplify reach
4. **Use media effectively**: Photos and videos boost engagement
5. **Understand the signals**: Likes, retweets, and replies are the strongest positive signals
6. **Avoid negative signals**: Don't spam, use clickbait, or post low-quality content
7. **Be consistent**: Regular posting builds audience and authority
8. **Analyze and iterate**: Learn from your data and improve

### For System Builders:
1. **Multi-stage pipeline**: Candidate generation → Feature hydration → ML scoring → Filtering
2. **Rich features**: Collect diverse signals (author, content, engagement, graph, temporal)
3. **ML models**: Start simple (gradient boosting), scale to neural networks
4. **Diversity matters**: Don't show all content from one source or author
5. **Balance exploration and exploitation**: Mix familiar and new content
6. **Real-time processing**: Engagement velocity is a powerful signal
7. **A/B test everything**: Continuously experiment and improve
8. **Monitor and iterate**: Track metrics and adapt to user behavior changes

---

## Additional Resources

### From This Repository:
- [System Architecture](../README.md)
- [Retrieval Signals](../RETREIVAL_SIGNALS.md)
- [Home Mixer](../home-mixer/README.md) - Main ranking service
- [Heavy Ranker ML Models](https://github.com/twitter/the-algorithm-ml) - Neural network models

### External Resources:
- [X Engineering Blog](https://blog.x.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm) - Algorithm overview
- [GraphJet Paper](https://github.com/twitter/GraphJet) - Real-time graph processing
- [Recommendation Systems Handbook](https://www.springer.com/gp/book/9780387858203) - Academic resource

### Academic Papers:
- "Deep Neural Networks for YouTube Recommendations" - Google (2016)
- "Recommending What Video to Watch Next: A Multitask Ranking System" - Google (2019)
- "TwHIN: Embedding the Twitter Heterogeneous Information Network" - Twitter (2022)

---

## Conclusion

X's tweet ranking algorithm is a sophisticated system that balances multiple objectives: relevance, diversity, quality, and safety. By understanding these principles and applying them thoughtfully, you can both optimize your tweets for better performance and build your own recommendation systems.

Remember: **The algorithm favors authentic, high-quality content that generates genuine engagement.** Focus on creating value for your audience, and the algorithmic signals will follow.

Good luck with your tweets and your ranking systems! 🚀
