# QUB — Information Cubed

**A next-generation video container format for AI-native editing workflows.**

QUB stores AI preprocessing results (vision, audio, segmentation) alongside traditional media streams, eliminating redundant inference and enabling semantic search across video content.

---

## The Problem

Every time video moves between systems, AI analysis starts over. A clip transcribed during ingest gets re-transcribed in your NLE. Shot detection runs again. Scene understanding is lost entirely.

This is wasteful, slow, and prevents the kind of intelligent editing workflows that modern AI makes possible.

## The Solution

QUB wraps standard codecs (H.264, H.265, ProRes, AAC, etc.) in a container that also carries:

| Track Type | What It Stores |
|------------|----------------|
| **Vision Semantic** | Natural language scene descriptions from VLMs |
| **Shot Boundaries** | Cut points, transition types, shot classification |
| **Object Detection** | Frame-by-frame detection with persistent tracking IDs |
| **Face Detection** | Faces with identity clustering (privacy-respecting) |
| **Transcription** | Speech-to-text with word-level timestamps |
| **Speaker Diarization** | Who spoke when |
| **Audio Classification** | Music, ambient, effects, dialogue |
| **Depth Maps** | Monocular depth estimation per frame |
| **Instance Segmentation** | Per-object masks with tracking |
| **Alpha Mattes** | Foreground/background separation |
| **Semantic Index** | Queryable entity/concept index with embeddings |

Standard players see a normal video file. AI-aware tools get the full semantic payload.

## Design Principles

1. **Inference Once, Query Forever** — Preprocessing happens at ingest; results travel with the media
2. **Codec Agnostic** — Any modern video/audio codec, unchanged
3. **Model Versioned** — Full provenance (model ID, version, parameters) for all AI data
4. **Temporally Dense** — Microsecond-accurate alignment for all metadata
5. **Query Native** — Built-in semantic index for natural language queries
6. **Graceful Degradation** — Unknown tracks are skipped, not fatal
7. **Extensible** — New AI modalities slot in without breaking parsers

## Quick Example

```
# Hypothetical CLI (reference implementation TBD)

# Preprocess video, generating AI tracks
qub ingest input.mov -o output.qub \
    --vision-model FastVLM \
    --transcription-model whisper-large-v3 \
    --segmentation-model sam2

# Query the container
qub query output.qub "shots where someone mentions 'deadline'"
qub query output.qub "close-ups of the interview subject"

# Export for NLE
qub export output.qub --format fcpxml -o project.fcpxml
qub export output.qub --format otio -o timeline.otio
qub export output.qub --format aaf -o sequence.aaf
```

## Specification

The full technical specification is in [SPECIFICATION.md](SPECIFICATION.md).

It covers:
- Container architecture and chunk format
- All track type definitions with data structures
- Semantic index and embedding store
- Provenance ledger for AI audit trails
- NLE integration patterns (Avid, Final Cut, Resolve, Premiere)
- Security and privacy considerations
- Versioning and compatibility guarantees

## Status

**Draft Specification v1.0** — Seeking feedback from:
- Video engineers and NLE developers
- ML engineers working on video understanding
- Professional editors and post-production facilities

This is not yet a reference implementation. The spec is published to establish the architecture and invite collaboration.

## NLE Integration Targets

| NLE | Integration Approach |
|-----|---------------------|
| **Avid Media Composer** | AAF with metadata, bin columns, ScriptSync/PhraseFind replacement |
| **Final Cut Pro** | FCPXML with keywords, roles from diarization, semantic search |
| **DaVinci Resolve** | OTIO, smart bins, scene cut detection bypass |
| **Premiere Pro** | Panel extension, markers, Essential Sound metadata |

## Why "QUB"?

Information cubed. Three dimensions of data:
- **Media** — The pixels and waveforms
- **Understanding** — What the AI sees and hears
- **Structure** — How it all connects temporally and semantically

Also: it's short, memorable, and `.qub` is available.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to participate.

Key areas where input is needed:
- Track type definitions (are we missing critical AI outputs?)
- NLE integration details (what does your workflow actually need?)
- Codec support (edge cases, professional formats)
- Security model (encryption, privacy levels, redaction)

## License

Apache License 2.0. See [LICENSE](LICENSE).

The spec is open. Build on it, fork it, implement it, extend it. The Apache 2.0 license includes patent protections for contributors and users.

## Author

**Houman Shekarchi**

Video editor with 20+ years of professional experience, now building AI-powered tools for media workflows.

This spec emerged from hands-on work developing a vision-based assistant editor using FastVLM and Whisper on Apple Silicon. The problem of "AI results don't travel with media" became obvious enough that solving it seemed necessary.

## Contact

- **Issues & Discussion:** [GitHub Issues](../../issues)
- **Pull Requests:** [GitHub PRs](../../pulls)

---

*"The best time to define a new container format was when AI video understanding became practical. The second best time is now."*
