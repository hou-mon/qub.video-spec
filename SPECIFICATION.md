# QUB Container Specification v1.0

## Information Cubed — AI-Forward Video Container

**Status:** Draft Specification  
**Version:** 1.0.0  
**Date:** January 2026  
**Author:** Houman Shekarchi  
**License:** Apache License 2.0

---

## 1. Executive Summary

QUB (Information Cubed) is a next-generation container format designed to store not just media streams, but the complete semantic understanding of video content. By embedding multimodal AI preprocessing results directly alongside traditional audio/video codecs, QUB eliminates redundant inference during editing workflows and enables a new class of intelligent, query-driven Non-Linear Editors.

### Design Principles

1. **Inference Once, Query Forever** — Expensive AI preprocessing (vision, audio, segmentation) happens at ingest; results persist with the media
2. **Codec Agnostic** — Wrap any modern video/audio codec without modification
3. **Model Versioned** — All AI-derived data carries full provenance (model ID, version, parameters, confidence)
4. **Temporally Dense** — Sub-frame precision for all metadata; microsecond-accurate alignment
5. **Query Native** — Built-in semantic index enables natural language queries against content
6. **Graceful Degradation** — Standard players can decode media tracks; AI tracks are optional enhancement
7. **Extensible by Design** — New AI modalities slot in without breaking existing parsers

---

## 2. Container Architecture

### 2.1 High-Level Structure

```
┌─────────────────────────────────────────────────────────────┐
│                      QUB CONTAINER                          │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   FILE HEADER                        │   │
│  │  Magic Number | Version | Flags | UUID | Created    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  TRACK DIRECTORY                     │   │
│  │  Track Count | Track Descriptors[] | Index Offsets  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               MEDIA TRACKS (Traditional)             │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│  │  │  Video   │  │  Audio   │  │  Audio   │  ...     │   │
│  │  │  H.265   │  │  AAC L   │  │  AAC R   │          │   │
│  │  └──────────┘  └──────────┘  └──────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  AI TRACKS (Novel)                   │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│  │  │  Vision  │  │ Temporal │  │  Audio   │  ...     │   │
│  │  │ Semantic │  │ Analysis │  │ Analysis │          │   │
│  │  └──────────┘  └──────────┘  └──────────┘          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│  │  │  Depth   │  │ Instance │  │  Alpha   │  ...     │   │
│  │  │   Maps   │  │   Seg    │  │  Mattes  │          │   │
│  │  └──────────┘  └──────────┘  └──────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  SEMANTIC INDEX                      │   │
│  │  Entity Index | Concept Index | Temporal Index      │   │
│  │  Embedding Store | Query Interface Metadata         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 PROVENANCE LEDGER                    │   │
│  │  Model Registry | Processing History | Checksums    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Chunk-Based Storage

QUB uses a chunk-based architecture similar to RIFF/IFF, enabling:
- Streaming playback with interleaved AI data
- Selective loading of specific track types
- Efficient seeking without loading entire file

```
Chunk Structure:
┌────────────┬────────────┬────────────┬─────────────────────┐
│  FourCC    │   Size     │   Flags    │      Payload        │
│  4 bytes   │  8 bytes   │  4 bytes   │    Size bytes       │
└────────────┴────────────┴────────────┴─────────────────────┘
```

**Reserved FourCC Codes:**

| FourCC | Description |
|--------|-------------|
| `QUB\0` | File identifier |
| `TDIR` | Track directory |
| `MTRK` | Media track data |
| `ATRK` | AI track data |
| `SIDX` | Semantic index |
| `PROV` | Provenance ledger |
| `EMBD` | Embedding store |
| `EXTN` | Extension chunk |

---

## 3. File Header

### 3.1 Header Structure

```c
struct QUBHeader {
    uint8_t  magic[4];           // "QUB\x00"
    uint16_t version_major;      // 1
    uint16_t version_minor;      // 0
    uint32_t flags;              // See Flag Definitions
    uint8_t  uuid[16];           // Unique file identifier
    int64_t  created_timestamp;  // Unix timestamp (microseconds)
    int64_t  modified_timestamp; // Unix timestamp (microseconds)
    uint64_t duration_us;        // Total duration in microseconds
    uint32_t track_count;        // Number of tracks
    uint64_t track_dir_offset;   // Offset to track directory
    uint64_t semantic_idx_offset;// Offset to semantic index
    uint64_t provenance_offset;  // Offset to provenance ledger
    uint8_t  reserved[64];       // Future expansion
};
```

### 3.2 Flag Definitions

```c
enum QUBFlags {
    QUB_FLAG_STREAMABLE      = 0x0001,  // Optimized for streaming
    QUB_FLAG_ENCRYPTED       = 0x0002,  // AI tracks encrypted
    QUB_FLAG_FRAGMENTED      = 0x0004,  // Multi-file container
    QUB_FLAG_COMPLETE_AI     = 0x0008,  // All AI preprocessing done
    QUB_FLAG_PARTIAL_AI      = 0x0010,  // Some AI tracks pending
    QUB_FLAG_EMBEDDINGS      = 0x0020,  // Contains embedding store
    QUB_FLAG_REALTIME        = 0x0040,  // Processed for realtime
};
```

---

## 4. Track System

### 4.1 Track Types

QUB defines two primary track categories:

**Media Tracks (Type 0x00-0x3F):**
- `0x01` — Primary Video
- `0x02` — Secondary Video (multicam, proxy)
- `0x10` — Primary Audio
- `0x11` — Secondary Audio
- `0x20` — Timecode
- `0x21` — Closed Captions (source)

**AI Tracks (Type 0x40-0xFF):**
- `0x40` — Vision Semantic
- `0x41` — Vision Object Detection
- `0x42` — Vision Face Detection
- `0x43` — Vision OCR
- `0x44` — Vision Action Recognition
- `0x50` — Temporal Shot Boundaries
- `0x51` — Temporal Scene Changes
- `0x52` — Temporal Motion Analysis
- `0x53` — Temporal Camera Motion
- `0x60` — Audio Transcription
- `0x61` — Audio Speaker Diarization
- `0x62` — Audio Classification
- `0x63` — Audio Music Analysis
- `0x64` — Audio Emotion/Sentiment
- `0x70` — Segmentation Depth Map
- `0x71` — Segmentation Semantic
- `0x72` — Segmentation Instance
- `0x73` — Segmentation Panoptic
- `0x74` — Segmentation Alpha Matte
- `0x75` — Segmentation Saliency
- `0x80` — Embedding Dense (per-frame)
- `0x81` — Embedding Sparse (keyframes)
- `0x82` — Embedding Audio
- `0x90` — Custom/Extension

### 4.2 Track Descriptor

```c
struct TrackDescriptor {
    uint32_t track_id;           // Unique within file
    uint8_t  track_type;         // From Track Types enum
    uint8_t  track_subtype;      // Type-specific subtype
    uint16_t flags;              // Track flags
    uint8_t  codec_fourcc[4];    // Codec identifier
    uint8_t  language[3];        // ISO 639-2 language code
    uint8_t  reserved;
    uint64_t data_offset;        // Offset to track data
    uint64_t data_size;          // Total track data size
    uint64_t index_offset;       // Offset to track index
    uint32_t sample_count;       // Number of samples/frames
    uint64_t duration_us;        // Track duration
    uint32_t model_ref;          // Index into provenance ledger
    char     name[64];           // Human-readable name
    uint8_t  metadata[128];      // Track-specific metadata
};
```

### 4.3 Track Flags

```c
enum TrackFlags {
    TRACK_ENABLED         = 0x0001,
    TRACK_DEFAULT         = 0x0002,
    TRACK_FORCED          = 0x0004,
    TRACK_LOSSLESS        = 0x0008,
    TRACK_COMPRESSED      = 0x0010,
    TRACK_DELTA_ENCODED   = 0x0020,  // Only changes stored
    TRACK_SPARSE          = 0x0040,  // Non-continuous samples
    TRACK_DERIVED         = 0x0080,  // Derived from other tracks
};
```

---

## 5. Media Track Specifications

### 5.1 Supported Video Codecs

| Codec | FourCC | Notes |
|-------|--------|-------|
| H.264/AVC | `avc1` | Baseline, Main, High profiles |
| H.265/HEVC | `hvc1` | Main, Main10, Main12 profiles |
| H.266/VVC | `vvc1` | Future support |
| AV1 | `av01` | Main, High profiles |
| VP9 | `vp09` | Profile 0, 2 |
| ProRes | `apch` | All ProRes variants |
| DNxHD/HR | `AVdn` | All DNx variants |
| JPEG 2000 | `mjp2` | DCI compliance |
| RAW | `raw ` | Uncompressed, various pixel formats |

### 5.2 Supported Audio Codecs

| Codec | FourCC | Notes |
|-------|--------|-------|
| AAC | `mp4a` | LC, HE-AAC, HE-AACv2 |
| FLAC | `fLaC` | Lossless |
| Opus | `Opus` | Low latency |
| PCM | `lpcm` | 16, 24, 32-bit; 44.1-192kHz |
| Dolby AC-3 | `ac-3` | Surround |
| Dolby E-AC-3 | `ec-3` | Atmos-ready |
| DTS | `dtsc` | Core, HD, HD-MA |

### 5.3 Media Sample Structure

```c
struct MediaSample {
    int64_t  pts;                // Presentation timestamp (us)
    int64_t  dts;                // Decode timestamp (us)
    uint32_t duration;           // Sample duration (us)
    uint32_t size;               // Payload size
    uint32_t flags;              // Keyframe, etc.
    uint64_t offset;             // Offset in track data
};
```

---

## 6. AI Track Specifications

### 6.1 Vision Semantic Track (0x40)

Stores dense scene descriptions from vision-language models.

```c
struct VisionSemanticSample {
    int64_t  start_pts;          // Start timestamp
    int64_t  end_pts;            // End timestamp (scene boundary)
    float    confidence;         // Model confidence [0,1]
    uint32_t description_len;    // UTF-8 description length
    char     description[];      // Natural language description
    uint32_t tag_count;          // Number of semantic tags
    SemanticTag tags[];          // Structured tags
};

struct SemanticTag {
    uint32_t category;           // Tag category ID
    uint32_t value_id;           // Value within category
    float    confidence;
    BoundingBox bbox;            // Optional spatial localization
};
```

**Recommended Processing Pipeline:**
- FastVLM, LLaVA-NeXT, or Qwen-VL for descriptions
- Run at 1 FPS minimum, scene boundaries, or shot changes
- Store at multiple detail levels (brief, standard, detailed)

### 6.2 Vision Object Detection Track (0x41)

Frame-by-frame object detection with tracking IDs.

```c
struct ObjectDetectionFrame {
    int64_t  pts;
    uint32_t object_count;
    DetectedObject objects[];
};

struct DetectedObject {
    uint32_t track_id;           // Persistent across frames
    uint32_t class_id;           // COCO/custom class
    char     class_name[32];     // Human-readable
    float    confidence;
    BoundingBox bbox;            // Normalized [0,1]
    uint32_t attribute_count;
    ObjectAttribute attrs[];     // Color, pose, etc.
};

struct BoundingBox {
    float x, y;                  // Top-left corner
    float width, height;         // Normalized dimensions
};
```

**Recommended Processing Pipeline:**
- YOLO v8/v9, DETR, or Grounding DINO
- Process every frame or keyframes with interpolation
- Maintain tracking IDs via ByteTrack/BoT-SORT

### 6.3 Vision Face Detection Track (0x42)

Specialized face detection with identity clustering.

```c
struct FaceDetectionFrame {
    int64_t  pts;
    uint32_t face_count;
    DetectedFace faces[];
};

struct DetectedFace {
    uint32_t person_id;          // Clustered identity
    BoundingBox bbox;
    float    landmarks[68][2];   // 68-point landmarks
    float    pose[3];            // Yaw, pitch, roll
    float    quality;            // Face quality score
    float    embedding[512];     // Optional identity embedding
    uint8_t  attributes;         // Flags: glasses, beard, etc.
    char     assigned_name[64];  // User-assigned name
};
```

**Privacy Note:** Face embeddings should be stored encrypted by default. The `assigned_name` field is populated by user, never by AI.

### 6.4 Vision OCR Track (0x43)

On-screen text detection and recognition.

```c
struct OCRFrame {
    int64_t  pts;
    uint32_t text_region_count;
    TextRegion regions[];
};

struct TextRegion {
    float    polygon[8];         // 4-point quadrilateral
    char     text[1024];         // Recognized text (UTF-8)
    char     language[3];        // Detected language
    float    confidence;
    uint8_t  text_type;          // 0=scene, 1=graphic, 2=caption
};
```

### 6.5 Temporal Shot Boundaries Track (0x50)

Shot detection and classification.

```c
struct ShotBoundary {
    int64_t  timestamp;          // Boundary timestamp
    uint8_t  transition_type;    // See TransitionType enum
    float    confidence;
    uint32_t duration;           // Transition duration (us)
    uint32_t shot_before_id;     // Shot ending
    uint32_t shot_after_id;      // Shot beginning
};

enum TransitionType {
    TRANS_CUT           = 0,
    TRANS_DISSOLVE      = 1,
    TRANS_FADE_OUT      = 2,
    TRANS_FADE_IN       = 3,
    TRANS_WIPE          = 4,
    TRANS_PUSH          = 5,
    TRANS_SLIDE         = 6,
    TRANS_FLASH         = 7,
    TRANS_OTHER         = 255,
};

struct ShotDescriptor {
    uint32_t shot_id;
    int64_t  start_pts;
    int64_t  end_pts;
    uint8_t  shot_type;          // See ShotType enum
    uint8_t  camera_motion;      // See CameraMotion enum
    float    avg_brightness;
    float    motion_intensity;
    char     description[256];   // Brief shot description
};

enum ShotType {
    SHOT_UNKNOWN        = 0,
    SHOT_EXTREME_WIDE   = 1,
    SHOT_WIDE           = 2,
    SHOT_FULL           = 3,
    SHOT_MEDIUM_WIDE    = 4,
    SHOT_MEDIUM         = 5,
    SHOT_MEDIUM_CLOSE   = 6,
    SHOT_CLOSE          = 7,
    SHOT_EXTREME_CLOSE  = 8,
    SHOT_INSERT         = 9,
    SHOT_POV            = 10,
    SHOT_OVER_SHOULDER  = 11,
    SHOT_TWO_SHOT       = 12,
    SHOT_GROUP          = 13,
};

enum CameraMotion {
    CAM_STATIC          = 0,
    CAM_PAN_LEFT        = 1,
    CAM_PAN_RIGHT       = 2,
    CAM_TILT_UP         = 3,
    CAM_TILT_DOWN       = 4,
    CAM_ZOOM_IN         = 5,
    CAM_ZOOM_OUT        = 6,
    CAM_DOLLY_IN        = 7,
    CAM_DOLLY_OUT       = 8,
    CAM_TRACK_LEFT      = 9,
    CAM_TRACK_RIGHT     = 10,
    CAM_CRANE_UP        = 11,
    CAM_CRANE_DOWN      = 12,
    CAM_HANDHELD        = 13,
    CAM_STEADICAM       = 14,
    CAM_DRONE           = 15,
};
```

### 6.6 Audio Transcription Track (0x60)

Speech-to-text with word-level alignment.

```c
struct TranscriptionSegment {
    int64_t  start_pts;
    int64_t  end_pts;
    uint32_t speaker_id;         // From diarization
    char     language[3];        // ISO 639-2
    float    confidence;
    char     text[4096];         // Segment text
    uint32_t word_count;
    WordAlignment words[];
};

struct WordAlignment {
    int64_t  start_pts;
    int64_t  end_pts;
    float    confidence;
    uint16_t text_offset;        // Offset into segment text
    uint16_t text_length;        // Word length in bytes
    float    probability;        // Token probability
};
```

**Recommended Processing Pipeline:**
- Whisper Large v3 or faster variants
- Always include word-level timestamps
- Store alternative transcriptions for low-confidence segments

### 6.7 Audio Speaker Diarization Track (0x61)

Speaker segmentation and clustering.

```c
struct DiarizationSegment {
    int64_t  start_pts;
    int64_t  end_pts;
    uint32_t speaker_id;         // Clustered speaker
    float    confidence;
    float    embedding[256];     // Speaker embedding
};

struct SpeakerProfile {
    uint32_t speaker_id;
    char     assigned_name[64];  // User-assigned
    float    reference_embedding[256];
    uint32_t total_speech_us;    // Total speaking time
    float    avg_pitch;
    uint8_t  gender_estimate;    // 0=unknown, 1=male, 2=female
};
```

### 6.8 Audio Classification Track (0x62)

Non-speech audio event detection.

```c
struct AudioClassificationFrame {
    int64_t  start_pts;
    int64_t  end_pts;
    uint32_t event_count;
    AudioEvent events[];
};

struct AudioEvent {
    uint32_t class_id;           // AudioSet or custom ontology
    char     class_name[64];
    float    confidence;
    float    loudness_db;
    uint8_t  channel_mask;       // Which channels contain event
};
```

**Standard Categories:**
- Speech, Music, Ambient, Effects
- Sub-categories: Dialogue, Narration, Crowd, Traffic, Nature, Foley, etc.

### 6.9 Segmentation Depth Map Track (0x70)

Monocular depth estimation.

```c
struct DepthMapFrame {
    int64_t  pts;
    uint16_t width, height;      // May differ from source
    uint8_t  precision;          // Bits per pixel
    uint8_t  encoding;           // 0=raw, 1=png, 2=exr
    float    min_depth;          // Depth range
    float    max_depth;
    uint32_t data_size;
    uint8_t  data[];             // Depth map payload
};
```

**Recommended Processing Pipeline:**
- Depth Anything v2, MiDaS, or ZoeDepth
- Store relative depth; absolute depth if available
- Compress with PNG for reasonable quality/size

### 6.10 Segmentation Instance Track (0x72)

Per-object instance segmentation masks.

```c
struct InstanceSegFrame {
    int64_t  pts;
    uint16_t width, height;
    uint32_t instance_count;
    InstanceMask instances[];
};

struct InstanceMask {
    uint32_t object_track_id;    // Links to Object Detection
    uint32_t class_id;
    BoundingBox bbox;
    uint8_t  encoding;           // 0=RLE, 1=polygon, 2=bitmap
    uint32_t mask_size;
    uint8_t  mask_data[];        // Encoded mask
};
```

**RLE Encoding:**
Run-length encoding with (count, value) pairs for efficient storage.

### 6.11 Segmentation Alpha Matte Track (0x74)

High-quality foreground/background separation.

```c
struct AlphaMatteFrame {
    int64_t  pts;
    uint16_t width, height;
    uint8_t  precision;          // 8 or 16 bit
    uint8_t  encoding;           // 0=raw, 1=png
    uint32_t subject_id;         // Which subject this mattes
    float    trimap_threshold;   // Threshold used
    uint32_t data_size;
    uint8_t  data[];             // Alpha channel
};
```

**Recommended Processing Pipeline:**
- SAM 2, RMBG, or specialized matting models
- Store at source resolution for quality
- Include trimap parameters for refinement

---

## 7. Semantic Index

The semantic index enables natural language queries against video content.

### 7.1 Index Structure

```c
struct SemanticIndex {
    uint32_t version;
    uint64_t entity_index_offset;
    uint64_t concept_index_offset;
    uint64_t temporal_index_offset;
    uint64_t embedding_store_offset;
    uint32_t embedding_dimension;
    uint32_t entity_count;
    uint32_t concept_count;
};
```

### 7.2 Entity Index

Maps detected entities to temporal occurrences.

```c
struct EntityEntry {
    uint32_t entity_id;
    uint8_t  entity_type;        // Person, object, place, etc.
    char     canonical_name[64];
    uint32_t alias_count;
    uint32_t occurrence_count;
    uint64_t occurrences_offset;
    float    representative_embedding[512];
};

struct EntityOccurrence {
    int64_t  start_pts;
    int64_t  end_pts;
    uint32_t track_id;           // Source AI track
    float    confidence;
    BoundingBox bbox;            // If applicable
};
```

### 7.3 Concept Index

Semantic concepts and themes.

```c
struct ConceptEntry {
    uint32_t concept_id;
    char     concept_name[128];
    char     concept_category[32];  // Emotion, action, setting, etc.
    uint32_t occurrence_count;
    float    embedding[512];
    uint64_t occurrences_offset;
};
```

### 7.4 Temporal Index

B-tree index for fast temporal queries.

```c
struct TemporalIndex {
    uint32_t node_count;
    uint32_t depth;
    TemporalNode nodes[];
};

struct TemporalNode {
    int64_t  timestamp;
    uint32_t left_child;
    uint32_t right_child;
    uint32_t event_count;
    TemporalEvent events[];      // Events at this timestamp
};
```

### 7.5 Query Interface

QUB defines a standard query language:

```
// Example queries:
FIND shots WHERE person="John" AND shot_type=CLOSE
FIND segments WHERE audio_contains="laughter" DURATION > 2s
FIND frames WHERE object="car" AND object="bicycle" WITHIN 2s
FIND scenes WHERE description SIMILAR TO "romantic dinner" THRESHOLD 0.8
```

---

## 8. Embedding Store

Dense vector storage for similarity search.

### 8.1 Store Structure

```c
struct EmbeddingStore {
    uint32_t dimension;          // Vector dimension
    uint32_t count;              // Number of embeddings
    uint8_t  quantization;       // 0=float32, 1=float16, 2=int8
    uint8_t  index_type;         // 0=flat, 1=IVF, 2=HNSW
    uint64_t vectors_offset;
    uint64_t index_offset;
    uint32_t metadata_size;
    EmbeddingMetadata metadata[];
};

struct EmbeddingMetadata {
    uint32_t embedding_id;
    uint8_t  source_type;        // Frame, shot, segment, entity
    uint64_t source_ref;         // Reference to source
    int64_t  timestamp;          // Associated timestamp
};
```

### 8.2 Quantization

Support for reduced precision storage:

| Quantization | Precision | Size (512d) | Use Case |
|--------------|-----------|-------------|----------|
| Float32 | Full | 2048 bytes | Maximum accuracy |
| Float16 | Half | 1024 bytes | Default storage |
| Int8 | 8-bit | 512 bytes | Large archives |
| Binary | 1-bit | 64 bytes | Fast pre-filter |

---

## 9. Provenance Ledger

Complete audit trail for all AI-derived data.

### 9.1 Ledger Structure

```c
struct ProvenanceLedger {
    uint32_t entry_count;
    ProvenanceEntry entries[];
};

struct ProvenanceEntry {
    uint32_t entry_id;
    uint32_t model_id;           // Index into model registry
    int64_t  processed_timestamp;
    uint32_t parameters_size;
    char     parameters[];       // JSON model parameters
    uint8_t  checksum[32];       // SHA-256 of output
    uint32_t dependent_tracks[]; // Tracks using this entry
};
```

### 9.2 Model Registry

```c
struct ModelRegistry {
    uint32_t model_count;
    ModelDescriptor models[];
};

struct ModelDescriptor {
    uint32_t model_id;
    char     model_family[64];   // "whisper", "sam", "yolo"
    char     model_name[128];    // "whisper-large-v3"
    char     model_version[32];  // "v3.0.1"
    char     model_hash[64];     // Weights hash
    char     framework[32];      // "pytorch", "mlx", "onnx"
    char     source_url[256];    // Model source
    char     license[64];        // Model license
    int64_t  release_date;       // Model release timestamp
};
```

### 9.3 Processing History

```c
struct ProcessingHistory {
    uint32_t event_count;
    ProcessingEvent events[];
};

struct ProcessingEvent {
    int64_t  timestamp;
    uint8_t  event_type;         // 0=created, 1=updated, 2=reprocessed
    uint32_t model_ref;
    char     processor_id[64];   // Machine/app identifier
    uint32_t duration_ms;        // Processing time
    char     notes[256];
};
```

---

## 10. Extension System

### 10.1 Extension Chunks

```c
struct ExtensionChunk {
    uint8_t  fourcc[4];          // "EXTN"
    uint64_t size;
    uint8_t  vendor_uuid[16];    // Vendor identifier
    uint32_t extension_type;
    uint32_t version;
    uint32_t flags;
    uint8_t  data[];
};
```

### 10.2 Vendor Registration

Vendors register UUID namespaces for extensions:
- Anthropic: `00000000-0000-0000-0000-ANTHROPIC001`
- Adobe: `00000000-0000-0000-0000-ADOBEEXT0001`
- Apple: `00000000-0000-0000-0000-APPLE0000001`
- Avid: `00000000-0000-0000-0000-AVIDTECH0001`
- Blackmagic: `00000000-0000-0000-0000-BMDESIGN0001`

### 10.3 Extension Categories

| Type ID | Category | Description |
|---------|----------|-------------|
| 0x0001 | NLE Project | Editor-specific project data |
| 0x0002 | Color Grade | LUT references, CDL data |
| 0x0003 | VFX Metadata | Tracking data, clean plates |
| 0x0004 | Custom AI | Vendor-specific AI tracks |
| 0x0005 | Workflow | Approval status, notes |
| 0x0006 | Rights | DRM, licensing metadata |
| 0x0007 | Interchange | AAF/MXF/OTIO mapping data |
| 0x0008 | Script | Script integration, lined script data |
| 0x0009 | Dailies | Circle takes, director notes, camera reports |

---

## 11. Implementation Guidelines

### 11.1 Ingest Pipeline

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Source File │────▶│   Demuxer   │────▶│Media Tracks │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Decoder    │
                    └─────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Vision    │     │   Audio     │     │ Segmentation│
│  Pipeline   │     │  Pipeline   │     │  Pipeline   │
└─────────────┘     └─────────────┘     └─────────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                    ┌─────────────┐
                    │   Indexer   │────▶ Semantic Index
                    └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │QUB Muxer   │────▶ .qub File
                    └─────────────┘
```

### 11.2 Recommended Processing Order

1. **Media demux** — Extract video/audio streams
2. **Shot detection** — Establish temporal structure
3. **Transcription** — Audio → text with alignment
4. **Speaker diarization** — Cluster speakers
5. **Vision semantic** — Scene descriptions at shot boundaries
6. **Object detection** — Frame-by-frame with tracking
7. **Face detection** — Extract and cluster faces
8. **Segmentation** — Depth and instance masks
9. **Audio classification** — Non-speech events
10. **Embedding generation** — Dense vectors for search
11. **Index building** — Construct semantic index

### 11.3 Streaming Considerations

For streaming playback:
- Interleave AI samples with media samples
- AI data should lead media by configurable amount (default 2s)
- Support partial AI tracks (processing in progress)
- Provide fallback for missing AI tracks

### 11.4 NLE Integration

**Avid Media Composer:**
- AAF export with QUB metadata preservation
- Bin columns populated from AI track data (shot type, speaker, scene description)
- Markers from shot boundaries with locator colors by transition type
- ScriptSync replacement from transcription track with word-level sync
- PhraseFind index generated from transcription + audio classification
- SubCap integration from transcription segments
- Source browser extension for semantic queries across project media

**Final Cut Pro X:**
- FCPXML export with QUB references
- Keyword ranges from shot boundaries
- Roles from speaker diarization
- Content auto-analysis bypass (use QUB data instead)

**DaVinci Resolve:**
- OTIO (OpenTimelineIO) integration
- Smart bins from semantic queries
- Markers from shot/scene changes
- Scene cut detection bypass (use QUB shot boundaries)
- Auto-subtitle from transcription track

**Premiere Pro:**
- Panel extension for QUB queries
- Markers and subclips from AI data
- Speech-to-text replacement from transcription
- Essential Sound panel metadata from audio classification

---

## 12. File Extension and MIME Type

**File Extension:** `.qub`

**MIME Type:** `video/x-qub`

**UTI (macOS):** `com.qub.container`

---

## 13. Reference Implementation

### 13.1 Minimum Viable Reader

A compliant reader MUST:
1. Parse file header and validate magic number
2. Read track directory
3. Decode at least one video codec (H.264 recommended)
4. Decode at least one audio codec (AAC recommended)
5. Gracefully skip unknown AI track types
6. Expose AI data through documented API

### 13.2 Minimum Viable Writer

A compliant writer MUST:
1. Generate valid file header with UUID
2. Write track directory with correct offsets
3. Mux media tracks in spec-compliant format
4. Include provenance entry for any AI track written
5. Generate checksum for written data

---

## 14. Security Considerations

### 14.1 AI Track Encryption

Sensitive AI tracks (face detection, transcription) can be encrypted:

```c
struct EncryptedTrack {
    uint8_t  encryption_method;  // 0=AES-256-GCM
    uint8_t  key_derivation;     // 0=PBKDF2, 1=Argon2
    uint8_t  iv[16];
    uint8_t  auth_tag[16];
    uint32_t encrypted_size;
    uint8_t  encrypted_data[];
};
```

### 14.2 Privacy Levels

Files should declare privacy level:
- `0` — Public (no restrictions)
- `1` — Internal (no face embeddings)
- `2` — Confidential (encrypted AI tracks)
- `3` — Restricted (media-only, no AI data)

### 14.3 Redaction Support

Tracks can be selectively removed or redacted while maintaining file integrity.

---

## 15. Versioning and Compatibility

### 15.1 Version Numbering

`MAJOR.MINOR.PATCH`

- **MAJOR:** Breaking changes to core structure
- **MINOR:** New track types, backward compatible
- **PATCH:** Bug fixes, clarifications

### 15.2 Forward Compatibility

Readers MUST:
- Skip unknown chunk types
- Skip unknown track types
- Ignore unknown flags
- Process known tracks even if some fail

### 15.3 Backward Compatibility

Writers SHOULD:
- Maintain compatibility with previous minor versions
- Document breaking changes clearly
- Provide migration tools for major version changes

---

## Appendix A: FourCC Registry

| FourCC | Description | Specification Section |
|--------|-------------|----------------------|
| `QUB\0` | File identifier | §2.2 |
| `TDIR` | Track directory | §4 |
| `MTRK` | Media track | §5 |
| `ATRK` | AI track | §6 |
| `SIDX` | Semantic index | §7 |
| `EMBD` | Embedding store | §8 |
| `PROV` | Provenance ledger | §9 |
| `EXTN` | Extension chunk | §10 |
| `ENCR` | Encrypted data | §14 |

---

## Appendix B: Class ID Ontologies

### Object Detection Classes

Default ontology: COCO 80 classes + extensions

| ID | Class | ID | Class |
|----|-------|----|-------|
| 0 | person | 40 | wine glass |
| 1 | bicycle | 41 | cup |
| 2 | car | 42 | fork |
| ... | ... | ... | ... |

### Audio Event Classes

Default ontology: AudioSet (527 classes)

Top-level categories:
- Human sounds
- Animal
- Music
- Source-ambiguous
- Natural sounds
- Channel, environment, background

---

## Appendix C: Sample QUB File (Hex Dump)

```
00000000: 5155 4200 0001 0000 0001 0000 xxxx xxxx  QUB\0............
00000010: xxxx xxxx xxxx xxxx xxxx xxxx xxxx xxxx  ................
00000020: [UUID 16 bytes]                          ................
00000030: [timestamps]                             ................
...
```

---

## Appendix D: Glossary

| Term | Definition |
|------|------------|
| Chunk | Self-describing data block with FourCC and size |
| Diarization | Speaker segmentation and clustering |
| Embedding | Dense vector representation for similarity |
| FourCC | Four-character code identifying data type |
| Instance Segmentation | Per-object pixel-level masks |
| Panoptic Segmentation | Combined semantic + instance |
| Provenance | Origin and processing history of data |
| RLE | Run-length encoding for mask compression |
| Shot | Continuous camera take between cuts |
| Trimap | Three-region mask (foreground, background, unknown) |

---

## Appendix E: Reference Models

### Recommended Models by Track Type

| Track Type | Recommended Models | Notes |
|------------|-------------------|-------|
| Vision Semantic | FastVLM, Qwen-VL, LLaVA-NeXT | Balance speed/quality |
| Object Detection | YOLO v9, RT-DETR, Grounding DINO | YOLO for speed |
| Face Detection | RetinaFace, SCRFD | Include ArcFace for embedding |
| Transcription | Whisper large-v3, Distil-Whisper | Word-level timestamps |
| Diarization | pyannote 3.0, NeMo | Requires separate speaker embedding |
| Depth | Depth Anything v2, ZoeDepth | Relative depth preferred |
| Instance Seg | SAM 2, Mask2Former | SAM 2 for video consistency |
| Alpha Matte | RMBG v2, MODNet | SAM 2 for complex scenes |

---

*End of QUB Container Specification v1.0*
