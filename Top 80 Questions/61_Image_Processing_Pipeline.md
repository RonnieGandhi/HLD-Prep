# 61. Design an Image Processing Pipeline

---

## 1. Functional Requirements (FR)

- **Image upload**: Accept images in any format (JPEG, PNG, WebP, HEIC, RAW, TIFF, BMP, GIF)
- **Format conversion**: Convert to web-optimized formats (WebP, AVIF) with fallback (JPEG)
- **Resizing**: Generate multiple sizes (thumbnail, small, medium, large, original) for responsive serving
- **Cropping**: Smart crop (face-aware, content-aware) and manual crop support
- **Filters & transforms**: Rotate, flip, brightness/contrast, blur, watermark
- **Content moderation**: Detect NSFW, violence, spam text in images (ML-based)
- **Metadata extraction**: EXIF data (GPS, camera, date), dominant colors, image quality score
- **Face detection/tagging**: Detect and tag faces in photos
- **CDN delivery**: Serve processed images via CDN with on-the-fly transformations
- **Batch processing**: Process thousands of images for catalog updates, migrations

---

## 2. Non-Functional Requirements (NFRs)

- **Throughput**: Process 50K+ images/sec (social media scale — Instagram processes 100M+ photos/day)
- **Latency**: Pre-processed image variants ready within 30 seconds of upload; on-demand transforms in < 200 ms
- **Durability**: Original images NEVER lost (11 nines)
- **Quality**: No perceptible quality loss in processed images (SSIM > 0.95)
- **Cost Efficient**: Storage optimized via format conversion (WebP 30% smaller than JPEG)
- **Scalability**: Handle viral upload spikes (10× normal load during events)
- **Security**: Strip sensitive EXIF data (GPS location) before serving public images
- **Idempotent**: Re-processing same image produces identical output

---

## 3. Capacity Estimations

| Metric | Value |
|---|---|
| Images uploaded / day | 100M (Instagram-scale) |
| Images / sec | ~1,200 |
| Avg original image size | 3 MB |
| Upload storage / day | 300 TB |
| Variants per image | 5 (thumb, small, medium, large, original) |
| Variant storage / day | 300 TB × 1.5 (multiple sizes but smaller) = 450 TB |
| On-demand transform requests / sec | 50K |
| CDN bandwidth for images | ~500 Gbps |

---

## 4. High-Level Design (HLD)

```
┌──────────────┐
│   Client     │
│   Upload     │
└──────┬───────┘
       │ Pre-signed S3 URL
┌──────▼───────┐
│   S3         │ ← Original images (never modified, cross-region replicated)
│   /originals │
└──────┬───────┘
       │ S3 Event / Kafka
       │
┌──────▼─────────────────────────────────────────────────┐
│              Image Processing Pipeline                  │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │ Validate │  │  Format  │  │  Resize  │  │Content ││
│  │ + Probe  │→ │  Convert │→ │ Generate │→ │Moder-  ││
│  │          │  │  (WebP,  │  │ Variants │  │ation   ││
│  │ Decode,  │  │  AVIF)   │  │          │  │        ││
│  │ EXIF     │  │          │  │ thumb,   │  │ NSFW,  ││
│  │ extract  │  │          │  │ sm, md,  │  │ face   ││
│  │          │  │          │  │ lg       │  │ detect ││
│  └──────────┘  └──────────┘  └──────────┘  └────────┘│
│                                                         │
│  Orchestrated by: Temporal / SQS + Lambda / K8s Jobs   │
└──────┬─────────────────────────────────────────────────┘
       │
┌──────▼───────┐    ┌──────────────┐    ┌──────────────┐
│   S3         │    │  Metadata    │    │   CDN        │
│   /processed │    │  DB (MySQL)  │    │  (Edge       │
│              │    │  + Redis     │    │   transform  │
│   WebP,      │    │              │    │   on miss)   │
│   thumbnails │    │              │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
                                               │
                                        ┌──────▼───────┐
                                        │ On-Demand    │
                                        │ Transform    │
                                        │ Service      │
                                        │ (imgproxy /  │
                                        │  Thumbor)    │
                                        └──────────────┘
```

### Component Deep Dive

#### Pre-Processing vs On-Demand Transforms — Hybrid Architecture

```
Strategy 1: Pre-process all variants on upload
  Upload → generate 5 sizes × 2 formats = 10 files per image
  ✓ Instant serving (no compute at request time)
  ✗ Storage waste: 10× per image; many variants never requested
  ✗ Adding a new variant = reprocess ALL existing images
  Use case: Known, fixed set of variants (e.g., Instagram: square thumb, feed size, story size)

Strategy 2: On-demand processing (transform at CDN edge / origin)
  Upload → store original only
  Request: GET /image/abc.jpg?w=720&h=480&format=webp
  Transform service processes on first request → cache result on CDN
  ✓ No wasted storage (only process what's actually requested)
  ✓ Infinite flexibility (any size, crop, format)
  ✗ First request is slow (200-500 ms to process)
  ✗ CDN cache cold start for new images
  Use case: E-commerce catalogs (thousands of image sizes), user-generated content

Strategy 3: Hybrid ⭐ (recommended for social media scale):
  On upload: generate CORE variants (thumbnail 150×150, feed 1080×1080, profile 640×640)
    These cover 90% of requests → pre-compute for instant serving
  
  On demand: any other variant requested
    Unusual sizes, formats, crops → processed on first request → cached
    Covers the long tail of 10% requests without pre-computing everything

  Cost comparison (100M images/day):
    Pre-process all: 100M × 10 variants = 1B images/day → massive compute + storage
    Hybrid: 100M × 3 core variants + ~5M on-demand = 305M images/day → 70% less compute
```

#### Image Format Pipeline — Why WebP and AVIF

```
Format comparison (same image, same perceptual quality):

Format   Quality   File Size   Browser Support        Encode Speed
JPEG     Baseline  100 KB      100% (universal)       Very fast
PNG      Lossless  300 KB      100%                   Fast
WebP     Equal     70 KB       96% (all modern)       Fast
AVIF     Equal     50 KB       85% (growing)          Slow (10× WebP)
JPEG XL  Equal     55 KB       5% (experimental)      Medium

Strategy: serve the best format the client supports

  Client: Accept: image/avif, image/webp, image/jpeg
  
  Server/CDN decision:
    If client supports AVIF → serve AVIF (50% smaller than JPEG)
    Elif supports WebP → serve WebP (30% smaller than JPEG)
    Else → serve JPEG (universal fallback)

Content negotiation at CDN:
  CDN caches 3 versions: image_abc.avif, image_abc.webp, image_abc.jpg
  Vary: Accept header → CDN serves correct format per client
  
  Alternative: URL-based format selection
    /images/abc.jpg?format=auto → server detects Accept header → returns best format
    CDN cache key includes format → no Vary header needed

Quality optimization:
  Not all images need the same JPEG quality!
  
  Complex image (detailed photo): quality 85 → SSIM 0.97
  Simple image (text, UI screenshot): quality 70 → SSIM 0.99
  
  Adaptive quality: analyze image complexity → set quality accordingly
  Facebook/Instagram does this → saves 10-15% bandwidth at scale
```

#### Smart Cropping — Face-Aware and Content-Aware

```
Problem: Resize 4000×3000 landscape photo to 500×500 square thumbnail
  Center crop: might cut off faces, important subjects
  Letterbox: adds black bars (ugly)

Smart crop approaches:

1. Face detection crop:
   Run face detection (OpenCV Haar cascades, or MTCNN)
   If faces found → crop with faces centered
   If no faces → fall back to saliency crop
   
   face_locations = face_detector.detect(image)
   if face_locations:
       center = average_center(face_locations)
       crop_region = crop_around(center, target_size)
   else:
       crop_region = saliency_crop(image, target_size)

2. Saliency/attention crop:
   ML model predicts "attention heatmap" (where humans look)
   Crop region centered on highest-attention area
   Libraries: smartcrop.js, thumbor smart crop
   
3. Rule of thirds:
   Divide image into 3×3 grid
   Crop to include the most "interesting" intersection point
   Simple, fast, usually looks good

4. Edge detection:
   Find image region with most edges (detail)
   Avoid cropping through important edges (faces, objects)

Implementation choice for scale:
  - Face detection: ~50 ms per image (with GPU acceleration)
  - Saliency model: ~100 ms per image (ML inference)
  - Rule of thirds: ~1 ms per image (simple math)
  
  For 50K images/sec: Use lightweight face detection → saliency fallback
  Pre-compute: run on upload (batch, can take 100 ms)
  Store: crop coordinates per variant size in metadata DB
  Apply: fast crop at serving time using stored coordinates
```

#### Content Moderation — ML Safety Pipeline

```
Every uploaded image passes through moderation:

Step 1: NSFW Detection
  Model: ResNet-50 or EfficientNet trained on NSFW dataset
  Output: P(nsfw), P(suggestive), P(safe)
  Threshold: P(nsfw) > 0.8 → auto-reject
              P(nsfw) > 0.5 → queue for human review
              P(nsfw) < 0.5 → auto-approve

Step 2: Violence/Gore Detection
  Similar ML model for violent content
  
Step 3: Text Detection (OCR + spam filter)
  Extract text from image using Tesseract/PaddleOCR
  Check against spam/abuse word list
  Detect: phone numbers, URLs, promotional text in profile photos

Step 4: Face Detection (for tagging, not moderation)
  Detect face locations + face embeddings
  Used for: photo tagging, face clustering

Inference at scale (50K images/sec):
  Each image needs ~4 ML inferences → 200K inferences/sec
  GPU inference: ~5 ms per image → each GPU handles 200 images/sec
  200K / 200 = 1,000 GPUs needed (NVIDIA T4 or A10G)
  
  Optimization:
  1. Batch inference: process 32 images per GPU batch → 8× throughput
     1,000 GPUs → 125 GPUs needed
  2. Model distillation: smaller model (MobileNet) for initial screening
     Only run heavy model on borderline cases (top 10%)
     125 GPUs → ~30 GPUs for initial + ~20 for secondary = 50 GPUs total
  3. TensorRT / ONNX optimization: 2× speedup on same hardware
     50 GPUs → 25 GPUs

  Cost: 25 × NVIDIA T4 × $0.50/hr = $300/hr = $7,200/day
  Processing: 100M images/day for $7,200 = $0.000072 per image
```

---

## 5. APIs

### Upload Image
```http
POST /api/v1/images/upload
{
  "filename": "vacation.jpg",
  "content_type": "image/jpeg"
}
Response: 200 OK
{
  "image_id": "img-uuid",
  "upload_url": "https://s3.amazonaws.com/originals/img-uuid?X-Amz-..."
}
```

### Get Processed Image (with transformations)
```http
GET /api/v1/images/{image_id}?w=720&h=480&format=webp&crop=smart&quality=80
Response: 302 Redirect → CDN URL
  Location: https://cdn.example.com/images/img-uuid/720x480_smart_q80.webp

Or direct CDN URL pattern:
  https://cdn.example.com/img-uuid/w_720,h_480,f_webp,c_smart,q_80
```

### Get Image Metadata
```http
GET /api/v1/images/{image_id}/metadata
Response: 200 OK
{
  "image_id": "img-uuid",
  "original": {
    "width": 4000, "height": 3000, "format": "jpeg",
    "size_bytes": 3145728, "color_space": "sRGB"
  },
  "exif": { "camera": "iPhone 15 Pro", "date": "2025-03-14", "iso": 100 },
  "analysis": {
    "dominant_colors": ["#2E86C1", "#F4D03F", "#2ECC71"],
    "faces_detected": 2,
    "nsfw_score": 0.01,
    "quality_score": 0.89
  },
  "variants": {
    "thumbnail": "https://cdn.example.com/img-uuid/150x150.webp",
    "small": "https://cdn.example.com/img-uuid/320x320.webp",
    "medium": "https://cdn.example.com/img-uuid/640x640.webp",
    "large": "https://cdn.example.com/img-uuid/1080x1080.webp"
  },
  "processing_status": "completed"
}
```

### Batch Process
```http
POST /api/v1/images/batch
{
  "image_ids": ["img-1", "img-2", ...],
  "transforms": {
    "sizes": [{"name": "card", "width": 400, "height": 300, "crop": "smart"}],
    "format": "webp",
    "quality": 85,
    "watermark": {"text": "© 2025", "position": "bottom-right", "opacity": 0.3}
  }
}
Response: 202 Accepted
{ "batch_id": "batch-uuid", "total": 500, "status": "processing" }
```

---

## 6. Data Model

### MySQL — Image Metadata

```sql
CREATE TABLE images (
    image_id        VARCHAR(36) PRIMARY KEY,
    user_id         VARCHAR(36) NOT NULL,
    original_s3_key TEXT NOT NULL,
    original_format VARCHAR(10),
    original_width  INT,
    original_height INT,
    file_size_bytes INT,
    color_space     VARCHAR(10),
    orientation     SMALLINT,
    exif_data       JSON,
    dominant_colors JSON,              -- ["#2E86C1", "#F4D03F"]
    faces_detected  SMALLINT DEFAULT 0,
    face_regions    JSON,              -- [{x, y, width, height}, ...]
    nsfw_score      DECIMAL(4,3),
    quality_score   DECIMAL(4,3),
    moderation_status ENUM('pending','approved','rejected','review') DEFAULT 'pending',
    processing_status ENUM('uploaded','processing','completed','failed') DEFAULT 'uploaded',
    smart_crop_data JSON,              -- pre-computed crop regions per target size
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user (user_id, created_at DESC),
    INDEX idx_moderation (moderation_status),
    INDEX idx_processing (processing_status)
);

CREATE TABLE image_variants (
    variant_id      VARCHAR(36) PRIMARY KEY,
    image_id        VARCHAR(36) NOT NULL,
    variant_name    VARCHAR(50),        -- "thumbnail", "medium", "720x480_webp"
    s3_key          TEXT NOT NULL,
    format          VARCHAR(10),
    width           INT,
    height          INT,
    file_size_bytes INT,
    quality         SMALLINT,
    crop_type       VARCHAR(20),        -- "smart", "center", "face"
    created_at      TIMESTAMP,
    INDEX idx_image (image_id),
    UNIQUE INDEX idx_image_variant (image_id, variant_name)
);
```

### S3 Storage Layout

```
Bucket: image-originals (cross-region replicated, never deleted)
  /{image_id}/original.jpg

Bucket: image-processed (lifecycle: delete after 1 year if no access)
  /{image_id}/thumbnail_150x150.webp
  /{image_id}/small_320x320.webp
  /{image_id}/medium_640x640.webp
  /{image_id}/large_1080x1080.webp
  /{image_id}/custom/720x480_smart_q80.webp    (on-demand generated)
```

### Redis — Transform Cache + Processing State

```
# Processing pipeline state
img_status:{image_id}    → Hash { status, step, started_at }
TTL: 3600

# Transform result cache (for on-demand transforms)
transform:{image_id}:{params_hash}  → S3 key of processed variant
TTL: 86400

# Popular image metadata cache
img_meta:{image_id}      → Hash { width, height, format, cdn_base_url, smart_crop }
TTL: 3600

# Rate limiting per user
upload_rate:{user_id}    → INT (INCR, check < 100/hour)
TTL: 3600
```

### Kafka Topics

```
Topic: image-uploaded        (trigger processing pipeline)
Topic: image-processed       (notify downstream: feed service, search indexing)
Topic: moderation-results    (ML moderation output → human review queue if needed)
Topic: batch-tasks           (batch processing job distribution)
```

---

## 7. Fault Tolerance

| Concern | Solution |
|---|---|
| **Original image lost** | S3 cross-region replication; 11 nines durability |
| **Processing worker crash** | Retry from Kafka/SQS; idempotent processing |
| **Corrupt image upload** | Validate image header + decode test before processing |
| **ML moderation error** | Human review queue for borderline scores; false positive rate < 1% |
| **CDN cache miss storm** | On-demand transform with singleflight; origin shield |
| **Format unsupported** | ImageMagick/libvips handle 100+ formats; unknown format → reject with error |
| **Excessively large image** | Limit: 50 MB max, 30K × 30K max pixels; reject larger |
| **Transform service overload** | Auto-scale based on queue depth; circuit breaker at 500ms latency |

### Specific: Image Bomb / Decompression Bomb

```
Attack: Upload a 50 KB PNG that decompresses to 100 GB (billions of pixels)
  "zip bomb" equivalent for images

Detection and prevention:
  1. Before full decode: read image header → get dimensions
     If width × height × 4 (RGBA bytes) > 1 GB → reject immediately
     Header read: O(1), no decompression needed
  
  2. Set resource limits on processing:
     ulimit -v 2048000  (2 GB virtual memory limit)
     If processing exceeds limit → killed by OS → job marked failed
  
  3. Use libvips instead of ImageMagick:
     libvips: streaming, doesn't load full image into memory
     ImageMagick: loads full image → vulnerable to memory exhaustion
     libvips handles 100K × 100K images in ~50 MB of RAM
  
  4. Timeout: if processing takes > 30 seconds → kill and reject
```

### Specific: EXIF GPS Stripping — Privacy Concern

```
Problem: User uploads photo from phone. EXIF contains exact GPS coordinates.
  If served with EXIF → anyone can find out where user was.
  
Solution: Strip sensitive EXIF data BEFORE storing processed variants
  
  Keep: camera model, lens, ISO, aperture (useful for photography)
  Strip: GPS coordinates, timestamps, serial numbers, thumbnails
  
  Implementation:
    # Using Pillow (Python)
    from PIL import Image
    img = Image.open("photo.jpg")
    exif = img.getexif()
    SENSITIVE_TAGS = [0x8825]  # GPSInfo tag
    for tag in SENSITIVE_TAGS:
        if tag in exif:
            del exif[tag]
    img.save("photo_clean.jpg", exif=exif.tobytes())
  
  Original: keep full EXIF in original bucket (private, owner access only)
  Processed: stripped EXIF in all public variants
  
  The original with GPS is valuable for the owner (photo management, maps)
  But must NEVER be served publicly
```

---

## 8. Deep Dive: Engineering Trade-offs

### ImageMagick vs libvips vs Pillow: Processing Library Choice

```
ImageMagick:
  ✓ Supports 200+ formats
  ✓ Extremely versatile (any transform imaginable)
  ✗ Memory hog: loads entire image into RAM (decode → full bitmap → encode)
  ✗ Security: historically many CVEs (ImageTragick RCE in 2016)
  ✗ Slow for simple operations (resize, convert)
  Use case: Batch processing, rare format support

libvips ⭐ (recommended for production):
  ✓ Streaming architecture: processes image in tiles, not all at once
  ✓ 10× faster than ImageMagick for resize/convert
  ✓ 10× less memory (processes 100K×100K in 50 MB)
  ✓ Thread-safe (can parallelize within process)
  ✓ Excellent WebP, AVIF, HEIF support
  ✗ Fewer exotic format support than ImageMagick
  Use case: High-throughput image processing (web services, CDN transforms)

Pillow (Python):
  ✓ Simple Python API
  ✓ Good for prototyping
  ✗ Single-threaded (GIL)
  ✗ Slower than libvips
  Use case: Scripts, ML pipeline integration, prototyping

Sharp (Node.js, based on libvips):
  ✓ Node.js native bindings to libvips
  ✓ Async processing (fits Node event loop)
  ✓ Very popular for web backends
  Use case: Node.js backends, serverless functions

Performance comparison (resize 4000×3000 → 800×600 JPEG → WebP):
  ImageMagick:  450 ms, 180 MB RAM
  Pillow:       200 ms, 120 MB RAM
  libvips:       35 ms,  20 MB RAM ← 6× faster, 9× less memory
  Sharp:         40 ms,  25 MB RAM
```

### On-Demand Transforms: imgproxy vs Thumbor vs Cloudinary

```
Self-hosted options for on-demand image transformation:

imgproxy ⭐:
  Written in Go, uses libvips
  URL: /img/w:720/h:480/f:webp/{s3_url}
  Signed URLs (HMAC) prevent abuse (can't request arbitrary transforms)
  Performance: 1000+ transforms/sec per instance
  Memory: 100-300 MB per instance
  
  URL signing:
    signature = HMAC-SHA256(secret, "/w:720/h:480/f:webp/s3://bucket/img.jpg")
    URL: /img/{signature}/w:720/h:480/f:webp/...
    Prevents: attackers requesting 50,000 × 50,000 resize (DoS)

Thumbor (Python):
  Open source, used by many companies
  Smart crop with face/feature detection
  Plugin system for custom transforms
  Slower than imgproxy (Python vs Go)

Cloudinary / Imgix (SaaS):
  ✓ Zero maintenance, global CDN included
  ✓ Advanced features (AI crop, background removal, text overlay)
  ✗ Cost: $0.01-0.05 per 1000 transforms + storage
  ✗ Vendor lock-in
  At 100M images/day: $1M-5M/year (expensive at scale)
  
Decision tree:
  < 1M images/day → Cloudinary (simpler, acceptable cost)
  1M-100M images/day → imgproxy + CDN (cost-effective, fast)
  > 100M images/day → custom pipeline + imgproxy for long tail
```

### Responsive Image Serving: srcset and Client Hints

```
Modern web: don't serve 4K image to a 320px phone screen!

HTML srcset:
  <img srcset="img_300w.webp 300w,
               img_600w.webp 600w,
               img_1200w.webp 1200w"
       sizes="(max-width: 600px) 300px, (max-width: 1200px) 600px, 1200px"
       src="img_600w.webp" />
  
  Browser selects optimal size based on viewport + device pixel ratio

Client Hints (HTTP header):
  Client sends: DPR: 2, Viewport-Width: 375, Width: 320
  Server/CDN uses hints to select optimal variant automatically
  
  Advantage: server decides (can optimize aggressively)
  Disadvantage: limited browser support (Chrome yes, Safari no)

For our pipeline:
  Pre-generate: 300w, 600w, 900w, 1200w, 2000w (5 widths)
  × 3 formats: JPEG, WebP, AVIF = 15 variants per image
  
  Or: generate on-demand with CDN caching (only requested sizes get generated)
  
  URL convention:
    /images/{id}/w_{width}.{format}
    /images/abc123/w_600.webp → 600px wide WebP
    CDN caches each unique URL → subsequent requests are instant
```

### Storage Cost Optimization at Scale

```
100M images/day × 365 days × 3 MB avg = 109.5 PB/year (originals alone)

Cost at S3 Standard: $0.023/GB/month
  109.5 PB × $0.023/GB = $2.52M/month just for originals!

Optimization strategies:

1. Format conversion: JPEG → WebP saves 30%
   3 MB JPEG → 2.1 MB WebP → saves 32 PB/year → $735K/year

2. Quality optimization: perceptual quality targeting
   Instead of JPEG quality 95, use quality 85 (visually identical)
   Saves 40% file size → $1M/year

3. Storage tiering:
   Hot (< 30 days): S3 Standard → $0.023/GB
   Warm (30-180 days): S3 IA → $0.0125/GB (45% cheaper)
   Cold (180+ days): S3 Glacier IR → $0.004/GB (83% cheaper)
   
   Most images are only accessed in the first 7 days
   Tiering saves ~60% of storage costs

4. Deduplication:
   Hash original images → detect duplicate uploads
   Store once, reference multiple times
   Typical dedup rate: 10-15% on social platforms
   
   perceptual_hash = compute_phash(image)
   if EXISTS in hash_index → link to existing image, don't store again

5. Progressive deletion:
   Delete processed variants (can be regenerated) after 1 year
   Keep only originals in cold storage
   If user requests old variant → regenerate from original on-demand
```



