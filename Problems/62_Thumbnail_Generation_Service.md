# 62. Design a Thumbnail Generation Service

---

## 1. Functional Requirements (FR)

- **Auto-generate thumbnails**: Create thumbnails from uploaded videos, images, PDFs, and documents
- **Multiple thumbnail sizes**: Generate various sizes (small 150×150, medium 300×200, large 640×360) per content
- **Video thumbnails**: Extract the "best" frame from a video as thumbnail
- **Sprite sheets**: Generate sprite sheets for video seek preview (scrubbing)
- **Custom thumbnails**: Allow users to upload custom thumbnails or select from auto-generated candidates
- **A/B test thumbnails**: Generate multiple candidates; serve variants and measure click-through rate (CTR)
- **Animated thumbnails**: Generate short GIF/WebP previews from videos (hover-to-preview)
- **Batch regeneration**: Regenerate thumbnails when algorithms improve
- **Format optimization**: Serve in WebP/AVIF with JPEG fallback

---

## 2. Non-Functional Requirements (NFRs)

- **Speed**: Thumbnails ready within 30 seconds of content upload (< 5 seconds for images)
- **Quality**: Thumbnails should be visually appealing (sharp, well-composed, representative)
- **Scale**: Process 50K+ videos/hour and 500K+ images/hour
- **Durability**: Generated thumbnails stored reliably; but regenerable from originals
- **Cacheability**: Thumbnails highly cacheable on CDN (immutable content-addressable URLs)
- **Cost Efficient**: Minimize compute by avoiding unnecessary regeneration
- **Availability**: 99.9% — missing thumbnails degrade UX but aren't critical

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Videos uploaded / hour | 50K |
| Images uploaded / hour | 500K |
| Thumbnails per video | 5 (1 primary + 4 candidates + 1 sprite sheet) |
| Thumbnails per image | 3 sizes (small, medium, large) |
| Total thumbnails / hour | 50K × 6 + 500K × 3 = 1.8M |
| Thumbnails / sec | 500 |
| Avg thumbnail size | 20 KB |
| Thumbnail storage / day | 1.8M × 24 × 20 KB = 864 GB |
| Sprite sheet size (per video) | 500 KB |
| CDN bandwidth for thumbnails | ~100 Gbps |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐    ┌──────────────┐
│  Video       │    │  Image       │
│  Upload      │    │  Upload      │
│  Service     │    │  Service     │
└──────┬───────┘    └──────┬───────┘
       │                    │
       └─────────┬──────────┘
                 │
          ┌──────▼───────┐
          │    Kafka     │
          │ (thumbnail   │
          │  requests)   │
          └──────┬───────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼────┐  ┌───▼────┐  ┌───▼────┐
│ Video  │  │ Image  │  │ Doc    │
│ Thumb  │  │ Thumb  │  │ Thumb  │
│ Worker │  │ Worker │  │ Worker │
│        │  │        │  │ (PDF,  │
│ FFmpeg │  │ libvips│  │ PPTX)  │
│ + ML   │  │        │  │        │
└───┬────┘  └───┬────┘  └───┬────┘
    │            │            │
    └────────────┼────────────┘
                 │
          ┌──────▼───────┐
          │   S3         │
          │  /thumbnails │
          └──────┬───────┘
                 │
          ┌──────▼───────┐    ┌──────────────┐
          │  Metadata    │    │     CDN      │
          │  DB (MySQL)  │    │  (serve      │
          │  + Redis     │    │   thumbnails │
          │              │    │   at edge)   │
          └──────────────┘    └──────────────┘
```

### Component Deep Dive

#### Video Thumbnail Selection — Finding the "Best" Frame

```
Problem: A 10-minute video has 18,000 frames (at 30fps). Which ONE becomes the thumbnail?

Naive approach: Extract frame at 50% mark
  ✗ Might be a transition, black frame, or uninteresting moment

Multi-criteria scoring approach ⭐:

Step 1: Extract N candidate frames (uniform sampling + key moments)
  - Uniform: extract 1 frame every 10 seconds → 60 candidates for 10-min video
  - Scene changes: detect scene boundaries (FFmpeg scenedetect) → first frame of each scene
  - High-motion peaks: frames during action sequences
  Total candidates: ~100 frames

Step 2: Score each candidate (0-100):

  a. Sharpness (weight: 25%)
     Laplacian variance of grayscale image
     Higher variance = sharper (not blurry)
     score_sharp = normalize(laplacian_var, min=0, max=1000)
  
  b. Brightness (weight: 15%)
     Average pixel luminance
     Optimal: between 80-180 (out of 255)
     Too dark or too bright → penalize
     score_bright = 1 - abs(avg_luminance - 130) / 130
  
  c. Contrast (weight: 10%)
     Standard deviation of pixel values
     Higher = more visual interest
  
  d. Face presence (weight: 30%)
     Run face detection (OpenCV Haar or MTCNN)
     Frames with faces → strong preference (humans are drawn to faces)
     score_face = 1.0 if face detected, 0.0 otherwise
     Bonus: face should be > 10% of frame area (close-up > distant)
  
  e. Aesthetic quality (weight: 20%)
     ML model trained on "good thumbnail" vs "bad thumbnail" dataset
     Input: frame → Output: aesthetic_score (0-1)
     Training data: YouTube thumbnails with high CTR → "good"
                     Random frames → "bad"

  total_score = 0.25 × score_sharp + 0.15 × score_bright + 0.10 × score_contrast 
              + 0.30 × score_face + 0.20 × score_aesthetic

Step 3: Select top 3-5 candidates
  - #1 becomes the default thumbnail
  - Others offered as alternatives (user can choose)
  - Or: A/B test all candidates → winner becomes default

FFmpeg for candidate extraction:
  # Extract 1 frame per 10 seconds
  ffmpeg -i input.mp4 -vf "fps=1/10" -q:v 2 candidates/frame_%04d.jpg
  
  # Extract scene-change frames
  ffmpeg -i input.mp4 -vf "select='gt(scene,0.4)'" -vsync vfn candidates/scene_%04d.jpg
```

#### Sprite Sheet Generation — Video Seek Preview

```
When user hovers over video timeline → show thumbnail of that moment

Implementation:
  1. Extract 1 frame per second (or per 5 seconds for long videos)
     10-min video → 600 frames (1/sec) or 120 frames (1/5sec)
  
  2. Resize each frame to small size (160×90 pixels)
  
  3. Arrange in a grid (sprite sheet):
     10 columns × 12 rows = 120 thumbnails per sheet
     Sheet size: 1600 × 1080 pixels
     File size: ~200-500 KB (JPEG quality 60)
  
  4. Generate VTT metadata file:
     WEBVTT
     
     00:00:00.000 --> 00:00:05.000
     sprite_0.jpg#xywh=0,0,160,90
     
     00:00:05.000 --> 00:00:10.000
     sprite_0.jpg#xywh=160,0,160,90
     
     00:00:10.000 --> 00:00:15.000
     sprite_0.jpg#xywh=320,0,160,90
     ...

  5. Client loads sprite sheet + VTT → renders correct region at cursor position

Why sprite sheet instead of individual thumbnails?
  Individual: 120 separate HTTP requests → slow, high overhead
  Sprite sheet: 1 HTTP request → load once, render from memory
  CDN: 1 cached file vs 120 cached files → better cache efficiency

FFmpeg command:
  # Extract frames at 1 per 5 seconds, resize to 160x90
  ffmpeg -i input.mp4 -vf "fps=1/5,scale=160:90" -q:v 5 frames/f_%04d.jpg
  
  # Combine into sprite sheet using ImageMagick
  montage frames/f_*.jpg -tile 10x -geometry 160x90+0+0 sprite.jpg
```

#### Animated Thumbnail (Hover Preview) — YouTube-Style

```
User hovers over video thumbnail → shows 6-second animated preview

Generation:
  1. Select 3 interesting segments from the video (each 2 seconds):
     - 20% mark: early content (intro hook)
     - 50% mark: middle content
     - 80% mark: climax/conclusion
  
  2. Extract frames at these positions:
     ffmpeg -ss 120 -t 2 -i input.mp4 -vf "fps=10,scale=320:-1" frames_1/f_%03d.jpg
     ffmpeg -ss 300 -t 2 -i input.mp4 -vf "fps=10,scale=320:-1" frames_2/f_%03d.jpg
     ffmpeg -ss 480 -t 2 -i input.mp4 -vf "fps=10,scale=320:-1" frames_3/f_%03d.jpg
  
  3. Combine into animated WebP (smaller than GIF):
     # 60 frames (3 × 2sec × 10fps) → 6 second loop
     ffmpeg -framerate 10 -i frames_all/f_%03d.jpg \
       -vf "scale=320:-1" -loop 0 -quality 60 preview.webp
  
  File size: ~200-400 KB per animated preview (WebP)
  GIF equivalent: ~1-2 MB (4× larger!)

  4. Serve via CDN:
     On hover: <img src="preview.webp" /> replaces static thumbnail
     Preload: start loading animated preview when mouse is near (not on hover → too slow)

Bandwidth consideration:
  YouTube has billions of video thumbnails on screen
  If every hover loads 300 KB → massive bandwidth
  
  Optimization:
  - Only generate for top 10% most-viewed videos
  - Lazy load: only fetch when mouse enters thumbnail area
  - Prefetch: browser idle time → prefetch previews for visible thumbnails
  - Quality tiers: 180p for mobile, 320p for desktop
```

#### A/B Testing Thumbnails — Measuring CTR

```
YouTube's thumbnail A/B testing system:

1. Generate 3 thumbnail candidates per video
   - AI-selected best frames
   - Or: creator uploads 3 custom thumbnails

2. Serve variants:
   Viewer A sees thumbnail_1
   Viewer B sees thumbnail_2
   Viewer C sees thumbnail_3
   
   Split: even distribution across viewers (not random per request!)
   User hash: hash(user_id) % 3 → consistent variant per user

3. Measure CTR per variant:
   impressions: how many times thumbnail was shown
   clicks: how many times viewer clicked to watch
   CTR = clicks / impressions
   
   Also measure: watch time after click (avoid clickbait thumbnails)
   Combined metric: CTR × avg_watch_time → "effective CTR"

4. Statistical significance:
   Run test until each variant has > 10K impressions
   Chi-squared test: is the difference statistically significant?
   Typical test duration: 24-48 hours

5. Winner selection:
   If variant_2 has 15% higher effective CTR → auto-promote to default
   Update: image_thumbnails SET is_default = true WHERE variant = 2

Storage:
  thumbnail_variants table:
    (video_id, variant_index, s3_url, impressions, clicks, avg_watch_time)
  
  Redis for serving:
    thumb_variant:{video_id}:{user_hash % 3} → CDN URL of variant
```

---

## 5. APIs

### Generate Thumbnails for Video
```http
POST /api/v1/thumbnails/video
{
  "video_id": "vid-uuid",
  "s3_key": "originals/vid-uuid/video.mp4",
  "options": {
    "sizes": ["150x150", "300x200", "640x360"],
    "candidates": 5,
    "sprite_sheet": true,
    "animated_preview": true,
    "format": "webp"
  }
}
Response: 202 Accepted
{
  "job_id": "job-uuid",
  "status": "processing",
  "estimated_completion": "2025-03-14T11:01:00Z"
}
```

### Get Thumbnails for Content
```http
GET /api/v1/thumbnails/{content_id}
Response: 200 OK
{
  "content_id": "vid-uuid",
  "thumbnails": {
    "default": "https://cdn.example.com/thumbs/vid-uuid/default_640x360.webp",
    "small": "https://cdn.example.com/thumbs/vid-uuid/small_150x150.webp",
    "medium": "https://cdn.example.com/thumbs/vid-uuid/medium_300x200.webp"
  },
  "candidates": [
    {"index": 0, "url": "https://cdn.example.com/thumbs/vid-uuid/candidate_0.webp", "score": 0.92},
    {"index": 1, "url": "https://cdn.example.com/thumbs/vid-uuid/candidate_1.webp", "score": 0.87}
  ],
  "sprite_sheet": {
    "url": "https://cdn.example.com/thumbs/vid-uuid/sprite.jpg",
    "vtt_url": "https://cdn.example.com/thumbs/vid-uuid/sprite.vtt",
    "columns": 10, "rows": 12, "thumb_width": 160, "thumb_height": 90
  },
  "animated_preview": "https://cdn.example.com/thumbs/vid-uuid/preview.webp"
}
```

### Select Custom Thumbnail
```http
PUT /api/v1/thumbnails/{content_id}/select
{
  "candidate_index": 2
}
Response: 200 OK
{ "thumbnail_url": "https://cdn.example.com/thumbs/vid-uuid/candidate_2.webp" }
```

### Upload Custom Thumbnail
```http
POST /api/v1/thumbnails/{content_id}/custom
{
  "image_data": "base64_encoded..."
}
Response: 200 OK
{ "thumbnail_url": "https://cdn.example.com/thumbs/vid-uuid/custom.webp" }
```

---

## 6. Data Model

### MySQL — Thumbnail Metadata

```sql
CREATE TABLE thumbnails (
    thumbnail_id    BIGINT PRIMARY KEY AUTO_INCREMENT,
    content_id      VARCHAR(36) NOT NULL,
    content_type    ENUM('video', 'image', 'document') NOT NULL,
    variant_type    ENUM('default', 'candidate', 'custom', 'sprite', 'animated') NOT NULL,
    candidate_index SMALLINT,
    s3_key          TEXT NOT NULL,
    cdn_url         TEXT NOT NULL,
    format          VARCHAR(10) DEFAULT 'webp',
    width           INT,
    height          INT,
    file_size_bytes INT,
    quality_score   DECIMAL(4,3),              -- ML-computed aesthetic score
    source_timestamp DECIMAL(10,3),            -- video timestamp this frame was extracted from
    is_default      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_content (content_id, variant_type),
    INDEX idx_default (content_id, is_default)
);

CREATE TABLE thumbnail_ab_tests (
    test_id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    content_id      VARCHAR(36) NOT NULL,
    variant_index   SMALLINT NOT NULL,
    impressions     INT DEFAULT 0,
    clicks          INT DEFAULT 0,
    total_watch_seconds BIGINT DEFAULT 0,
    status          ENUM('running', 'completed') DEFAULT 'running',
    winner          BOOLEAN DEFAULT FALSE,
    started_at      TIMESTAMP,
    ended_at        TIMESTAMP,
    INDEX idx_content (content_id, status)
);

CREATE TABLE sprite_sheets (
    sprite_id       BIGINT PRIMARY KEY AUTO_INCREMENT,
    content_id      VARCHAR(36) NOT NULL UNIQUE,
    s3_key          TEXT NOT NULL,
    vtt_s3_key      TEXT NOT NULL,
    columns         SMALLINT,
    rows            SMALLINT,
    thumb_width     SMALLINT,
    thumb_height    SMALLINT,
    interval_seconds DECIMAL(5,2),             -- time between consecutive thumbnails
    created_at      TIMESTAMP
);
```

### S3 — Thumbnail Storage

```
Bucket: thumbnails (CDN-served, lifecycle: S3 IA after 90 days)
  /{content_id}/default_640x360.webp
  /{content_id}/small_150x150.webp
  /{content_id}/medium_300x200.webp
  /{content_id}/candidate_0.webp
  /{content_id}/candidate_1.webp
  /{content_id}/candidate_2.webp
  /{content_id}/sprite_sheet.jpg
  /{content_id}/sprite.vtt
  /{content_id}/animated_preview.webp
  /{content_id}/custom.webp
```

### Redis — Serving Cache

```
# Default thumbnail URL per content (hot cache)
thumb:{content_id}           → CDN URL of default thumbnail
TTL: 86400

# A/B test variant assignment
thumb_ab:{content_id}:{user_hash}  → variant_index
TTL: 604800 (7 days)

# Thumbnail generation job status
thumb_job:{job_id}           → Hash { status, progress, content_id }
TTL: 3600
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **FFmpeg crash** | Retry with exponential backoff; 3 attempts; DLQ for persistent failures |
| **Corrupt video frame** | Skip corrupt segment; use adjacent frame instead |
| **ML model failure** | Fall back to rule-based scoring (sharpness + brightness only) |
| **S3 upload failure** | Retry with jitter; S3 handles concurrent uploads via versioning |
| **Missing thumbnail** | CDN serves placeholder image; queue regeneration |
| **Worker pool exhaustion** | Auto-scale based on Kafka consumer lag; alert if lag > 5 minutes |
| **Sprite sheet too large** | Reduce interval (1/10 sec instead of 1/sec) for long videos (> 2 hours) |

### Specific: Content-Addressable Thumbnails for Cache Efficiency

```
Problem: Same video re-uploaded by different users → generate identical thumbnails

Solution: Content-addressable URLs
  1. Hash the thumbnail content: sha256(thumbnail_bytes) → abc123def
  2. S3 key: /thumbnails/{sha256}.webp
  3. CDN URL: https://cdn.example.com/t/abc123def.webp
  
  Benefits:
  - Duplicate uploads → same hash → same S3 object → no extra storage
  - Immutable URL: thumbnail content never changes for a given URL
  - CDN: can set Cache-Control: max-age=31536000, immutable
    (cache for 1 year — content at this URL will never change)
  - CDN hit rate: near 100% for popular thumbnails
  
  If user changes thumbnail:
  - New thumbnail → new hash → new URL → CDN caches new version
  - Old thumbnail URL still works (immutable) but nothing links to it
  - Old thumbnail cleaned up after 30 days (no references)
```

---

## 8. Deep Dive: Engineering Trade-offs

### FFmpeg Seek: Input Seeking vs Output Seeking

```
For extracting a frame at timestamp 5:00:

Input seeking (fast) ⭐:
  ffmpeg -ss 300 -i input.mp4 -frames:v 1 thumb.jpg
  
  -ss BEFORE -i: seeks using keyframes (very fast, ~50 ms)
  Accuracy: may be off by up to 1 GOP (0.5-2 seconds)
  Good enough for thumbnails (exact timestamp doesn't matter)

Output seeking (precise but slow):
  ffmpeg -i input.mp4 -ss 300 -frames:v 1 thumb.jpg
  
  -ss AFTER -i: decodes from start to timestamp (very slow!)
  A 2-hour video seeking to 1:00:00 → decodes 1 hour of video
  Time: 30-60 seconds just to extract ONE frame
  Accurate: frame-exact timestamp

For thumbnail generation:
  Use input seeking (fast). Timestamp accuracy of ±1 second is irrelevant.
  
  But: if generating sprite sheet (frame every 5 seconds):
    Option 1: 120 separate ffmpeg calls with input seeking → 120 × 50ms = 6 seconds
    Option 2: Single ffmpeg pass with fps filter → decode once, extract all:
      ffmpeg -i input.mp4 -vf "fps=1/5,scale=160:90" frame_%04d.jpg
      Decodes full video once → extracts frames during decode → 15 seconds for 10-min video
    
    Option 2 is faster for sprite sheets (single decode pass)
```

### Thumbnail Quality: Perceptual Hashing for Dedup

```
Problem: Different frames may look nearly identical
  Frame at 5:00.0 and 5:02.0 → same scene, almost same image
  Don't want 2 nearly-identical thumbnail candidates

Solution: Perceptual hash (pHash) deduplication

  For each candidate frame:
    1. Resize to 32×32 grayscale
    2. Apply DCT (Discrete Cosine Transform)
    3. Keep top-left 8×8 DCT coefficients
    4. Compare each to median → generate 64-bit hash
    
  Hamming distance between two pHashes:
    < 5 bits different → nearly identical images
    5-10 bits → somewhat similar
    > 10 bits → different images
  
  Filter: if candidate's pHash is within 5 bits of any selected candidate → skip
  Result: diverse, non-redundant thumbnail candidates
```

### Thumbnail CDN Serving: Content Negotiation for Format

```
Same thumbnail, optimal format per client:

Approach 1: URL-based format selection
  /thumbs/{id}.webp → WebP version
  /thumbs/{id}.avif → AVIF version
  /thumbs/{id}.jpg  → JPEG fallback
  
  Client selects format based on browser support
  CDN caches each URL separately → simple, no Vary header

Approach 2: Accept header negotiation
  Client: Accept: image/avif, image/webp, image/*
  Server returns best supported format
  CDN: Vary: Accept → caches per format
  
  Problem: Vary: Accept reduces CDN cache efficiency
  (each combination of Accept header → separate cache entry)

Approach 3: Client detection at CDN edge ⭐
  CDN edge (CloudFront/Cloudflare) inspects User-Agent
  Chrome → serve WebP or AVIF
  Safari 16+ → serve WebP
  Safari 15 → serve JPEG
  
  No Vary header needed → excellent cache efficiency
  Implementation: CDN Lambda@Edge or Cloudflare Workers

Recommended: Approach 1 (URL-based) for simplicity and cache efficiency
  Client knows its own format support → requests correct URL
  CDN caches each format variant by URL → near-100% hit rate
```



