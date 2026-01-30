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

* Media tracks (video, audio, timecode, captions)
* AI tracks (vision, audio analysis, segmentation, embeddings)
* Index structures for temporal and semantic retrieval
* Provenance data capturing model lineage and processing history

### 0.2 Conformance Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to be interpreted as described in RFC 2119.

### 0.3 Conformance Classes

An implementation MAY claim one or more of the following:

* **Class A — Playback Reader**

  * Parses the file header and Track Directory (`TDIR`)
  * Decodes at least one supported video codec and one audio codec
  * Safely skips unknown chunks and track types

* **Class B — Metadata Reader**

  * Includes Class A
  * Parses recognized AI tracks
  * Exposes AI samples and track relationships

* **Class C — Index / Query Reader**

  * Includes Class B
  * Parses the Semantic Index (`SIDX`) and Embedding Store (`EMBD`) if present
  * Supports timestamp-based seeking via TrackIndex

* **Writer**

  * Writes a valid file header and chunk stream
  * Writes a canonical Track Directory (`TDIR`)
  * Writes per-track TrackIndex structures
  * Writes provenance entries for all AI tracks, or explicitly sets `model_ref = 0`

### 0.4 Data Types

* All integer fields are little-endian
* Floating point values are IEEE‑754
* All timestamps are signed 64‑bit integers representing microseconds relative to file timeline start (t = 0)

### 0.5 Alignment and Padding

* All chunks MUST begin at an 8‑byte aligned file offset
* Chunk payloads MAY be padded with zero bytes for alignment
* Chunk `Size` excludes padding
* Readers MUST ignore padding bytes

### 0.6 Strings

* Variable-length text is UTF‑8
* Fixed-size character fields MUST be zero‑terminated if shorter than their capacity

---

## 1. Executive Summary (Informative)

QUB is a next‑generation container designed to store not only audio and video, but the full semantic understanding of media. AI preprocessing is performed once at ingest and persisted alongside the media, enabling query‑driven editing workflows and eliminating redundant inference.

Design principles:

1. Inference Once, Query Forever
2. Codec Agnostic
3. Model‑Versioned Provenance
4. Temporally Dense (microsecond alignment)
5. Query Native (optional semantic index)
6. Graceful Degradation (unknown data skipped)
7. Extensible by Design

---

## 2. File Format Overview (Normative)

### 2.1 High‑Level Layout

A `.qub` file consists of:

1. A fixed‑size file header (`QUBHeader`)
2. A sequence of chunks (FourCC + Size + Flags + Payload)

Header offsets point to required and optional top‑level chunks:

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

* **FourCC:** ASCII identifier
* **Size:** payload size in bytes (excluding padding)
* **Flags:** chunk flags (see §2.4)

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
  CHUNK_FLAG_COMPRESSED  = 0x00000001,
  CHUNK_FLAG_ENCRYPTED   = 0x00000002,
  CHUNK_FLAG_FRAGMENT    = 0x00000004,
};
```

### 2.5 Reserved FourCC Registry

| FourCC  | Description       |
| ------- | ----------------- |
| `QUB\0` | File identifier   |
| `TDIR`  | Track directory   |
| `MTRK`  | Media track data  |
| `ATRK`  | AI track data     |
| `SIDX`  | Semantic index    |
| `EMBD`  | Embedding store   |
| `PROV`  | Provenance ledger |
| `EXTN`  | Extension chunk   |
| `ENCR`  | Encrypted payload |

---

## 3. File Header (Normative)

```c
struct QUBHeader {
  uint8_t  magic[4];            // "QUB\x00"
  uint16_t version_major;       // 1
  uint16_t version_minor;       // 1
  uint32_t flags;               // QUBFlags
  uint8_t  uuid[16];            // File UUID

  int64_t  created_timestamp;   // Unix epoch (µs)
  int64_t  modified_timestamp;  // Unix epoch (µs)

  uint64_t duration_us;         // File duration
  uint32_t track_count;         // Number of tracks
  uint32_t header_size;         // Header size in bytes

  uint64_t track_dir_offset;    // ABS offset to TDIR payload
  uint64_t semantic_idx_offset; // ABS offset to SIDX payload or 0
  uint64_t embedding_offset;    // ABS offset to EMBD payload or 0
  uint64_t provenance_offset;   // ABS offset to PROV payload or 0

  uint8_t  reserved[64];        // MUST be zero
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

|   Type | Name                | Description               |
| -----: | ------------------- | ------------------------- |
| 0x0001 | TLV_CODEC_CONFIG    | Codec extradata           |
| 0x0002 | TLV_COLOR_INFO      | Color/HDR metadata        |
| 0x0003 | TLV_AUDIO_LAYOUT    | Channel layout            |
| 0x0004 | TLV_COMPRESSION     | Compression parameters    |
| 0x0005 | TLV_TRACK_RELATIONS | Track relationship table  |
| 0x0006 | TLV_ONTOLOGY        | Ontology ID/version       |
| 0x0007 | TLV_TEXT_ENCODING   | Text encoding             |
| 0x0008 | TLV_SCHEMA_ID       | Payload schema identifier |

---

## 5. Timing Model (Normative)

* All timestamps are microseconds relative to timeline start (t = 0)
* Variable frame rate is supported
* Media samples MUST have `duration_us > 0`
* Event/marker samples MAY have `duration_us = 0`
* AI tracks MUST set `dts = 0`

---

## 6. Track System (Normative)

### 6.1 Track Types

* `0x00–0x3F`: Media tracks
* `0x40–0xFF`: AI tracks

(Full registry in Appendix B.)

### 6.2 Track Directory (`TDIR`) — Canonical Layout

All offsets inside `TDIR` are **relative to the start of the `TDIR` payload**.

Layout:

1. `TrackDirectory`
2. `TrackDescriptor[track_count]`
3. `uint32_t string_table_size`
4. `uint8_t string_table[]`
5. `uint32_t tlv_table_size`
6. `uint8_t tlv_table[]`
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

  uint64_t data_offset;         // ABS file offset
  uint64_t data_size;
  uint64_t index_offset;        // ABS file offset
  uint64_t index_size;

  uint32_t sample_count;
  uint32_t model_ref;           // Provenance entry or 0
  uint64_t duration_us;

  uint32_t name_offset;         // REL to string table
  uint32_t name_len;

  uint32_t metadata_offset;     // REL to TLV table
  uint32_t metadata_size;
};
```

### 6.4 Track Relations

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

### 7.3 Sample Flags

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

All offsets are relative to the start of the `SIDX` payload.

```c
struct SemanticIndex {
  uint32_t version;                // 1
  uint32_t flags;
  uint32_t entity_index_offset;
  uint32_t concept_index_offset;
  uint32_t temporal_index_offset;
  uint32_t capabilities_offset;    // JSON UTF‑8
  uint32_t capabilities_size;
  uint32_t embedding_dimension;
  uint32_t entity_count;
  uint32_t concept_count;
  uint32_t reserved0;
  uint32_t reserved1;
};
```

### 9.1 Temporal Cue Table

```c
struct TemporalCueTable {
  uint32_t version;     // 1
  uint32_t cue_count;
  TemporalCue cues[cue_count];
};

struct TemporalCue {
  int64_t  timestamp;
  uint32_t event_count;
  uint32_t events_offset;
};

struct TemporalEventRef {
  uint32_t track_id;
  uint32_t sample_index;
};
```

---

## 10. Embedding Store (`EMBD`) (Normative)

```c
struct EmbeddingStore {
  uint32_t version;
  uint32_t dimension;
  uint32_t count;
  uint8_t  quantization;   // f32, f16, int8, binary
  uint8_t  index_type;     // flat, IVF, HNSW
  uint16_t reserved;

  uint64_t vectors_offset;
  uint64_t vectors_size;
  uint64_t index_offset;
  uint64_t index_size;

  uint64_t metadata_offset;
  uint32_t metadata_count;
  uint32_t metadata_entry_size;
};
```

---

## 11. Provenance Ledger (`PROV`) (Normative)

```c
struct ProvenanceLedger {
  uint32_t version;
  uint32_t entry_count;
  uint64_t model_registry_offset;
  uint64_t processing_history_offset;
  uint64_t entries_offset;
};
```

---

## 12. Security and Encryption (`ENCR`) (Normative)

```c
struct EncryptedChunk {
  uint8_t  encryption_method; // AES‑256‑GCM
  uint8_t  key_derivation;
  uint16_t reserved0;
  uint8_t  kid[16];
  uint8_t  iv[12];
  uint8_t  auth_tag[16];
  uint32_t encrypted_size;
  uint8_t  encrypted_data[encrypted_size];
};
```

---

## 13. Versioning and Compatibility (Normative)

* Version format: MAJOR.MINOR.PATCH
* Readers MUST skip unknown chunks, tracks, TLVs
* Writers SHOULD preserve backward compatibility within a major version

---

## 14. File Extension and MIME Type

* **Extension:** `.qub`
* **MIME Type:** `video/x-qub`
* **UTI (macOS):** `com.qub.container`

---

## Appendix A — FourCC Registry

(See §2.5)

## Appendix B — Track Type Registry

(Media and AI track IDs retained from v1.0.)

## Appendix C — Codec Registry

(H.264, H.265, AV1, AAC, FLAC, Opus, PCM, etc.)

## Appendix D — Optional Query Language Examples

```
FIND shots WHERE person="John" AND shot_type=CLOSE
FIND scenes WHERE description SIMILAR TO "romantic dinner" THRESHOLD 0.8
```

## Appendix E — Recommended Models and Ingest Order (Informative)

(As previously specified.)

---

*End of QUB Container Specification v1.1.0*
