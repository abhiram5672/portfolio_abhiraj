Scroll-driven image-sequence portfolio

See DEPLOY-VERCEL.txt for deployment instructions.
For local testing, run in this folder: python3 -m http.server 8000
then open http://localhost:8000

index.html        main page
case-study.html   project case-study viewer
vercel.json       static asset caching rules
assets/           portfolio imagery and case-study assets
frames-hd/        192 WebP frames at 1920x1080 (used by the player)
frames-4k/        192 WebP frames at 3840x2160 (retained, not requested by current player)

The intro is a scroll-driven canvas animation, not an MP4 video. The player has
been adjusted to use the 1080p sequence, keep a smaller decoded-frame cache,
load only nearby frames, and avoid prefetching the full sequence.
