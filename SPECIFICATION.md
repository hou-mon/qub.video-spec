# QUB Container Specification v1.1.0

## Information Cubed — AI-Forward Video Container

**Status:** Draft Specification
**Version:** 1.1.0
**Date:** January 2026
**Author:** Houman Shekarchi
**License:** Apache License 2.0

---

## 0. Scope, Conformance, and Notation (Normative)

### 0.1 Scope

QUB ("Information Cubed") is a container format that stores:

* **Media tracks** (video, audio, timecode, captions) in a codec-agnostic way
* **AI tracks** (vision, audio analysis, segmentation, embeddings) aligned to the same timeline
* **Indices** for fast temporal and semantic retrieval
* **Provenance** capturing model lineage, parameters, checksums, and processing history
* **Extensions** for vendor and workflow metadata

### 0.2 Conformance Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to be interpreted as described in RFC 2119.

### 0.3 Conformance Classes

An implementation MAY claim one or more of the following:

* **Class A — Playback Reader**

  * MUST parse the file header and Track Directory (`TDIR`)
  * MUST decode at least one supported video codec and one supported audio codec (Appendix C)
  * MUST safely skip unknown chunks and unknown track types

* **Class B — Metadata Reader**

  * Includes Class A
  * MUST parse AI tracks it recognizes
  * MUST expose AI samples and track relationships

* **Class C — Index / Query Reader**

  * Includes Class B
  * MUST parse the Semantic Index (`SIDX`) and Embedding Store (`EMBD`) if present
  * MUST support timestamp-based seeking via TrackIndex (§8)

* **Writer**

  * MUST write a valid header and chunk stream
  * MUST write a canonical Track Directory (`TDIR`, §6.2)
  * MUST write TrackIndex structures for written tracks (§8)
  * MUST write provenance entries for all AI tracks it writes (or explicitly set `model_ref = 0`)

### 0.4 Data Types

* All integer fields are **little-endian**
* Floating point values are IEEE-754
* All timestamps are signed 64-bit integers representing microseconds relative to file timeline start (t = 0)

### 0.5 Alignment and Padding

* All chunks MUST begin at an **8-byte aligned** file offset
* Chunk payloads MAY be padded with zero bytes for alignment
* Chunk `Size` excludes padding
* Readers MUST ignore padding bytes

### 0.6 Strings

* Variable-length text is UTF-8
* Fixed-size character fields MUST be zero-terminated if shorter than their capacity

---

## 1. Executive Summary (Informative)

QUB is a next-generation container designed to store not only audio and video, but the full semantic understanding of media. AI preprocessing is performed once at ingest and persisted alongside the media, enabling query-driven editing workflows and eliminating redundant inference.

Design principles:

1. **Inference Once, Query Forever**
2. **Codec Agnostic** (wrap any modern audio/video codec)
3. **Model-Versioned** (provenance for all AI outputs)
4. **Temporally Dense** (microsecond alignment)
5. **Query Native** (optional semantic index + embeddings)
6. **Graceful Degradation** (standard playback possible; AI optional)
7. **Extensible by Design**

---

## 2. File Format Overview (Normative)

### 2.1 High-Level Layout

A `.qub` file consists of:

1. A fixed-size file header (`QUBHeader`)
2. A sequence of chunks (FourCC + Size + Flags + Payload)

Header offsets point to required and optional top-level chunks:

* `TDIR` — Track Directory (REQUIRED)
* `SIDX` — Semantic Index (OPTIONAL)
* `EMBD` — Embedding Store (OPTIONAL)
* `PROV` — Provenance Ledger (REQUIRED if AI tracks exist)

### 2.2 Chunk Header

```
┌────────────┬────────────┬────────────┬─────────────────────┐
│  FourCC    │   Size     │   Flags    │      Payload        │
│  4 bytes   │  8 bytes   │  4 bytes   │    Size bytes       │
└────────────┴────────────┴────────────┴─────────────────────┘
```

### 2.3 Chunk Ordering

* `TDIR` MUST appear exactly once
* `PROV` SHOULD appear at most once
* `SIDX` and `EMBD` MAY appear
* Multiple `MTRK` / `ATRK` chunks MAY exist for fragmentation
* Readers MUST rely on offsets, not physical order

### 2.4 Chunk Flags

```c
enum QUBChunkFlags {
  CHUNK_FLAG_NONE        = 0x00000000,
  CHUNK_FLAG_COMPRESSED  = 0x00000001, // TLV_COMPRESSION describes compression
  CHUNK_FLAG_ENCRYPTED   = 0x00000002, // Payload is EncryptedChunk (§15)
  CHUNK_FLAG_FRAGMENT    = 0x00000004, // Part of fragmented sequence
};
```

### 2.5 Reserved FourCC Registry

| FourCC | Description               |
| ------ | ------------------------- |
| `QUB�` | File identifier (magic)   |
| `TDIR` | Track directory           |
| `MTRK` | Media track data          |
| `ATRK` | AI track data             |
| `SIDX` | Semantic index            |
| `EMBD` | Embedding store           |
| `PROV` | Provenance ledger         |
| `EXTN` | Extension chunk           |
| `ENCR` | Encrypted payload wrapper |

---

## 3. File Header (Normative)

```c
struct QUBHeader {
  uint8_t  magic[4];            // "QUB�"
  uint16_t version_major;       // 1
  uint16_t version_minor;       // 1
  uint32_t flags;               // QUBFlags
  uint8_t  uuid[16];            // File UUID

  int64_t  created_timestamp;   // Unix epoch (µs)
  int64_t  modified_timestamp;  // Unix epoch (µs)

  uint64_t duration_us;         // File duration (µs)
  uint32_t track_count;         // Number of tracks
  uint32_t header_size;         // Header size in bytes

  uint64_t track_dir_offset;    // ABS offset to TDIR payload start
  uint64_t semantic_idx_offset; // ABS offset to SIDX payload start or 0
  uint64_t embedding_offset;    // ABS offset to EMBD payload start or 0
  uint64_t provenance_offset;   // ABS offset to PROV payload start or 0

  uint8_t  reserved[64];        // MUST be zero
};
```

```c
enum QUBFlags {
  QUB_FLAG_STREAMABLE   = 0x0001,
  QUB_FLAG_ENCRYPTED    = 0x0002,
  QUB_FLAG_FRAGMENTED   = 0x0004,
  QUB_FLAG_COMPLETE_AI  = 0x0008,
  QUB_FLAG_PARTIAL_AI   = 0x0010,
  QUB_FLAG_EMBEDDINGS   = 0x0020,
  QUB_FLAG_REALTIME     = 0x0040,
};
```

---

## 4. Metadata Encoding (Normative)

### 4.1 TLV Format

```c
struct QUBTLV {
  uint16_t type;
  uint16_t flags;
  uint32_t length;
  uint8_t  value[length];
};
```

Unknown TLVs MUST be skipped using `length`.

### 4.2 TLV Type Registry

|   Type | Name                | Description                           |
| -----: | ------------------- | ------------------------------------- |
| 0x0001 | TLV_CODEC_CONFIG    | Codec extradata / decoder config blob |
| 0x0002 | TLV_COLOR_INFO      | Color primaries/transfer/matrix/HDR   |
| 0x0003 | TLV_AUDIO_LAYOUT    | Channel layout + speaker positions    |
| 0x0004 | TLV_COMPRESSION     | Compression type/params               |
| 0x0005 | TLV_TRACK_RELATIONS | Track relationship table              |
| 0x0006 | TLV_ONTOLOGY        | Ontology ID/version for class IDs     |
| 0x0007 | TLV_TEXT_ENCODING   | Text encoding options                 |
| 0x0008 | TLV_SCHEMA_ID       | Payload schema identifier/hash        |
| 0x0009 | TLV_PRIVACY_LEVEL   | File privacy level (0–3)              |

---

## 5. Timing Model (Normative)

* All timestamps are microseconds relative to timeline start (t = 0)
* Variable frame rate is supported
* Media samples MUST have `duration_us > 0`
* Event/marker samples MAY have `duration_us = 0`
* AI tracks MUST set `dts = 0`

---

## 6. Track System (Normative)

### 6.1 Track Types

* `0x00–0x3F`: Media tracks
* `0x40–0xFF`: AI tracks

(Full registry in Appendix B.)

### 6.2 Track Directory (`TDIR`) — Canonical Layout

**All offsets inside `TDIR` are relative to the start of the `TDIR` payload.**

Layout:

1. `TrackDirectory`
2. `TrackDescriptor[track_count]`
3. `uint32_t string_table_size`
4. `uint8_t string_table[string_table_size]`
5. `uint32_t tlv_table_size`
6. `uint8_t tlv_table[tlv_table_size]`
7. Optional `TrackRelation[]`

```c
struct TrackDirectory {
  uint32_t version;             // 1
  uint32_t track_count;
  uint32_t descriptors_offset;
  uint32_t string_table_offset;
  uint32_t tlv_table_offset;
  uint32_t relations_offset;    // or 0
  uint32_t relations_size;      // or 0
  uint32_t reserved0;
  uint32_t reserved1;
};
```

### 6.3 Track Descriptor

```c
struct TrackDescriptor {
  uint32_t track_id;
  uint8_t  track_type;
  uint8_t  track_subtype;
  uint16_t track_flags;

  uint8_t  codec_fourcc[4];
  uint8_t  language[3];
  uint8_t  reserved0;

  uint64_t data_offset;         // ABS file offset (points into MTRK/ATRK sample stream)
  uint64_t data_size;

  uint64_t index_offset;        // ABS file offset to TrackIndex
  uint64_t index_size;

  uint32_t sample_count;
  uint32_t model_ref;           // Provenance entry ID or 0
  uint64_t duration_us;

  uint32_t name_offset;         // REL offset into string_table
  uint32_t name_len;

  uint32_t metadata_offset;     // REL offset into tlv_table
  uint32_t metadata_size;
};
```

### 6.4 Track Flags

```c
enum TrackFlags {
  TRACK_ENABLED       = 0x0001,
  TRACK_DEFAULT       = 0x0002,
  TRACK_FORCED        = 0x0004,
  TRACK_LOSSLESS      = 0x0008,
  TRACK_COMPRESSED    = 0x0010,
  TRACK_DELTA_ENCODED = 0x0020,
  TRACK_SPARSE        = 0x0040,
  TRACK_DERIVED       = 0x0080,
};
```

### 6.5 Track Relations

```c
enum RelationType {
  REL_DERIVED_FROM = 1,
  REL_ALIGNS_TO    = 2,
  REL_REFERENCES   = 3,
};

struct TrackRelation {
  uint32_t from_track_id;
  uint32_t to_track_id;
  uint16_t relation_type;
  uint16_t flags;
  uint32_t reserved;
};
```

---

## 7. Track Data Regions (Normative)

### 7.1 Track Chunks (`MTRK` / `ATRK`)

```c
struct TrackChunkHeader {
  uint32_t track_id;
  uint32_t reserved0;
  uint64_t sequence_number; // 0 if unfragmented
};
```

### 7.2 Sample Block Framing

```c
struct SampleBlockHeader {
  int64_t  pts;
  int64_t  dts;
  uint32_t duration_us;
  uint32_t flags;
  uint32_t payload_size;
  uint32_t payload_type;
  uint8_t  payload[payload_size];
};
```

### 7.3 Payload Type Registry

Media tracks:

* `payload_type = 0`: codec elementary stream bytes

AI tracks:

* `payload_type = 0`: QUB binary schema v1 (Section 13)
* `payload_type = 1`: JSON UTF-8
* `payload_type = 2`: MessagePack
* `payload_type = 3`: Protobuf (requires TLV_SCHEMA_ID)

### 7.4 Sample Flags

```c
enum SampleFlags {
  SAMPLE_NONE      = 0x00000000,
  SAMPLE_KEYFRAME  = 0x00000001,
  SAMPLE_DISCONT   = 0x00000002,
  SAMPLE_REDACTED  = 0x00000004,
  SAMPLE_ENCRYPTED = 0x00000008,
};
```

---

## 8. Track Index (Normative)

```c
struct TrackIndex {
  uint32_t version;      // 1
  uint32_t entry_count;
  uint8_t  index_type;   // 0 = flat
  uint8_t  reserved0[3];
  IndexEntry entries[entry_count];
};

struct IndexEntry {
  int64_t  pts;
  uint64_t file_offset;  // ABS offset to SampleBlockHeader
  uint32_t sample_size;
  uint32_t flags;
};
```

---

## 9. Semantic Index (`SIDX`) (Normative Storage)

All offsets in `SIDX` are relative to the start of the `SIDX` payload.

```c
struct SemanticIndex {
  uint32_t version;                // 1
  uint32_t flags;

  uint32_t entity_index_offset;    // REL or 0
  uint32_t concept_index_offset;   // REL or 0
  uint32_t temporal_index_offset;  // REL or 0

  uint32_t capabilities_offset;    // REL or 0 (JSON UTF-8)
  uint32_t capabilities_size;

  uint32_t embedding_dimension;
  uint32_t entity_count;
  uint32_t concept_count;

  uint32_t reserved0;
  uint32_t reserved1;
};
```

### 9.1 Entity Index

```c
struct EntityEntry {
  uint32_t entity_id;
  uint8_t  entity_type;          // registry-defined
  uint8_t  reserved0[3];
  uint32_t name_len;
  uint32_t name_offset;          // REL offset to UTF-8 name
  uint32_t alias_count;
  uint32_t occurrence_count;
  uint32_t occurrences_offset;   // REL offset to EntityOccurrence[occurrence_count]
  uint8_t  has_embedding;
  uint8_t  reserved1[3];
  float    representative_embedding[512]; // valid if has_embedding=1
};

struct EntityOccurrence {
  int64_t  start_pts;
  int64_t  end_pts;
  uint32_t source_track_id;
  float    confidence;
  uint8_t  has_bbox;
  uint8_t  reserved2[3];
  // BoundingBox follows if has_bbox=1
};
```

### 9.2 Concept Index

```c
struct ConceptEntry {
  uint32_t concept_id;
  uint32_t name_len;
  uint32_t name_offset;          // REL offset to UTF-8 name
  uint32_t category_len;
  uint32_t category_offset;      // REL offset to UTF-8 category
  uint32_t occurrence_count;
  uint32_t occurrences_offset;   // REL offset
  uint8_t  has_embedding;
  uint8_t  reserved0[3];
  float    embedding[512];       // valid if has_embedding=1
};
```

### 9.3 Temporal Cue Table

```c
struct TemporalCueTable {
  uint32_t version;     // 1
  uint32_t cue_count;
  // TemporalCue[cue_count]
};

struct TemporalCue {
  int64_t  timestamp;
  uint32_t event_count;
  uint32_t events_offset; // REL offset to TemporalEventRef[event_count]
};

struct TemporalEventRef {
  uint32_t track_id;
  uint32_t sample_index;
};
```

---

## 10. Embedding Store (`EMBD`) (Normative)

All offsets in `EMBD` are relative to the start of the `EMBD` payload.

```c
struct EmbeddingStore {
  uint32_t version;           // 1
  uint32_t dimension;
  uint32_t count;
  uint8_t  quantization;      // 0=f32,1=f16,2=int8,3=binary
  uint8_t  index_type;        // 0=flat,1=IVF,2=HNSW
  uint16_t reserved;

  uint32_t vectors_offset;    // REL
  uint32_t vectors_size;
  uint32_t index_offset;      // REL
  uint32_t index_size;

  uint32_t metadata_offset;   // REL
  uint32_t metadata_count;
  uint32_t metadata_entry_size;
};

struct EmbeddingMetadata {
  uint32_t embedding_id;
  uint8_t  source_type;   // 0=frame,1=shot,2=segment,3=entity,4=custom
  uint8_t  reserved0[3];
  uint64_t source_ref;
  int64_t  timestamp;
  uint32_t track_id;
  float    confidence;
};
```

---

## 11. Provenance Ledger (`PROV`) (Normative)

All offsets in `PROV` are relative to the start of the `PROV` payload.

```c
struct ProvenanceLedger {
  uint32_t version;                 // 1
  uint32_t entry_count;
  uint32_t model_registry_offset;   // REL
  uint32_t processing_history_offset;// REL
  uint32_t entries_offset;          // REL
};

struct ProvenanceEntry {
  uint32_t entry_id;
  uint32_t model_id;
  int64_t  processed_timestamp;
  uint32_t parameters_size;
  uint32_t parameters_offset;       // REL to UTF-8 JSON
  uint8_t  checksum[32];            // SHA-256
  uint8_t  checksum_scope;          // 0=payload bytes,1=sample stream
  uint8_t  reserved0[3];
  uint32_t dependent_track_count;
  uint32_t dependent_tracks_offset; // REL to uint32_t[dependent_track_count]
};

struct ModelRegistry {
  uint32_t version;                 // 1
  uint32_t model_count;
  // ModelDescriptor[model_count]
};

struct ModelDescriptor {
  uint32_t model_id;
  uint32_t family_len;  uint32_t family_offset;
  uint32_t name_len;    uint32_t name_offset;
  uint32_t version_len; uint32_t version_offset;
  uint32_t hash_len;    uint32_t hash_offset;
  uint32_t framework_len; uint32_t framework_offset;
  uint32_t source_url_len; uint32_t source_url_offset;
  uint32_t license_len;  uint32_t license_offset;
  int64_t  release_date;
};

struct ProcessingHistory {
  uint32_t version;                 // 1
  uint32_t event_count;
  // ProcessingEvent[event_count]
};

struct ProcessingEvent {
  int64_t  timestamp;
  uint8_t  event_type;              // 0=created,1=updated,2=reprocessed
  uint8_t  reserved0[3];
  uint32_t provenance_entry_id;
  uint32_t processor_id_len;  uint32_t processor_id_offset;
  uint32_t duration_ms;
  uint32_t notes_len;         uint32_t notes_offset;
};
```

---

## 12. Media Codec Registry (Normative)

QUB is codec-agnostic. Supported codec identifiers are carried in `TrackDescriptor.codec_fourcc`, with decoder configuration stored in `TLV_CODEC_CONFIG`.

See Appendix C for the initial registry (H.264, HEVC, AV1, ProRes, AAC, FLAC, Opus, PCM, etc.).

---

## 13. AI Track Schemas (Normative)

This section defines the **normative binary schemas** for standard AI track types. These schemas are used when `payload_type = 0`.

### 13.1 Common AI Conventions

* `dts` MUST be 0 for all AI samples
* Optional fields MUST be guarded by explicit presence flags
* Spatial data uses normalized coordinates `[0.0–1.0]`

```c
struct BoundingBox {
  float x;
  float y;
  float width;
  float height;
};
```

---

### 13.2 Vision Semantic Track (0x40)

```c
struct VisionSemanticSample {
  int64_t  start_pts;
  int64_t  end_pts;
  float    confidence;
  uint32_t description_len;
  // UTF-8 description[description_len]
  uint32_t tag_count;
  // SemanticTag[tag_count]
};

struct SemanticTag {
  uint32_t category_id;
  uint32_t value_id;
  float    confidence;
  uint8_t  has_bbox;
  uint8_t  reserved[3];
  // BoundingBox bbox (if has_bbox=1)
};
```

---

### 13.3 Vision Object Detection Track (0x41)

```c
struct ObjectDetectionFrame {
  int64_t pts;
  uint32_t object_count;
  // DetectedObject[object_count]
};

struct DetectedObject {
  uint32_t track_id;
  uint32_t class_id;
  float    confidence;
  BoundingBox bbox;
  uint32_t class_name_len;
  // UTF-8 class_name
  uint32_t attribute_count;
  // ObjectAttribute[attribute_count]
};

struct ObjectAttribute {
  uint32_t key_id;
  uint32_t value_id;
  float    confidence;
};
```

---

### 13.4 Vision Face Detection Track (0x42)

```c
struct FaceDetectionFrame {
  int64_t pts;
  uint32_t face_count;
  // DetectedFace[face_count]
};

struct DetectedFace {
  uint32_t person_id;
  BoundingBox bbox;
  float    pose[3];          // yaw, pitch, roll
  float    quality;
  uint8_t  has_landmarks;
  uint8_t  has_embedding;
  uint8_t  attributes;
  uint8_t  reserved0;
  // landmarks[68][2] if present
  // embedding[512] if present
  uint32_t assigned_name_len;
  // UTF-8 assigned_name
};
```

---

### 13.5 Vision OCR Track (0x43)

```c
struct OCRFrame {
  int64_t pts;
  uint32_t region_count;
  // OCRRegion[region_count]
};

struct OCRRegion {
  float polygon[8];
  float confidence;
  uint8_t text_type; // 0=scene,1=graphic,2=caption
  uint8_t reserved0[3];
  uint32_t language_len;
  // UTF-8 language
  uint32_t text_len;
  // UTF-8 text
};
```

---

### 13.6 Vision Action Recognition Track (0x44)

```c
struct ActionSegment {
  int64_t start_pts;
  int64_t end_pts;
  uint32_t class_id;
  float    confidence;
  uint32_t class_name_len;
  // UTF-8 class_name
};
```

---

### 13.7 Temporal Shot Boundaries & Shot Descriptors (0x50)

```c
enum TransitionType {
  TRANS_CUT      = 0,
  TRANS_DISSOLVE = 1,
  TRANS_FADE_OUT = 2,
  TRANS_FADE_IN  = 3,
  TRANS_WIPE     = 4,
  TRANS_OTHER    = 255,
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
  CAM_STATIC      = 0,
  CAM_PAN_LEFT    = 1,
  CAM_PAN_RIGHT   = 2,
  CAM_TILT_UP     = 3,
  CAM_TILT_DOWN   = 4,
  CAM_ZOOM_IN     = 5,
  CAM_ZOOM_OUT    = 6,
  CAM_DOLLY_IN    = 7,
  CAM_DOLLY_OUT   = 8,
  CAM_HANDHELD    = 13,
  CAM_STEADICAM   = 14,
  CAM_DRONE       = 15,
};

struct ShotBoundary {
  int64_t  timestamp;
  uint8_t  transition_type;
  uint8_t  reserved0[3];
  float    confidence;
  uint32_t duration_us;
  uint32_t shot_before_id;
  uint32_t shot_after_id;
};

struct ShotDescriptor {
  uint32_t shot_id;
  int64_t  start_pts;
  int64_t  end_pts;
  uint8_t  shot_type;
  uint8_t  camera_motion;
  uint8_t  reserved1[2];
  float    avg_brightness;
  float    motion_intensity;
  uint32_t description_len;
  // UTF-8 description
};
```

---

### 13.8 Temporal Scene Changes Track (0x51)

```c
struct SceneSegment {
  uint32_t scene_id;
  int64_t  start_pts;
  int64_t  end_pts;
  float    confidence;
  uint32_t description_len;
  // UTF-8 description
};
```

---

### 13.9 Audio Transcription Track (0x60)

```c
struct TranscriptionSegment {
  int64_t  start_pts;
  int64_t  end_pts;
  uint32_t speaker_id;
  float    confidence;
  uint32_t language_len;
  // UTF-8 language
  uint32_t text_len;
  // UTF-8 text
  uint32_t word_count;
  // WordAlignment[word_count]
};

struct WordAlignment {
  int64_t start_pts;
  int64_t end_pts;
  float   confidence;
  uint32_t text_offset;
  uint32_t text_length;
  float   probability;
};
```

---

### 13.10 Audio Speaker Diarization Track (0x61)

```c
struct DiarizationSegment {
  int64_t start_pts;
  int64_t end_pts;
  uint32_t speaker_id;
  float    confidence;
  uint8_t  has_embedding;
  uint8_t  reserved0[3];
  // embedding[256] if present
};
```

---

### 13.11 Audio Classification Track (0x62)

```c
struct AudioClassificationFrame {
  int64_t start_pts;
  int64_t end_pts;
  uint32_t event_count;
  // AudioEvent[event_count]
};

struct AudioEvent {
  uint32_t class_id;
  float    confidence;
  float    loudness_db;
  uint8_t  channel_mask;
  uint8_t  reserved0[3];
  uint32_t class_name_len;
  // UTF-8 class_name
};
```

---

### 13.12 Depth Map Track (0x70)

```c
struct DepthMapFrame {
  int64_t  pts;
  uint16_t width;
  uint16_t height;
  uint8_t  precision_bits;
  uint8_t  encoding;       // 0=raw,1=png,2=exr
  float    min_depth;
  float    max_depth;
  uint32_t data_size;
  // data[data_size]
};
```

---

### 13.13 Semantic Segmentation Track (0x71)

```c
struct SemanticSegFrame {
  int64_t  pts;
  uint16_t width;
  uint16_t height;
  uint8_t  encoding;       // 0=RLE,1=bitmap
  uint8_t  reserved0[3];
  uint32_t data_size;
  // data[data_size]
};
```

---

### 13.14 Instance Segmentation Track (0x72)

```c
struct InstanceSegFrame {
  int64_t  pts;
  uint16_t width;
  uint16_t height;
  uint32_t instance_count;
  // InstanceMask[instance_count]
};

struct InstanceMask {
  uint32_t object_track_id;    // references object detection track IDs
  uint32_t class_id;
  BoundingBox bbox;
  uint8_t  encoding;           // 0=RLE,1=polygon,2=bitmap
  uint8_t  reserved0[3];
  uint32_t mask_size;
  // mask_data[mask_size]
};
```

---

### 13.15 Panoptic Segmentation Track (0x73)

```c
struct PanopticSegFrame {
  int64_t  pts;
  uint16_t width;
  uint16_t height;
  uint8_t  encoding;       // 0=RLE,1=bitmap
  uint8_t  reserved0[3];
  uint32_t data_size;
  // data[data_size]
};
```

---

### 13.16 Alpha Matte Track (0x74)

```c
struct AlphaMatteFrame {
  int64_t  pts;
  uint16_t width;
  uint16_t height;
  uint8_t  precision_bits;  // 8 or 16
  uint8_t  encoding;        // 0=raw,1=png
  uint32_t subject_id;
  float    trimap_threshold;
  uint32_t data_size;
  // data[data_size]
};
```

---

### 13.17 Saliency Track (0x75)

```c
struct SaliencyFrame {
  int64_t  pts;
  uint16_t width;
  uint16_t height;
  uint8_t  encoding;       // 0=raw,1=png
  uint8_t  reserved0[3];
  uint32_t data_size;
  // data[data_size]
};
```

---

## 14. Extension System (`EXTN`) (Normative)

```c
struct ExtensionChunk {
  uint8_t  fourcc[4];         // "EXTN"
  uint64_t size;
  uint8_t  vendor_uuid[16];   // RFC 4122 UUID
  uint32_t extension_type;
  uint32_t version;
  uint32_t flags;
  // data[]
};
```

Readers MUST skip unknown extensions.

---

## 15. Security, Privacy, and Redaction (`ENCR`) (Normative)

### 15.1 Encryption Wrapper

```c
struct EncryptedChunk {
  uint8_t  encryption_method; // 0=AES-256-GCM
  uint8_t  key_derivation;    // 0=PBKDF2,1=Argon2,2=external-KMS
  uint16_t reserved0;
  uint8_t  kid[16];
  uint8_t  iv[12];
  uint8_t  auth_tag[16];
  uint32_t encrypted_size;
  uint8_t  encrypted_data[encrypted_size];
};
```

* IV MUST be unique per encrypted payload
* Encryption MAY be applied at chunk-level or sample-level

### 15.2 Privacy Levels

A file SHOULD declare privacy level via `TLV_PRIVACY_LEVEL`:

* 0 — Public
* 1 — Internal
* 2 — Confidential (encrypted AI tracks)
* 3 — Restricted (media-only)

### 15.3 Redaction

* Tracks MAY be removed entirely
* Redacted samples SHOULD set `SAMPLE_REDACTED` and MAY use `payload_size=0`
* Indices MUST remain valid

---

## 16. Versioning and Compatibility (Normative)

* Version format: MAJOR.MINOR.PATCH
* Readers MUST skip unknown chunks, tracks, TLVs and ignore unknown flags
* Writers SHOULD preserve backward compatibility within a major version

---

## 17. File Extension and MIME Type

* **Extension:** `.qub`
* **MIME Type:** `video/x-qub`
* **UTI (macOS):** `com.qub.container`

---

## Appendix A — FourCC Registry

(See §2.5)

## Appendix B — Track Type Registry (v1.1.0)

### Media Tracks (0x00–0x3F)

* `0x01` — Primary Video
* `0x02` — Secondary Video (proxy/multicam)
* `0x10` — Primary Audio
* `0x11` — Secondary Audio
* `0x20` — Timecode
* `0x21` — Closed Captions (source)

### AI Tracks (0x40–0xFF)

* `0x40` — Vision Semantic
* `0x41` — Vision Object Detection
* `0x42` — Vision Face Detection
* `0x43` — Vision OCR
* `0x44` — Vision Action Recognition
* `0x50` — Temporal Shot Boundaries
* `0x51` — Temporal Scene Changes
* `0x52` — Temporal Motion Analysis (reserved)
* `0x53` — Temporal Camera Motion (reserved)
* `0x60` — Audio Transcription
* `0x61` — Audio Speaker Diarization
* `0x62` — Audio Classification
* `0x63` — Audio Music Analysis (reserved)
* `0x64` — Audio Emotion/Sentiment (reserved)
* `0x70` — Segmentation Depth Map
* `0x71` — Segmentation Semantic
* `0x72` — Segmentation Instance
* `0x73` — Segmentation Panoptic
* `0x74` — Segmentation Alpha Matte
* `0x75` — Segmentation Saliency
* `0x80` — Embedding Dense (per-frame)
* `0x81` — Embedding Sparse (keyframes)
* `0x82` — Embedding Audio
* `0x90` — Custom/Extension

## Appendix C — Codec FourCC Registry (Initial)

### Video

| Codec      | FourCC |
| ---------- | ------ |
| H.264/AVC  | `avc1` |
| H.265/HEVC | `hvc1` |
| AV1        | `av01` |
| VP9        | `vp09` |
| ProRes     | `apch` |
| DNxHD/HR   | `AVdn` |
| JPEG 2000  | `mjp2` |
| RAW        | `raw ` |

### Audio

| Codec        | FourCC |
| ------------ | ------ |
| AAC          | `mp4a` |
| FLAC         | `fLaC` |
| Opus         | `Opus` |
| PCM          | `lpcm` |
| Dolby AC-3   | `ac-3` |
| Dolby E-AC-3 | `ec-3` |
| DTS          | `dtsc` |

---

## Appendix D — Query Language & Capabilities (Informative)

QUB defines an **optional query layer** that operates over the Semantic Index (`SIDX`) and Embedding Store (`EMBD`). The container itself does **not mandate** a specific query engine implementation, but it **does define**:

* The data required to answer semantic queries
* A capability declaration contract
* A reference query grammar for interoperability

### D.1 Capabilities Declaration

If a file supports semantic queries, `SIDX.capabilities_offset` MUST point to a UTF-8 JSON object describing supported operations.

Example:

```json
{
  "query_language": "QQL",
  "version": "0.1",
  "supports": {
    "entity": true,
    "concept": true,
    "temporal": true,
    "similarity": true,
    "spatial": true
  },
  "similarity_metrics": ["cosine", "dot"],
  "embedding_model_ref": 3
}
```

### D.2 Reference Query Grammar (QQL)

The following grammar is **informative but canonical**. Implementations MAY support supersets.

Examples:

```
FIND shots WHERE person = "John"
FIND scenes WHERE description SIMILAR TO "romantic dinner" THRESHOLD 0.8
FIND segments WHERE audio_contains = "laughter" DURATION > 2s
FIND frames WHERE object = "car" AND object = "bicycle" WITHIN 2s
FIND faces WHERE identity = "Alice" AND screen_time > 5s
```

Semantics:

* `shots`, `scenes`, `frames`, `segments` map to AI track types
* Entity names resolve through the Entity Index
* Similarity queries use embeddings referenced in `EMBD`

---

## Appendix E — Ingest Pipeline & Processing Order (Informative)

QUB is designed around the principle **Inference Once, Query Forever**. The following ingest pipeline is RECOMMENDED to produce a complete, internally consistent file.

### E.1 Reference Ingest Pipeline

```
┌─────────────┐
│ Source Media│
└──────┬──────┘
       ▼
┌─────────────┐
│  Demuxer    │────▶ Media Tracks (MTRK)
└──────┬──────┘
       ▼
┌─────────────┐
│  Decoder    │
└──────┬──────┘
       ▼
┌───────────────────────────────────────────┐
│               AI Pipelines                │
│  Vision  |  Audio  |  Segmentation        │
└──────┬──────────────────────────────┬─────┘
       ▼                              ▼
┌─────────────┐               ┌─────────────┐
│ Shot / Scene│               │Transcription│
│ Detection   │               │+ Diarization│
└──────┬──────┘               └──────┬──────┘
       ▼                              ▼
┌───────────────────────────────────────────┐
│   Object / Face / OCR / Action Analysis   │
└──────┬──────────────────────────────┬─────┘
       ▼                              ▼
┌─────────────┐               ┌─────────────┐
│ Segmentation│               │ Audio Events│
└──────┬──────┘               └──────┬──────┘
       ▼                              ▼
┌───────────────────────────────────────────┐
│        Embedding Generation (EMBD)        │
└──────┬──────────────────────────────┬─────┘
       ▼                              ▼
┌───────────────────────────────────────────┐
│     Semantic Index Construction (SIDX)    │
└──────┬──────────────────────────────┬─────┘
       ▼                              ▼
┌─────────────┐               ┌─────────────┐
│ Provenance  │               │  QUB Muxer  │
│ Ledger      │               │             │
└─────────────┘               └─────────────┘
```

### E.2 Recommended Processing Order

1. Media demux and decode
2. Shot boundary detection
3. Scene segmentation
4. Audio transcription with word-level timestamps
5. Speaker diarization
6. Vision semantic description (per shot)
7. Object detection with tracking IDs
8. Face detection and clustering
9. OCR (graphics + captions)
10. Action recognition
11. Segmentation (depth, semantic, instance, panoptic, matte)
12. Audio event classification
13. Embedding generation
14. Semantic index build
15. Provenance ledger finalization

---

## Appendix F — NLE Integration Guidelines (Informative)

This section illustrates how QUB enables **editorial workflows that do not exist with current containers**.

### F.1 Avid Media Composer

* Shot boundaries → locators (color-coded by transition type)
* Shot descriptors → bin columns (shot type, motion, brightness)
* Transcription → ScriptSync replacement (word-level)
* Speaker diarization → roles / character tracks
* OCR → searchable markers and graphic replacement hooks
* Semantic queries → smart bins ("all close-ups of John")

### F.2 Final Cut Pro

* Shot boundaries → keyword ranges
* Scene segments → compound clips
* Speaker IDs → roles
* OCR → searchable metadata fields
* AI analysis bypasses built-in FCP analysis

### F.3 DaVinci Resolve

* Shot/scene cuts → timeline markers
* Transcription → subtitle tracks
* Object/face IDs → magic mask linking
* Semantic queries → smart bins

### F.4 Adobe Premiere Pro

* AI tracks → panel extension
* Transcription → Text-based editing replacement
* Audio classification → Essential Sound metadata
* OCR → caption + graphic workflows

---

*End of QUB Container Specification v1.1.0*
