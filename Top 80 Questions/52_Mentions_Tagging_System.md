# 52. Design a Mentions and Tagging System

---

## 1. Functional Requirements (FR)

- **Mention users**: Tag users in posts, comments, stories using @username syntax
- **Tag in media**: Tag users in photos/videos at specific positions (like Instagram photo tags)
- **Mention notifications**: Notify mentioned users in real-time
- **Mention feed**: "Posts you're mentioned in" aggregated view
- **Autocomplete**: As user types "@al", suggest matching usernames
- **Mention permissions**: Users can control who can mention them (everyone, followers, nobody)
- **Remove tags**: Users can remove themselves from mentions/tags
- **Mention linking**: Clickable @username links in rendered content

---

## 2. Non-Functional Requirements (NFRs)

- **Low Latency**: Autocomplete suggestions in < 100 ms
- **Notification Speed**: Mentioned users notified within 5 seconds
- **Scale**: Handle 100M+ mentions/day across posts, comments, stories
- **Accuracy**: Autocomplete must match exact usernames, handle edge cases (dots, underscores)
- **Availability**: 99.99%
- **Anti-Spam**: Prevent mass-mention spam (rate limiting)

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Posts with mentions / day | 200M |
| Avg mentions per post | 2 |
| Total mention events / day | 400M |
| Mention events / sec | ~4.6K |
| Autocomplete queries / sec | 50K |
| Username lookup latency | < 50 ms |
| Notification delivery target | < 5 seconds |

---

## 4. High-Level Design (HLD)

```
+--------------------------------------------------------------------+
|                        Content Creation Flow                        |
|                                                                      |
|  User types: "Great photo with @alice and @bob! #sunset"            |
|                                                                      |
|  Client-side:                                                        |
|  1. Detect "@" character -> trigger autocomplete                    |
|  2. Show dropdown: alice_smith, alice_wonder, alice_in_chains       |
|  3. User selects "alice_smith" -> store mention metadata            |
|  4. On post submit: send content + mention list to server           |
+----------------------------+---------------------------------------+
                             |
                      +------v------+
                      |   Post /    |
                      |   Comment   |
                      |   Service   |
                      +------+------+
                             |
         +-------------------+-------------------+
         |                   |                   |
  +------v------+    +-------v-------+   +-------v-------+
  |  Content    |    |  Mention      |   | Autocomplete  |
  |  Store      |    |  Service      |   | Service       |
  |             |    |               |   |               |
  |  MySQL:     |    |  1. Validate  |   | Trie / ES     |
  |  posts,     |    |     mentioned |   | index for     |
  |  comments   |    |     users     |   | username      |
  |  with       |    |     exist     |   | prefix search |
  |  rendered   |    |  2. Check     |   |               |
  |  mention    |    |     permissions|  |               |
  |  links      |    |  3. Store     |   |               |
  |             |    |     mention   |   |               |
  |             |    |     records   |   |               |
  |             |    |  4. Kafka     |   |               |
  |             |    |     event     |   |               |
  +-------------+    +-------+-------+   +---------------+
                             |
                      +------v------+
                      |    Kafka    |
                      | (mentions   |
                      |  topic)     |
                      +------+------+
                             |
              +--------------+--------------+
              |              |              |
       +------v------+ +----v------+ +-----v------+
       | Notification| | Mention   | | Analytics  |
       | Service     | | Feed      | | (mention   |
       |             | | Builder   | |  tracking) |
       | Push/email  | |           | |            |
       | to mentioned| | Build     | |            |
       | users       | | "tagged   | |            |
       |             | | in" feed  | |            |
       +-------------+ +-----------+ +------------+
```

### Component Deep Dive

#### Mention Extraction and Validation

```
Input text: "Amazing trip with @alice.smith and @bob_jones! cc @charlie"

Step 1: Extract mentions (client-side + server-side validation)
  Client-side: regex /@[a-zA-Z0-9._]+/ -> ["@alice.smith", "@bob_jones", "@charlie"]
  
  Client sends structured data:
  {
    "text": "Amazing trip with @alice.smith and @bob_jones! cc @charlie",
    "mentions": [
      {"username": "alice.smith", "offset": 18, "length": 12},
      {"username": "bob_jones", "offset": 35, "length": 10},
      {"username": "charlie", "offset": 50, "length": 8}
    ]
  }

Step 2: Server validates each mention:
  a. Does username exist? -> MySQL: SELECT user_id FROM users WHERE username = ?
     Batch lookup for efficiency: SELECT user_id, username FROM users WHERE username IN (...)
  b. Permission check: Can the poster mention this user?
     Redis: HGET mention_settings:{mentioned_user_id} allow_from
     Values: "everyone" | "followers" | "nobody"
     If "followers" -> check: is poster in mentioned user's followers?
  c. Is the poster blocked by the mentioned user?
     Redis: SISMEMBER blocked_by:{mentioned_user_id} {poster_id}
  
Step 3: Store valid mentions, skip invalid ones silently
  Invalid mention: render as plain text (not clickable)
  Valid mention: render as clickable link to profile

Step 4: Rate limit: max 20 mentions per post, max 50 mentions per hour
  Prevents mass-mention spam attacks
```

#### Autocomplete — "@al" -> Suggestions

```
User types "@al" in the compose box:

Priority-ranked results:
  1. Friends/following who match "al*" (highest priority — most likely intent)
  2. All users matching "al*" (lower priority)

Implementation:

  Tier 1: Client-side cache (instant, < 10ms)
    On app start: load user's following list (500 users) into local trie
    "@al" -> search local trie -> [alice_smith, alex_johnson] -> show immediately
    
  Tier 2: Server-side search (for non-following users)
    GET /api/v1/users/autocomplete?prefix=al&limit=5
    
    Backend options:
    
    a. Redis sorted set with lexicographic range:
       ZRANGEBYLEX usernames "[al" "[al\xff" LIMIT 0 5
       Pre-populate sorted set with all usernames
       Memory: 1B users * 20 bytes avg = 20 GB -> feasible in Redis cluster
    
    b. Elasticsearch prefix query:
       { "prefix": { "username": "al" } }
       Faster for fuzzy matching but higher latency (~20ms vs ~2ms Redis)
    
    c. Trie in-memory service:
       Custom service holding username trie in memory
       Fastest for exact prefix matching (< 1ms)
       Memory: ~5 GB for 1B usernames (compressed trie)
    
    Recommended: Redis sorted set for simplicity + Elasticsearch for fuzzy matching

  Ranking within results:
    Sort by: relevance score = 
      w1 * is_following(searcher, result) +
      w2 * mutual_friends_count +
      w3 * popularity(result) +
      w4 * recency_of_interaction
```

#### Photo/Video Tagging (Position-Based)

```
Instagram-style: Tap on a photo to tag a user at a specific (x, y) position

Data model for photo tags:
  {
    "media_id": "photo-123",
    "tags": [
      {"user_id": "u-alice", "x": 0.35, "y": 0.62},  // normalized coordinates
      {"user_id": "u-bob", "x": 0.71, "y": 0.45}
    ]
  }

  Coordinates normalized to [0, 1] range:
    (0, 0) = top-left, (1, 1) = bottom-right
    This works regardless of screen resolution or image crop
  
  Display: Overlay username labels at (x, y) positions
  Interaction: Tap label -> go to user profile

  Storage: JSONB column in posts table or separate media_tags table
  Indexing: "show me all photos I'm tagged in" -> index on tagged user_id
```

---

## 5. APIs

### Post with Mentions
```http
POST /api/v1/posts
{
  "text": "Great photo with @alice and @bob!",
  "mentions": [
    {"username": "alice", "offset": 17, "length": 6},
    {"username": "bob", "offset": 28, "length": 4}
  ],
  "media_tags": [
    {"user_id": "u-alice", "x": 0.35, "y": 0.62}
  ]
}
-> 201 Created { "post_id": "p-uuid" }
```

### Autocomplete
```http
GET /api/v1/users/autocomplete?prefix=al&limit=5
-> 200 OK
{
  "suggestions": [
    {"user_id": "u1", "username": "alice_smith", "name": "Alice Smith", "avatar": "..."},
    {"user_id": "u2", "username": "alex_jones", "name": "Alex Jones", "avatar": "..."}
  ]
}
```

### Get Mentions Feed ("Tagged In")
```http
GET /api/v1/mentions?cursor={last}&limit=20
-> 200 OK
{
  "mentions": [
    {"post_id": "p-uuid", "mentioned_by": {"id": "u3", "name": "Carol"},
     "type": "text_mention", "created_at": "..."},
    ...
  ]
}
```

### Remove Tag
```http
DELETE /api/v1/mentions/{mention_id}
-> 200 OK
```

### Update Mention Permissions
```http
PUT /api/v1/settings/mentions
{ "allow_from": "followers" }  // "everyone" | "followers" | "nobody"
-> 200 OK
```

---

## 6. Data Model

### MySQL — Mention Records
```sql
CREATE TABLE mentions (
    mention_id      BIGINT PRIMARY KEY AUTO_INCREMENT,
    content_type    ENUM('post', 'comment', 'story') NOT NULL,
    content_id      BIGINT NOT NULL,
    mentioned_user  BIGINT NOT NULL,          -- who is mentioned
    mentioned_by    BIGINT NOT NULL,          -- who mentioned them
    mention_type    ENUM('text', 'photo_tag') DEFAULT 'text',
    text_offset     INT,                      -- position in text
    text_length     INT,
    tag_x           FLOAT,                    -- photo tag position
    tag_y           FLOAT,
    status          ENUM('active', 'removed') DEFAULT 'active',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_mentioned_user (mentioned_user, created_at DESC),  -- "mentions of me" feed
    INDEX idx_content (content_type, content_id),                 -- "mentions in this post"
    INDEX idx_mentioned_by (mentioned_by, created_at DESC)        -- "posts I mentioned others in"
);
```

### Redis — Autocomplete + Permissions
```
# Username autocomplete (lexicographic sorted set)
usernames              -> Sorted Set (score=0, member=username)
  ZRANGEBYLEX usernames "[al" "[al\xff" LIMIT 0 5

# Mention permissions per user
mention_settings:{uid} -> Hash { allow_from: "followers" }

# Blocked users
blocked_by:{uid}       -> SET of user_ids who blocked this user

# Recent mentions feed (cached)
mentions_feed:{uid}    -> List of mention records (TTL: 5 min)
```

### Kafka — Mention Events
```
Topic: mention-events
Partition by: mentioned_user_id (all mentions of one user on same partition)
Message: { content_type, content_id, mentioned_user, mentioned_by, timestamp }
```

---

## 7. Fault Tolerance & Race Conditions

| Concern | Solution |
|---|---|
| **Mention spam** | Rate limit: max 20 mentions/post, 50/hour, 200/day per user |
| **Deleted post** | On post delete -> cascade delete mentions (or mark inactive) |
| **User renames** | Mentions store user_id not username; display renders current username |
| **Autocomplete staleness** | New user registration -> add to Redis sorted set immediately |
| **Permission change** | HSET in Redis (instant); existing mentions not retroactively removed |
| **Notification dedup** | If user mentioned 3x in same post -> ONE notification, not three |

### Race Condition: Post Published While Mentioned User Blocks Author

```
T=0:    Carol starts writing a post mentioning @alice
T=1s:   Alice blocks Carol
T=5s:   Carol submits post with @alice mention

Validation at T=5s:
  Check blocked_by:alice -> contains carol -> mention REJECTED
  Post published without @alice mention (rendered as plain text)
  Alice never notified. Correct behavior.

The real-time block check at submission time prevents this race.
If the check were cached (stale), mention could slip through.
Solution: Always check block list from Redis (real-time, not cached).
```

### Race Condition: Username Changed After Mention Typed

```
T=0:    Carol types "@alice_old" in post
T=1s:   Alice changes username to "alice_new"
T=5s:   Carol submits post with "@alice_old"

Server validation:
  SELECT user_id FROM users WHERE username = 'alice_old' -> NOT FOUND!
  
  Options:
  a. Mention fails silently (rendered as plain text) <- simple
  b. Client stores user_id at autocomplete selection time,
     server uses user_id (not username) for validation <- better

Recommended: Client sends both username AND user_id in mention metadata.
  Server validates by user_id (immutable), renders with current username.
  If user_id valid but username changed -> still works correctly.
```

---

## 8. Deep Dive: Engineering Trade-offs

### Text Mention Storage: Inline vs Separate Table

```
Inline (in post text):
  Post text stored as: "Great photo with @alice and @bob!"
  Rendering: regex parse @mentions -> replace with clickable links
  
  Problem: If alice changes username -> all old posts show wrong username
  Solution: Store mention metadata separately with user_id
  
  Post text: "Great photo with @{user:u-alice} and @{user:u-bob}!"
  Rendering: replace {user:u-alice} with current display name "Alice Smith"

Separate table (recommended):
  Post text stored as-is: "Great photo with @alice and @bob!"
  Mentions table: (post_id, user_id, offset, length)
  
  Rendering: fetch mentions for post -> overlay clickable links at offsets
  Username change: no impact (rendering always uses current username from user table)
  Deletion: user removes tag -> DELETE from mentions -> text stays but link removed
```

### Notification Grouping for Mentions

```
Scenario: Alice is mentioned in 50 comments on the same post within 1 minute

Without grouping: 50 push notifications -> notification spam -> user disables notifications

With grouping:
  First mention: "Bob mentioned you in a comment"
  2nd-5th: individual notifications
  6th+: "Bob, Carol, and 45 others mentioned you in comments on this post"
  
  Implementation:
    Notification service buffers mentions for same (user, post) pair
    Window: 5 minutes
    After window: send ONE grouped notification with count
    
    Redis: INCR mention_buffer:{mentioned_user}:{post_id} with TTL=300
    If count == 1 -> send individual notification
    If count > 5 -> wait for window to close -> send grouped
```

### @everyone and @channel — Group Mentions

```
Slack-style: @everyone mentions all users in a channel
  
  Challenge: Channel has 10K users -> 10K mention records + 10K notifications
  
  Solution: Don't create individual mention records
    Store as: mention_type = "group", target = "channel:general"
    Notification: broadcast to all channel members via WebSocket
    No individual records in mentions table (would be 10K rows per message!)
    
  Permission: Only admins/owners can use @everyone (prevents spam)
  Suppression: Users can mute @everyone notifications per channel
```

### Mention Analytics and Engagement Tracking

```
Track for each mention:
  - Was the notification opened?
  - Did the mentioned user visit the post?
  - Did they engage (like, comment, share)?
  
  Kafka event: { mention_id, action: "notification_opened" | "post_visited" | "engaged" }
  -> ClickHouse for analytics dashboard
  
  Metrics:
    Mention-to-visit rate: % of mentions that led to post visits
    Mention engagement rate: % of mentioned users who engaged
    Used for: spam detection (low engagement -> likely spam mentions)
```

