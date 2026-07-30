# References: Canva Background Remover (2026-07-30)

## Models and architecture (primary)
- U2-Net: Going Deeper with Nested U-Structure for Salient Object Detection (Qin et al., Pattern Recognition 2020) PDF: https://xuebinqin.github.io/U2Net_PR_2020.pdf
- U2-Net arXiv: https://arxiv.org/abs/2005.09007
- MODNet: Trimap-Free Portrait Matting in Real Time (Ke et al., 2020): https://arxiv.org/abs/2011.11961
- Deep Image Matting: A Comprehensive Survey (2023): https://arxiv.org/pdf/2304.04672

## Engineering / scaling (primary)
- Cloudflare Blog, Evaluating image segmentation models for background removal (BiRefNet, ~351ms inference numbers): https://blog.cloudflare.com/background-removal/
- RemBG Blog, Why Background Removal is Harder to Scale Than Generative AI Models (fixed-shape inference, TensorRT/CUDA Graphs plan swaps): https://www.rembg.com/en/blog/scaling-background-removal-vs-generative-ai
- Canva Engineering Blog, From Zero to 50 Million Uploads per Day: Scaling Media at Canva: https://www.canva.dev/blog/engineering/from-zero-to-50-million-uploads-per-day-scaling-media-at-canva/

## Open source
- rembg (U2-Net based one-click background removal): https://github.com/danielgatis/rembg

## Product / acquisition / scale
- TechCrunch, Canva acquires background removal specialists Kaleido (Feb 2021): https://techcrunch.com/2021/02/24/canva-acquires-background-removal-specialists-kaleido/
- Tech.eu, Canva acquires Kaleido AI and Smartmockups: https://tech.eu/2021/02/22/canva-kaleido-smartmockups/
- remove.bg (Kaleido) product site: https://www.remove.bg/
- Music Ally, Canva now has 265m monthly active users, 31m paying (Feb 2026): https://musically.com/2026/02/19/canva-now-has-265m-monthly-active-users-and-31m-are-paying/
- Canva Background Remover feature page: https://www.canva.com/features/background-remover/

## Key numbers to remember
- Matting equation: I = alpha*F + (1-alpha)*B. Solve for alpha per pixel.
- Trimap = 3 regions (pure fg / pure bg / unknown band). Trimap-free = neural net predicts it. That is the mass-market unlock.
- U2-Net full: 176.3 MB, ~30 FPS on GTX 1080Ti. Small U2-Netp: 4.7 MB, ~40 FPS.
- BiRefNet: ~351 ms average inference on larger GPUs (~2.4x faster on bigger hardware).
- Kaleido/remove.bg acquisition: Feb 2021, high two-digit millions approaching ~$100M.
- Canva scale (early 2026): 265M MAU, 31M paying, ~$4B ARR, ~50M media uploads/day, 1B+ designs/mo, 800M AI tool uses/mo.
- Scale-out spine: queue for burst fairness, batch within resolution buckets (512/1024/2048), keep compiled fast path fixed-shape, tile big images with Gaussian-weighted overlap, cache deterministic cutout by media-id+model-version.
