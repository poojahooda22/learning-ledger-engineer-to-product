# References: Netflix scrubbing preview thumbnails (trickplay)

Saved 2026-08-17.

## Netflix engineering (video pipeline / Cosmos)
- "Rebuilding Netflix Video Processing Pipeline with Microservices" (Cosmos: Optimus API layer, Plato workflow layer, Stratum serverless compute, Timestone message bus).
  https://netflixtechblog.com/rebuilding-netflix-video-processing-pipeline-with-microservices-4e5e6310e359
- "The Making of VES: the Cosmos Microservice for Netflix Video Encoding" (Video Encoding Service, chunked/parallel encoding shape that trickplay generation mirrors).
  https://netflixtechblog.com/the-making-of-ves-the-cosmos-microservice-for-netflix-video-encoding-946b9b3cd300

## Trickplay image formats and standards
- Roku Developer docs, BIF (Base Index Frames) file format for trick mode (magic header, image count, framewise separation in ms, index table of timestamp+offset, packed JPEGs).
  https://developer.roku.com/docs/developer-program/media-playback/trick-mode/bif-file-creation.md
- AWS Elemental MediaPackage, "Working with trick-play" (I-frame playlists vs image-based trick play vs DASH thumbnail tiles; Roku/Disney/WarnerMedia pushed the image-based spec).
  https://docs.aws.amazon.com/mediapackage/latest/ug/trick-play.html
- Apple HLS authoring specification, I-frame playlists (EXT-X-I-FRAME-STREAM-INF).
  https://developer.apple.com/documentation/http-live-streaming
- Unified Streaming, "Adding trick play to a DASH or HLS stream."
  https://docs.unified-streaming.com/documentation/package/trickplay.html
- Bitmovin, "Thumbnail Generation Support for VOD Encoding" (JPEG sprite sheets + WebVTT tile mapping).
  https://developer.bitmovin.com/encoding/docs/thumbnail-generation-support-for-vod-encoding

## Notes on fact vs inference
- Confirmed public: the Cosmos platform and its layer names (Optimus/Plato/Stratum/Timestone), VES, chunked parallel encoding; the BIF format byte layout; the HLS/DASH trickplay standards and who pushed the image-based one.
- Labeled inference in the report: the exact frame-sampling interval, thumbnail dimensions, and that Netflix generates trickplay images as a parallel chunked media artifact on Cosmos/Stratum. Netflix has not published a dedicated trickplay-internals post; the report uses the well-grounded "this is how this class of problem is solved" version and marks it as such.
