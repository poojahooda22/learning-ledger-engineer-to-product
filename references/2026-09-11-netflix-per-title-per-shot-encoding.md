# References: Netflix per-title and per-shot encoding (2026-09-11)

Keeper links for the encoding-optimization teardown. Primary Netflix sources first.

## Primary (Netflix)

- Per-Title Encode Optimization (2015), Anne Aaron, Jan De Cock, David Ronca et al. The origin: replace the fixed ladder with a content-aware per-title ladder built from a convex hull of test encodes. https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2
- Dynamic Optimizer, a perceptual video encoding optimization framework (March 2018). Per-shot convex hulls stitched by the constant-slope trellis, VMAF as the objective. https://netflixtechblog.com/dynamic-optimizer-a-perceptual-video-encoding-optimization-framework-e19f1e3a277f
- Optimized shot-based encodes: Now Streaming! (2018). Production deployment of the per-shot method. https://medium.com/netflix-techblog/optimized-shot-based-encodes-now-streaming-4b9464204830
- Toward A Practical Perceptual Video Quality Metric (VMAF, 2016). How VMAF fuses VIF + DLM/ADM + motion with an SVM regressor trained on subjective scores. https://netflixtechblog.com/toward-a-practical-perceptual-video-quality-metric-653f208b9652
- Netflix/vmaf, the open-source VMAF library (libvmaf, FFmpeg filter). https://github.com/Netflix/vmaf

## Papers

- I. Katsavounidis, L. Guo, "Video codec comparison using the dynamic optimizer framework," SPIE 2018. Reports the ~28% (x264), ~34% (x265), ~38% (VP9) BD-rate savings at equal VMAF and the within-1%-of-optimal claim. https://www.spiedigitallibrary.org/conference-proceedings-of-spie/10752/107520Q/Video-codec-comparison-using-the-dynamic-optimizer-framework/10.1117/12.2322118.short

## Secondary explainers

- Jan Ozer, "How Netflix Pioneered Per-Title Video Encoding Optimization" (Streaming Media / Streaming Learning Center). Reproduces the old fixed ladder numbers (235 kbps @ 320x240 up to 5,800 kbps @ 1080p) and the cartoon-vs-noisy-content argument. https://streaminglearningcenter.com/encoding/how-netflix-pioneered-per-title-video-encoding-optimization.html
- Fora Soft Learn, "Building a Bitrate Ladder: Classic Netflix Ladder, Per-Title, Per-Shot." https://www.forasoft.com/learn/video-streaming/articles-streaming/bitrate-ladder-per-title-per-shot
- Hackaday, "Decoding The Netflix Announcement: Explaining Optimized Shot-Based Encoding For 4K" (2020). https://hackaday.com/2020/09/16/decoding-the-netflix-announcement-explaining-optimized-shot-based-encoding-for-4k/
- Wikipedia, "Video Multimethod Assessment Fusion." https://en.wikipedia.org/wiki/Video_Multimethod_Assessment_Fusion

## One-line recall

Fixed ladder wastes bits on easy content and starves hard content, so build a per-title (then per-shot) ladder by convex hull of test encodes, choose operating points across shots with a constant-slope (Lagrangian) trellis under one budget, optimize toward VMAF not PSNR, keep it all offline and parallel, and the third-of-bytes saving is collected on every one of billions of streams while the live path stays a dumb rung-pick.
