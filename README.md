# QUB — Information Cubed

**A new class of AI-forward video container for query-native editing workflows.**

QUB stores AI-derived understanding (shots, scenes, objects, dialogue, OCR, segmentation, embeddings) **inside the media container itself**, alongside traditional audio/video streams—so AI analysis is performed once at ingest and travels with the file forever.

---

## The Problem

Every time video moves between systems, AI analysis starts over. A clip transcribed during ingest gets re-transcribed in your NLE. Shot detection runs again. Object and face detection reruns. Scene understanding gets recomputed—or is lost entirely.

This is wasteful, slow, and blocks the intelligent editorial workflows modern AI makes possible.

---

## The Solution

QUB wraps standard codecs (H.264, HEVC, ProRes, AAC, etc.) in a container that also carries **first-class semantic tracks**:

| Track Type | What It Stores |
|------------|----------------|
| **Vision Semantic** | Natural language scene descriptions from VLMs |
| **Shot Boundaries** | Cut points, transition types, shot classification + descriptors |
| **Scene Segments** | Scene grouping across shots |
| **Object Detection** | Frame-by-frame detection with persistent tracking IDs + bounds |
| **Face Detection** | Faces with identity clustering (privacy-aware) |
| **OCR** | On-screen text regions with polygons, language, and type |
| **Action Recognition** | Action and event segments |
| **Transcription** | Speech-to-text with word-level timestamps |
| **Speaker Diarization** | Who spoke when |
| **Audio Classification** | Music, ambient, effects, dialogue |
| **Depth Maps** | Monocular depth estimation per frame |
| **Semantic Segmentation** | Pixel-level class segmentation |
| **Instance Segmentation** | Per-object masks with tracking |
| **Panoptic Segmentation** | Combined semantic + instance segmentation |
| **Alpha Mattes** | Foreground/background separation |
| **Saliency** | Visual attention maps |
| **Embedding Store** | Dense vectors for similarity search |
| **Semantic Index** | Queryable entity, concept, and temporal index |

Standard players see a normal media file and ignore unknown tracks. AI-aware tools get the full semantic payload.

---

## Design Principles

1. **Inference Once, Query Forever** — AI preprocessing happens at ingest; results persist with the media
2. **Codec Agnostic** — Any modern video/audio codec, unchanged
3. **Model Versioned** — Full provenance (model ID, version, parameters, checksums)
4. **Temporally Dense** — Microsecond-accurate alignment across all metadata
5. **Query Native** — Semantic index and embeddings enable semantic queries
6. **Graceful Degradation** — Unknown tracks are skipped, not fatal
7. **Extensible** — New AI modalities and vendor extensions without breaking parsers

---

## Quick Example (Hypothetical CLI)

```bash
# Reference implementation TBD

# Preprocess video, generating AI tracks
qub ingest input.mov -o output.qub \
  --vision-model FastVLM \
  --transcription-model whisper-large-v3 \
  --segmentation-model sam2

# Query the container
qub query output.qub "shots where someone mentions 'deadline'"
qub query output.qub "close-ups of the interview subject"
qub query output.qub "frames where text contains 'CONFIDENTIAL'"

# Export for NLE interchange
qub export output.qub --format fcpxml -o project.fcpxml
qub export output.qub --format otio   -o timeline.otio
qub export output.qub --format aaf    -o sequence.aaf
```

---

## Specification

The full technical specification is in [SPECIFICATION.md](SPECIFICATION.md).

It defines:

- Container architecture and canonical chunk layout
- Track directory (TDIR), sample framing, and indexing
- Full AI track schemas (shots, OCR, detection, transcription, segmentation, etc.)
- Semantic index and embedding store
- Provenance ledger for model auditability
- Optional query capabilities contract and reference query language
- Ingest pipeline guidance ("Inference Once, Query Forever")
- NLE integration patterns (Avid, Final Cut, Resolve, Premiere)
- Security, privacy, and redaction rules
- Versioning and compatibility guarantees

---

## Status

**Draft Specification v1.1.0** — First Complete Semantic Media Container Draft

Seeking feedback from:

- Video engineers and NLE developers
- ML engineers working on video understanding
- Professional editors and post-production facilities

There is not yet a reference implementation.

The spec is published to define the architecture and invite collaboration.

---

## NLE Integration Targets (Informative)

| NLE | Integration Approach |
|-----|----------------------|
| **Avid Media Composer** | AAF with metadata, bin columns, ScriptSync/PhraseFind replacement |
| **Final Cut Pro** | FCPXML with keywords, roles from diarization, semantic search |
| **DaVinci Resolve** | OTIO, smart bins, shot/scene detection bypass |
| **Premiere Pro** | Panel extension, text-based editing replacement |

---

## Why "QUB"?

Information cubed — three dimensions of data:

- **Media** — The pixels and waveforms
- **Understanding** — What the AI sees and hears
- **Structure** — How it connects temporally and semantically

Also: it's short, memorable, and `.qub` is available.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

Areas where input is especially valuable:

- Missing or mis-scoped AI track schemas
- Editorial primitives that matter in real workflows
- Semantic indexing semantics (entities, concepts, time)
- NLE integration details
- Security and privacy defaults

---

## License

Apache License 2.0. See [LICENSE](LICENSE).

The spec is open. Build on it, fork it, implement it, extend it.

---

## Author

**Houman Shekarchi**

Video editor with 20+ years of professional experience, now building AI-powered tools for media workflows.

This project emerged from hands-on work developing a vision-based assistant editor using FastVLM and Whisper on Apple Silicon, where it became clear that AI results do not travel with media—and that this needed to change.

---

## Contact

- **Issues & Discussion:** [GitHub Issues](../../issues)
- **Pull Requests:** [GitHub PRs](../../pulls)
