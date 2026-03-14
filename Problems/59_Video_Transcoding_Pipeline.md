# 59. Design a Video Transcoding Pipeline

---

## 1. Functional Requirements (FR)

- **Ingest videos**: Accept uploaded video files in any format (MP4, AVI, MOV, MKV, WebM)
- **Multi-resolution transcoding**: Convert to multiple resolutions (240p, 360p, 480p, 720p, 1080p, 4K)
- **Multi-codec support**: Encode in H.264, H.265/HEVC, VP9, AV1
- **Adaptive bitrate packaging**: Package into HLS (.m3u8 + .ts segments) and DASH (.mpd + .m4s segments)
- **Audio processing**: Extract, normalize, and transcode audio (AAC, Opus) at multiple bitrates
- **Subtitle extraction**: Auto-generate subtitles via speech-to-text; support uploaded subtitle files
- **Thumbnail generation**: Extract keyframes, generate sprite sheets for seek preview
- **DRM encryption**: Encrypt segments with Widevine/FairPlay/PlayReady
- **Watermarking**: Forensic or visible watermarking for content protection
- **Progress tracking**: Real-time progress reporting for upload → transcode → ready
- **Priority queues**: Premium content (paid creators) gets transcoded faster

---

## 2. Non-Functional Requirements (NFRs)

- **Throughput**: Process 10,000+ videos/hour (YouTube uploads 500 hours/minute of video)
- **Latency**: Standard video ready for playback within 30 minutes of upload; short videos (< 5 min) within 5 minutes
- **Durability**: Original uploaded file NEVER lost; transcoded outputs can be regenerated
- **Fault Tolerance**: Any step failure → retry that step, not the entire pipeline
- **Cost Efficient**: GPU transcoding for H.265/AV1; CPU for H.264; spot instances for non-urgent work
- **Scalability**: Auto-scale based on queue depth; handle viral upload spikes
- **Quality**: Output quality comparable to or better than input (no unnecessary quality loss)
- **Idempotent**: Re-running a failed job produces the same output

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Videos uploaded / hour | 10,000 |
| Avg original video duration | 10 minutes |
| Avg original file size | 1 GB |
| Upload storage / day | 240 TB |
| Transcoded variants per video | 6 resolutions × 2 codecs = 12 |
| Expansion factor (transcoded/original) | ~3× (multiple resolutions, but lower bitrate) |
| Transcoded storage / day | 720 TB |
| CPU-hours per video (H.264, all resolutions) | ~2 CPU-hours |
| GPU-hours per video (H.265/AV1) | ~0.5 GPU-hours |
| Total compute / day | ~20K CPU-hours + ~5K GPU-hours |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│   Upload     │
│   Client     │
└──────┬───────┘
       │ (Pre-signed S3 URL)
┌──────▼───────┐
│   Object     │
│   Store (S3) │ ← Original files stored here (NEVER deleted)
│   /originals │
└──────┬───────┘
       │ S3 Event Notification
┌──────▼───────────────────────────────────────────────────┐
│                    Pipeline Orchestrator                   │
│              (Step Functions / Temporal / Airflow)         │
│                                                           │
│   ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌──────┐│
│   │  Pre-   │    │ Transcode│    │ Package │    │ Post ││
│   │ process │───▶│ (Parallel│───▶│ HLS/    │───▶│ proc ││
│   │         │    │  per     │    │ DASH    │    │      ││
│   │ Probe,  │    │  reso-   │    │         │    │ DRM, ││
│   │ Split   │    │  lution) │    │         │    │ thumb││
│   └─────────┘    └──────────┘    └─────────┘    └──────┘│
│                                                           │
│   DAG Execution Engine                                    │
└──────┬───────────────────────────────────────────────────┘
       │
       │ Parallel tasks distributed to:
       │
┌──────▼───────────────────────────────────┐
│         Worker Pool (Auto-scaling)        │
│                                           │
│  ┌─────────────┐  ┌─────────────────────┐│
│  │ CPU Workers │  │  GPU Workers        ││
│  │ (H.264,    │  │  (H.265, AV1,      ││
│  │  audio,    │  │   ML tasks)         ││
│  │  thumbnail)│  │                     ││
│  │            │  │  EC2 g5 / p4d       ││
│  │ EC2 c6i   │  │  (GPU instances)     ││
│  │ (compute   │  │                     ││
│  │  optimized)│  │                     ││
│  └─────────────┘  └─────────────────────┘│
└──────────────────────────────────────────┘
       │
┌──────▼───────┐    ┌──────────────┐    ┌──────────────┐
│   Object     │    │  Metadata    │    │  CDN Origin  │
│   Store (S3) │    │  DB (MySQL)  │    │  (serve      │
│   /transcoded│    │  (video      │    │   completed  │
│              │    │   status,    │    │   videos)    │
│              │    │   manifests) │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
```

### Component Deep Dive

#### Pipeline DAG — Task Dependencies

```
For each uploaded video, the pipeline executes this DAG:

                    ┌──────────┐
                    │  Upload  │
                    │ Complete │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │  Probe   │  ← FFprobe: detect codec, resolution, duration, audio tracks
                    │ (Analyze)│
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │  Split   │  ← Split video into 10-second segments (GOP-aligned)
                    │ (Segment)│     Enables parallel transcoding per segment
                    └────┬─────┘
                         │
          ┌──────────────┼──────────────┬──────────────┐
          │              │              │              │
     ┌────▼────┐   ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
     │Transcode│   │Transcode│   │Transcode│   │Transcode│
     │  240p   │   │  480p   │   │  720p   │   │  1080p  │
     │ H.264   │   │ H.264   │   │ H.264   │   │ H.264   │
     └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘
          │              │              │              │
          │         (also H.265 / AV1 variants in parallel)
          │              │              │              │
          └──────────────┼──────────────┘──────────────┘
                         │
                    ┌────▼─────┐
                    │ Reassemble│  ← Merge transcoded segments per resolution
                    │ Segments  │
                    └────┬─────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
     │ Package │   │  Audio  │   │Thumbnail│
     │ HLS +   │   │ Extract │   │ + Sprite│
     │ DASH    │   │ + Trans │   │ Sheet   │
     └────┬────┘   └────┬────┘   └────┬────┘
          │              │              │
          └──────────────┼──────────────┘
                         │
                    ┌────▼─────┐
                    │   DRM    │
                    │ Encrypt  │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ Content  │  ← ML-based NSFW/violence detection
                    │Moderation│
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │  Ready!  │  ← Update DB status; notify user; CDN can serve
                    │ Publish  │
                    └──────────┘

Key insight: Transcoding to different resolutions is EMBARRASSINGLY PARALLEL.
  10-minute video → 60 segments × 6 resolutions = 360 independent tasks
  Each runs on a separate worker → total wall-clock time ≈ 1 segment time
  Instead of 2 CPU-hours sequential → ~3 minutes wall-clock with 40 workers!
```

#### Segment-Based Parallel Transcoding — The Core Optimization

```
Why split into segments before transcoding?

Without splitting:
  1 video × 6 resolutions = 6 tasks
  Each task: 10 minutes of video → 20 minutes transcode time
  Wall-clock: 20 minutes (even with 6 workers, each handles one resolution)

With splitting (10-second segments):
  60 segments × 6 resolutions = 360 tasks
  Each task: 10 seconds of video → 3 seconds transcode time
  With 360 workers: wall-clock ≈ 3 seconds!
  With 40 workers (realistic): ~30 seconds wall-clock

GOP-aligned splitting:
  GOPs (Group of Pictures): video is encoded in groups starting with a keyframe (I-frame)
  Must split at GOP boundaries (I-frame positions)
  If you split mid-GOP → decoder can't start decoding that segment
  
  FFmpeg: ffmpeg -i input.mp4 -c copy -f segment -segment_time 10 
          -reset_timestamps 1 segment_%03d.mp4
  
  Flags:
    -c copy: don't re-encode during splitting (fast, lossless)
    -segment_time 10: target 10 seconds per segment (actual split at nearest GOP)
    -reset_timestamps 1: each segment starts at timestamp 0

Reassembly after transcoding:
  FFmpeg concat demuxer:
    file 'segment_000_720p.ts'
    file 'segment_001_720p.ts'
    ...
  ffmpeg -f concat -i list.txt -c copy output_720p.ts
  
  For HLS: segments ARE the final output (no reassembly needed!)
    Each transcoded segment → directly becomes an HLS .ts segment
    Just generate the .m3u8 playlist pointing to all segments
```

#### FFmpeg Command Breakdown — What Actually Runs

```
H.264 encoding (CPU, c6i instance):
  ffmpeg -i segment_005.mp4 \
    -vf "scale=1280:720" \
    -c:v libx264 \
    -preset medium \         # Balance speed/quality (faster/medium/slow/veryslow)
    -crf 23 \                # Constant Rate Factor (18=high quality, 23=default, 28=low)
    -profile:v high \        # H.264 profile (baseline/main/high)
    -level 4.0 \             # Compatibility level
    -maxrate 3M \            # Max bitrate (for ABR compliance)
    -bufsize 6M \            # Buffer size (2× maxrate)
    -g 48 \                  # Keyframe interval (match segment duration)
    -sc_threshold 0 \        # Disable scene change keyframes (consistent segments)
    -an \                    # No audio (process audio separately)
    segment_005_720p.ts

H.265 encoding (GPU, g5 instance with NVIDIA):
  ffmpeg -i segment_005.mp4 \
    -vf "scale=1280:720" \
    -c:v hevc_nvenc \        # NVIDIA GPU encoder
    -preset p5 \             # NVENC preset (p1=fastest, p7=quality)
    -rc:v vbr \              # Variable bitrate
    -cq 28 \                 # Constant quality parameter
    -maxrate 1.5M \          # 50% less bitrate than H.264 for same quality!
    -bufsize 3M \
    -g 48 \
    -tag:v hvc1 \            # Required for Apple compatibility
    segment_005_720p_h265.ts

AV1 encoding (GPU with SVT-AV1 or AOM):
  ffmpeg -i segment_005.mp4 \
    -vf "scale=1280:720" \
    -c:v libsvtav1 \         # SVT-AV1 encoder (fastest AV1 encoder)
    -preset 6 \              # 0=slowest/best, 12=fastest/worst
    -crf 30 \                
    -svtav1-params "tune=0" \
    -g 48 \
    segment_005_720p_av1.mp4
    
  AV1 encoding is 10-50× slower than H.264 → only encode popular videos in AV1
  Or use hardware AV1 encoders (NVIDIA Ada Lovelace GPUs)
```

#### Pipeline Orchestrator — Temporal/Step Functions

```
Why a workflow orchestrator (not just Kafka consumers)?

Kafka consumers:
  ✓ Great for simple event-driven processing
  ✗ Complex DAG dependencies hard to express
  ✗ No built-in retry with backoff per step
  ✗ No workflow visibility (which step is video X stuck on?)
  ✗ Partial failure handling: retry just the failed step, not the whole pipeline

Temporal ⭐ (recommended) or AWS Step Functions:
  ✓ DAG definition: declare task dependencies
  ✓ Per-step retries with exponential backoff
  ✓ Workflow state persisted: survives server restart
  ✓ Visibility: dashboard shows each video's pipeline progress
  ✓ Timeouts: if transcode takes > 1 hour → fail and alert
  ✓ Versioning: update pipeline definition without affecting in-flight workflows

Temporal workflow definition (pseudocode):
  @workflow
  def transcode_pipeline(video_id, s3_key):
      # Step 1: Analyze
      probe_result = await probe_video(s3_key)
      
      # Step 2: Split into segments
      segments = await split_video(s3_key, probe_result)
      
      # Step 3: Transcode all resolutions (PARALLEL)
      resolutions = determine_resolutions(probe_result.original_resolution)
      transcode_futures = []
      for res in resolutions:
          for segment in segments:
              future = transcode_segment.async(segment, res, codec='h264')
              transcode_futures.append(future)
      
      transcoded = await all(transcode_futures)  # Wait for ALL to complete
      
      # Step 4: Package HLS/DASH
      manifest = await package_hls_dash(transcoded, video_id)
      
      # Step 5: Parallel post-processing
      await all(
          generate_thumbnails.async(s3_key, probe_result),
          extract_audio.async(s3_key),
          content_moderation.async(s3_key),
          generate_subtitles.async(s3_key)
      )
      
      # Step 6: DRM encryption
      await encrypt_drm(manifest, video_id)
      
      # Step 7: Publish
      await update_video_status(video_id, 'ready')
      await notify_creator(video_id)

Each step has retry policy:
  @activity(retry_policy=RetryPolicy(
      initial_interval=timedelta(seconds=10),
      maximum_interval=timedelta(minutes=5),
      maximum_attempts=3,
      non_retryable_errors=[InvalidVideoFormat]
  ))
  def transcode_segment(segment, resolution, codec):
      ...
```

---

## 5. APIs

### Initiate Upload
```http
POST /api/v1/videos/upload
{
  "filename": "vacation.mp4",
  "content_type": "video/mp4",
  "file_size_bytes": 1073741824,
  "title": "Summer Vacation 2025"
}
Response: 200 OK
{
  "video_id": "vid-uuid",
  "upload_url": "https://s3.amazonaws.com/originals/vid-uuid/upload?X-Amz-...",
  "upload_id": "multipart-upload-id",
  "max_chunk_size": 104857600
}
```

### Check Transcoding Status
```http
GET /api/v1/videos/{video_id}/status
Response: 200 OK
{
  "video_id": "vid-uuid",
  "status": "transcoding",
  "pipeline_progress": {
    "probe": "completed",
    "split": "completed",
    "transcode": { "completed": 45, "total": 60, "percent": 75 },
    "package": "pending",
    "thumbnails": "completed",
    "moderation": "pending",
    "drm": "pending"
  },
  "estimated_completion": "2025-03-14T11:15:00Z",
  "started_at": "2025-03-14T10:45:00Z"
}
```

### Retry Failed Pipeline
```http
POST /api/v1/videos/{video_id}/retry
{
  "from_step": "transcode"     // retry from specific step (not the whole pipeline)
}
Response: 200 OK
{ "status": "retrying", "retry_from": "transcode" }
```

### Get Transcoding Presets
```http
GET /api/v1/presets
Response: 200 OK
{
  "presets": [
    {"name": "standard", "resolutions": ["480p","720p","1080p"], "codecs": ["h264"], "drm": false},
    {"name": "premium", "resolutions": ["480p","720p","1080p","4k"], "codecs": ["h264","h265","av1"], "drm": true},
    {"name": "fast", "resolutions": ["480p","720p"], "codecs": ["h264"], "drm": false}
  ]
}
```

---

## 6. Data Model

### MySQL — Video and Job Metadata

```sql
CREATE TABLE videos (
    video_id        VARCHAR(36) PRIMARY KEY,
    creator_id      VARCHAR(36) NOT NULL,
    title           VARCHAR(255),
    original_s3_key TEXT NOT NULL,
    original_format VARCHAR(10),
    duration_sec    INT,
    original_width  INT,
    original_height INT,
    original_codec  VARCHAR(20),
    original_bitrate_kbps INT,
    file_size_bytes BIGINT,
    status          ENUM('uploaded','probing','transcoding','packaging',
                         'moderating','ready','failed','removed') DEFAULT 'uploaded',
    preset          VARCHAR(20) DEFAULT 'standard',
    priority        ENUM('low','normal','high','urgent') DEFAULT 'normal',
    manifest_url    TEXT,
    thumbnail_url   TEXT,
    error_message   TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at    TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_creator (creator_id, created_at DESC)
);

CREATE TABLE transcoding_tasks (
    task_id         VARCHAR(36) PRIMARY KEY,
    video_id        VARCHAR(36) NOT NULL,
    task_type       ENUM('probe','split','transcode','package','thumbnail',
                         'audio','moderation','drm','subtitle'),
    resolution      VARCHAR(10),
    codec           VARCHAR(10),
    segment_index   INT,
    status          ENUM('pending','running','completed','failed','retrying'),
    worker_id       VARCHAR(36),
    s3_input_key    TEXT,
    s3_output_key   TEXT,
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    duration_ms     INT,
    error_message   TEXT,
    retry_count     INT DEFAULT 0,
    INDEX idx_video (video_id, task_type),
    INDEX idx_status (status),
    INDEX idx_worker (worker_id)
);
```

### S3 — Storage Structure

```
Bucket: video-originals (NEVER deleted, cross-region replicated)
  /{video_id}/original.mp4

Bucket: video-segments (temporary, auto-delete after 7 days)
  /{video_id}/segments/segment_000.mp4
  /{video_id}/segments/segment_001.mp4
  ...

Bucket: video-transcoded (long-term, lifecycle policies)
  /{video_id}/h264/
    720p/segment_000.ts
    720p/segment_001.ts
    720p/playlist.m3u8
    1080p/segment_000.ts
    ...
  /{video_id}/h265/
    720p/segment_000.ts
    ...
  /{video_id}/manifest.m3u8      (master playlist)
  /{video_id}/manifest.mpd       (DASH manifest)
  /{video_id}/thumbnails/
    thumb_001.jpg
    thumb_002.jpg
    sprite_sheet.jpg              (for seek preview)
  /{video_id}/subtitles/
    en.vtt
    es.vtt
  /{video_id}/audio/
    aac_128k.m4a
    aac_256k.m4a
```

### Redis — Pipeline State + Queue Management

```
# Video pipeline progress
pipeline:{video_id}   → Hash { 
  status, current_step, 
  transcode_completed, transcode_total,
  started_at, estimated_completion 
}
TTL: 86400

# Priority task queues (sorted sets, score = priority × timestamp)
task_queue:transcode:cpu  → Sorted Set { task_id: priority_score }
task_queue:transcode:gpu  → Sorted Set { task_id: priority_score }
task_queue:thumbnail      → Sorted Set
task_queue:moderation     → Sorted Set

# Worker heartbeats
worker:{worker_id}  → Hash { status, current_task, last_heartbeat }
TTL: 60
```

### Kafka Topics

```
Topic: video-uploaded        (trigger pipeline start)
Topic: pipeline-events       (step started/completed/failed — for monitoring)
Topic: video-ready           (consumed by CDN pre-warming, notification service)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Original file lost** | S3 cross-region replication; 11 nines durability; versioning enabled |
| **Transcode worker crash** | Temporal detects heartbeat timeout → re-schedule task on another worker |
| **Segment transcode failure** | Retry 3 times with backoff; after 3 failures → DLQ, alert ops |
| **S3 upload of transcoded segment fails** | Retry with exponential backoff; S3 multipart for large segments |
| **Entire pipeline failure** | Video status → "failed"; creator notified; manual retry available |
| **Worker pool exhaustion** | Auto-scaling based on queue depth; alert if queue depth > 1000 for > 10 min |
| **Corrupt input video** | Probe step detects invalid file → fail fast, notify creator |
| **Spot instance preemption** | Task checkpointing; preempted task re-queued automatically |

### Specific: Handling Spot Instance Preemption

```
GPU instances are expensive. Using spot instances saves 60-80%.
But AWS can reclaim spot instances with 2-minute warning.

Strategy:
  1. Use spot instances for transcoding workers (60-80% cost savings)
  2. All transcoding is segment-based → each segment is independent
  3. Segment transcode takes ~3-10 seconds → usually finishes before preemption
  4. If preempted mid-segment:
     a. Worker receives SIGTERM (2-minute warning)
     b. Worker marks current task as "interrupted" in Redis
     c. Temporal re-schedules task on another worker
     d. Wasted work: at most 1 segment (3-10 seconds of compute)
  
  5. Critical path (probe, package, publish): use on-demand instances (small, cheap)
     Only transcoding (compute-heavy) uses spot instances

  Cost comparison (1000 videos/hour):
    All on-demand: $150/hour
    Spot instances for transcode: $45/hour (70% savings)
    Savings: $76,650/month
```

### Specific: Quality Verification After Transcoding

```
Problem: Transcoded video may have artifacts, sync issues, or corruption

Automated quality checks after each transcode:

1. Duration check: transcoded duration == original duration ±0.5 seconds
   If mismatch → segment boundary issue → re-split and re-transcode

2. Frame count check: original_frames × (target_fps/original_fps) ≈ transcoded_frames
   Missing frames → transcode error → retry

3. VMAF (Video Multi-Method Assessment Fusion):
   Netflix's perceptual quality metric
   Compare original vs transcoded → VMAF score (0-100)
   Threshold: VMAF > 80 for 720p, > 85 for 1080p
   Below threshold → increase CRF quality and re-transcode

4. Audio-video sync: 
   Compare audio PTS (presentation timestamps) vs video PTS
   Drift > 50 ms → sync issue → re-mux

5. Black frame / freeze detection:
   FFmpeg: -vf "blackdetect=d=2:pic_th=0.98" → find 2+ second black frames
   If present in transcoded but not original → corruption

6. Bitrate compliance:
   Average bitrate should be within ±20% of target
   Max bitrate should not exceed maxrate parameter
   
These checks add ~5 seconds per video but catch 0.1% of transcoding errors
that would otherwise reach viewers.
```

---

## 8. Deep Dive: Engineering Trade-offs

### CRF vs CBR vs VBR: Bitrate Control Strategies

```
CBR (Constant Bitrate): Every second uses exactly the same bitrate
  ✓ Predictable file size
  ✓ Simple bandwidth planning
  ✗ Wastes bits on simple scenes (static shots use same bitrate as action scenes)
  ✗ Insufficient bits for complex scenes → quality drops
  Use case: Live streaming (need consistent bandwidth)

VBR (Variable Bitrate): Bitrate varies based on scene complexity
  ✓ Efficient: simple scenes use fewer bits, complex scenes use more
  ✓ Better perceptual quality at same average bitrate
  ✗ Less predictable file size
  Use case: VOD (pre-recorded content)

CRF (Constant Rate Factor) ⭐: Target constant QUALITY, let bitrate vary
  ✓ Best perceptual quality for a given average file size
  ✓ Simple: one parameter controls quality (CRF 18-28)
  ✗ File size completely unpredictable (1-minute scene could be 5 MB or 50 MB)
  Use case: VOD transcoding (YouTube, Netflix)
  
  CRF + maxrate (recommended):
    -crf 23 -maxrate 3M -bufsize 6M
    Quality-driven (CRF) with bitrate cap (maxrate)
    Ensures ABR player can predict bandwidth needs
    Best of both worlds

Netflix's per-title encoding:
  Don't use one CRF for all videos!
  Animated content (South Park): CRF 28 looks great (simple visuals)
  Action movie: CRF 20 needed for same quality (complex motion)
  
  Solution: Encode test segment at multiple CRFs → measure VMAF → pick optimal CRF
  Result: 20-40% bitrate savings over fixed-CRF approach
```

### Per-Title vs Per-Shot Encoding: Netflix's Approach

```
Traditional: Same encoding parameters for ALL videos
  720p always at 3 Mbps → wastes bits on animated shows, insufficient for action

Per-Title encoding:
  For each video: run convex hull analysis
    Test: CRF 20, 22, 24, 26, 28 at each resolution
    Measure: VMAF score vs bitrate for each
    Select: optimal CRF per resolution that maximizes VMAF/bitrate ratio
  
  Result: animated show at 720p needs only 1.5 Mbps; action movie needs 4 Mbps
  Same perceptual quality, up to 40% bandwidth savings

Per-Shot encoding (state of the art):
  Split video into "shots" (scene changes)
  Each shot gets its OWN encoding parameters
  
  Shot 1 (dialogue, static): CRF 28, low bitrate → looks great
  Shot 2 (car chase, motion): CRF 20, high bitrate → needs the bits
  Shot 3 (credits, text): CRF 30, very low bitrate
  
  Result: additional 10-20% savings over per-title
  Cost: significantly more analysis compute; worth it at Netflix scale

Implementation:
  1. Scene detection: FFmpeg -filter:v "select='gt(scene,0.4)'" → find scene changes
  2. For each scene: run VMAF-based CRF search (parallelizable)
  3. Encode each scene with optimal parameters
  4. Concatenate scenes into final segments
```

### Thumbnail Generation: More Complex Than You Think

```
Naive: Extract frame at 50% of video duration → use as thumbnail
  Problem: often captures a transition, blur, or uninteresting frame

Better approaches:

1. Multi-frame selection:
   Extract frames at 10%, 25%, 50%, 75%, 90% of duration
   Score each frame by:
     - Sharpness (Laplacian variance)
     - Brightness (not too dark, not too bright)
     - Face detection (prefer frames with faces)
     - Motion blur score (prefer still frames)
   Select highest-scoring frame

2. Sprite sheets (for seek preview):
   Extract 1 frame per second → arrange in grid image (10 × 30 = 300 thumbnails)
   Client loads sprite sheet → shows thumbnail at cursor position during seek
   File size: ~500 KB per sprite sheet (JPEG quality 60)
   
   VTT file mapping timestamps to sprite regions:
   WEBVTT
   00:00:00.000 --> 00:00:01.000
   sprite.jpg#xywh=0,0,160,90
   00:00:01.000 --> 00:00:02.000
   sprite.jpg#xywh=160,0,160,90

3. AI-generated thumbnails (YouTube):
   ML model trained on click-through rates
   "Which thumbnail gets the most clicks?"
   Generate 3-5 candidates → A/B test
   Features: faces, text overlay, bright colors, emotional expressions
```

### Cost Optimization: When to Encode Which Codec

```
Not all videos need all codecs. Encode based on predicted viewership:

Tier 1: All videos (immediately)
  H.264 at 480p, 720p, 1080p
  Cost: ~$0.02 per video
  Reason: H.264 plays everywhere; cover all devices

Tier 2: Videos with > 100 views in first hour
  H.265 at 720p, 1080p, 4K
  Cost: ~$0.08 per video (GPU)
  Reason: H.265 saves 40% bandwidth; ROI is positive if video gets views

Tier 3: Videos with > 10K views
  AV1 at all resolutions
  Cost: ~$0.50 per video (expensive encoding)
  Reason: AV1 saves 30% more bandwidth than H.265
  At 10K views × 5 min × 3 Mbps → 11 TB of bandwidth
  AV1 saves 3.3 TB → at $0.05/GB CDN cost → saves $165 per video
  Cost of encoding: $0.50 → ROI: 330×

Implementation:
  Upload → immediate H.264 transcoding
  Hourly cron: check view counts → queue H.265 for popular videos
  Daily cron: check view counts → queue AV1 for very popular videos
  
  Result: 95% of videos only get H.264 (cheap)
  Top 5% get H.265 (moderate cost, big bandwidth savings)
  Top 0.1% get AV1 (expensive encode, massive bandwidth savings)
```



