YT Lecture RAG
Retrieval over a YouTube DSA playlist. A student asks a question in Hinglish or English and gets an answer plus clickable links that jump to the exact second in the exact lecture.

"bhai memoization aur tabulation ka difference kahan explain kiya hai?"

→ Answer: 3-4 lines, grounded only in what was actually said
→ Sources:
   [1] DP Lecture 3 — Memoization Deep Dive        @ 12:04   ▸ jump
   [2] DP Lecture 5 — Tabulation Recipe            @ 04:31   ▸ jump
Every generic RAG demo returns a blob of text. This returns a place in a video.

Stack
Layer	Choice
Playlist + audio	yt-dlp (Python API)
Transcription	faster-whisper (CTranslate2), large-v3, batched
Chunking	hand-written, time-window based
Embeddings	sentence-transformers + BAAI/bge-m3 (1024-dim)
Vector store	Qdrant Cloud — same as day14 / day15
LLM	Groq openai/gpt-oss-120b — same as day15 / hiremeai
CLI	typer + rich
API + UI	FastAPI + a single HTML page with the YouTube IFrame API
Where this differs from RAGPLAN.md
The plan was written for an Apple Silicon machine with a Gemini key. Three swaps, all deliberate:

faster-whisper, not mlx-whisper. mlx is Apple-Silicon-only. On this box the CUDA path is the faster one anyway — and batched, it is 3x faster than the sequential config the plan assumed, at no cost in quality.
No ffmpeg, no WAV files. faster-whisper decodes audio itself through PyAV, so yt-dlp downloads the raw m4a and Whisper reads it directly. That removes an install, and it removes the 16kHz WAVs that run ~115 MB per hour of video.
Qdrant + Groq, not Chroma + Gemini. Matches the rest of week 3, and the keys are already in the repo-root .env. Qdrant Cloud also makes Phase 4 deploy trivial — the index is already remote, so there is no disk volume to persist.
Setup
cd week3/ytscraper
uv venv --python 3.11
uv sync
The CUDA runtime libs that ctranslate2 needs on Windows are an optional extra:

uv sync --extra cuda
If your GPU is newer than the shipped ctranslate2 build, transcribe.py catches the failure and falls back to CPU int8 with a printed warning. Slower, still correct.

Embeddings run on CPU by default. uv pulls the CPU build of torch from PyPI, so bge-m3 embeds on the CPU — fine for querying, and a few minutes rather than seconds when reindexing a full playlist. To put it on the GPU (~3GB download):

uv pip install torch --index-url https://download.pytorch.org/whl/cu124 --force-reinstall
Transcription is unaffected either way: ctranslate2 ships its own CUDA support and already uses the GPU.

Keys come from the repo-root .env (already present):

GROQ_API_KEY=...
QDRANT_URL=...
QDRANT_API_KEY=...
Runtime data
Lives outside the repo, in ~/.ytrag/:

~/.ytrag/
├── audio/        # <video_id>.m4a — DELETABLE, regenerable in seconds
├── transcripts/  # <video_id>.json — PRECIOUS, hours of GPU time
└── langtest/     # output of `ytrag langtest`
That split is the whole point. Transcripts are the expensive artifact; everything downstream of them is cheap to rebuild, which is why changing your mind about the embedding model or the chunk size later costs minutes instead of hours.

Usage
0a. Preflight
uv run ytrag preflight --playlist "<PLAYLIST_URL>"
Checks keys, Qdrant, the embedder, the Whisper model, the transcription code path, playlist listing and the LLM — about a minute. Each check calls the thing rather than asserting it looks importable, because this exists after an 8-hour run died ten seconds in on a missing import. Run it before any long ingest.

0b. Decide the language flag first
uv run ytrag langtest "https://www.youtube.com/watch?v=<one_lecture>"
The lectures are Hinglish. language="hi" gives Devanagari output; language="en" gives romanised/translated output. Student queries will be romanised Hinglish or English, so these two produce very different retrieval behaviour.

Read a few minutes of each and check what happens to the technical terms specifically — memoization, adjacency list, time complexity, subproblem, DP table. Whichever keeps those clean, wins. Then set YTRAG_WHISPER_LANG and move on. This decision gets baked into hours of compute afterwards, so make it on evidence.

1. Ingest
uv run ytrag ingest --playlist "<PLAYLIST_URL>" --limit 1     # Phase 0: one video, end to end
uv run ytrag ingest --playlist "<PLAYLIST_URL>"               # Phase 1: the whole thing
Per video: download audio → transcribe (cached) → delete audio → chunk → upsert. One failing video logs and the run continues. Re-running re-transcribes nothing and duplicates nothing.

Speed, measured on an RTX 5050 laptop
Transcription is the only expensive step, and it is paid exactly once — chunking, embedding and indexing take seconds, and everything downstream rebuilds from cached transcripts.

config	speed	68.7-hour playlist
large-v3 sequential, beam 5 (as RAGPLAN specified)	2.5x realtime	~27 h
large-v3 batch 8, beam 5 — the default here	8.1x	~8.5 h
large-v3 batch 8, beam 1	10.5x	~6.5 h
Batching is not a quality tradeoff: same model, same weights, identical technical-term capture in testing. The sequential path simply leaves the GPU idle between speech regions. Beam 1 is a real further gain but uses greedy decoding, which is likeliest to slip on exactly the thing that matters here — an unusual technical term in accented speech — so the default stays at beam 5.

Surviving a long unattended run
Everything is built to be re-run. The command never changes:

uv run ytrag ingest --playlist "<PLAYLIST_URL>"
Power cut. Transcripts are written to a temp file and atomically renamed, so you get a complete file or none — never a truncated one the cache then trusts forever. Re-run and every finished video is skipped; you lose only the one in flight.
Network drop. Downloads and Qdrant upserts retry four times with exponential backoff, so a brief blip costs seconds. Both operations are idempotent, so a retry after a partial success is harmless.
Network death. After --stop-after-failures videos fail back to back (default 5) the run aborts with a clear message instead of ripping through the rest of the playlist failing instantly and reporting a "finished" run that did nothing.
Ctrl-C. Caught explicitly; cached work is safe.
Whisper and the embedding model are cached on disk after first use, so transcription itself needs no network at all.
Don't let the machine sleep mid-run — power loss is survivable, suspending the GPU mid-inference is messier.

2. Ask
uv run ytrag ask "memoization aur tabulation ka difference kya hai?"
uv run ytrag search "adjacency list"      # retrieval only, shows raw distances
3. Evaluate
uv run ytrag eval --verbose
4. Web UI
uv run ytrag serve
Then http://127.0.0.1:8000. Citations call player.seekTo() on the embedded player, so clicking [2] jumps inside the page rather than opening a tab.

Other commands
uv run ytrag preflight            # check every dependency before a long run
uv run ytrag reindex              # re-chunk + re-embed from cached transcripts
uv run ytrag reindex --replace    # same, but clear old chunks first (needed if chunk size changed)
uv run ytrag stats
uv run ytrag clean-audio
