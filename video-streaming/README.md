# Video Streaming

How video gets from a source (upload, or a live camera feed) to a viewer's screen at a bitrate
their connection can actually sustain, at scale. Two genuinely different problems wearing the
same name: VOD (video-on-demand, e.g. YouTube/Netflix) has minutes-to-hours to prepare content
before anyone watches it; live streaming has seconds, and the pipeline design differs
accordingly.

## Adaptive bitrate streaming (ABR)

The core technique behind both: encode the same content at several quality/bitrate levels
("renditions" — e.g. 240p/500kbps up to 4K/25Mbps), split each into short segments (2-10s), and
let the player switch between renditions segment-by-segment based on measured network
throughput and buffer health — not a single fixed quality for the whole session.

- **HLS** (HTTP Live Streaming, Apple) and **DASH** (Dynamic Adaptive Streaming over HTTP, open
  standard) are the two dominant formats — both are just plain HTTP requests for a manifest file
  (listing available renditions and segment URLs) plus the segments themselves, which is what
  makes them trivially CDN-cacheable: no special streaming server/protocol needed, any HTTP CDN
  works.
- The player's ABR algorithm is a local feedback loop: estimate current throughput (from recent
  segment download time), check current buffer level, pick the highest rendition that won't
  under-run the buffer — favoring buffer health over throughput estimate alone avoids
  oscillating between renditions on a noisy connection.

## VOD pipeline

Upload → **transcode** (produce every rendition + segment them, often via a managed service like
AWS MediaConvert / a self-hosted FFmpeg fleet — this is CPU/GPU-heavy and fully parallelizable
per segment) → store renditions + manifest in object storage (S3-class) → **CDN** serves segments
to viewers, cached at edge nodes close to them. Because there's no real-time constraint, this
pipeline optimizes for encoding efficiency (better compression per bit, more renditions, per-title
encoding ladders tuned to the actual content) over raw speed.

## Live streaming pipeline

Encoder (camera/software, e.g. OBS) → **ingest** server (RTMP is still the common
capture-to-server protocol, being real-time and low-overhead) → **live transcoder** produces ABR
renditions in near-real-time, segment by segment, as the stream arrives → **packager** emits an
HLS/DASH manifest that grows as new segments become available → CDN, same as VOD from here.
Segment length is a direct latency/efficiency trade: shorter segments (1-2s) cut end-to-end
latency but add per-segment overhead and reduce compression efficiency; **low-latency HLS/DASH**
variants (chunked transfer within a segment, so the player can start playing a segment before
it's fully encoded) push glass-to-glass latency down to a few seconds without abandoning the
CDN-friendly HTTP model. Sub-second latency (real interactivity, e.g. live auctions/gaming) needs
WebRTC instead — a genuinely different, peer-connection-based protocol, not an HLS variant.

## Staff-engineer notes

- CDN cache hit rate is usually the single biggest cost and latency lever in a streaming system —
  segment URLs and cache-control headers should be designed so that popular content is served
  almost entirely from edge caches, with origin fetches as the rare exception, not the common
  path.
- Storage cost for VOD compounds fast: N renditions × segment overhead × however many titles.
  Per-title encoding (choosing the rendition ladder based on the actual content's complexity,
  rather than one fixed ladder for everything) and re-encoding to newer, more efficient codecs
  (AV1 over H.264) over time are real, ongoing cost levers worth revisiting periodically, not a
  one-time decision.
- For live, plan explicitly for the thundering-herd case of a popular stream starting — a burst
  of concurrent viewers all requesting the manifest and first segments at once is a predictable
  spike, not a surprise, and CDN + origin capacity should be sized/pre-warmed for it rather than
  discovered during the event.
