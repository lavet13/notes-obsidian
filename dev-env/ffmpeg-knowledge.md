---
id: ffmpeg-knowledge
aliases: []
tags: []
---

## Inspect a file first (ffprobe)

    # Human-readable stream summary (codecs, resolution, fps, bitrate, duration)
    ffprobe -hide_banner -i input.mkv
    #   -hide_banner  suppress the build/config dump
    #   -i            input file

    # Full machine-readable dump (everything, as JSON)
    ffprobe -hide_banner -v quiet -print_format json -show_format -show_streams input.mkv
    #   -v quiet          silence logs so only the JSON prints
    #   -print_format json output format (also: flat, csv, xml)
    #   -show_format      container-level info (size, duration, overall bitrate)
    #   -show_streams     per-stream info (one block per video/audio/subtitle track)

## Size math — the whole game

    file_size(bytes) ≈ total_bitrate(bits/s) × duration(s) / 8
    # To shrink a file you cut BITRATE. Duration is fixed.
    # total_bitrate = video_bitrate + audio_bitrate.
    # Example: 6055 kb/s × 1746 s / 8 ≈ 1.32 GB.

## Two ways to encode: quality target vs size target

CRF = "make it look this good, size falls where it falls." Use when you just
want smaller without hurting quality. ONE pass, fast to set up.

    ffmpeg -i input.mp4 \
      -c:v libx264 -crf 21 -preset slow -tune film \
      -c:a copy \
      output.mp4
    #   -crf 21     Constant Rate Factor, 0=lossless … 51=worst. ~18 near-lossless,
    #               21–23 great, 25+ visibly softer. LOWER = bigger+better.
    #   -preset slow slower = smaller file at same CRF (more encoding effort).
    #               ladder: ultrafast … veryfast … medium … slow … veryslow
    #   -tune film  preserve fine detail/grain (use for live/game footage).
    #               NOT -tune animation unless it's actual 2D cartoon line-art.
    #   -c:a copy   pass audio through untouched — no re-encode, no quality loss.

Two-pass = "hit THIS size precisely." Use for a hard ceiling (upload limits).
Runs twice: pass 1 analyzes → log, pass 2 encodes to the log.

    # target size → bitrate:  video_kbps ≈ target_MB × 8192 / duration_s − audio_kbps
    # e.g. 500 MB over 1746 s ≈ 2290 total − 192 audio ≈ 2000k video (leave margin)

    ffmpeg -y -i input.mp4 -c:v libx264 -b:v 2000k -preset veryfast -pass 1 -an -f null /dev/null
    ffmpeg    -i input.mp4 -c:v libx264 -b:v 2000k -preset veryfast -pass 2 -c:a copy output.mp4
    #   -b:v 2000k  target video bitrate. Size scales LINEARLY: 1500k→~360MB, 2500k→~590MB.
    #   -pass 1/2   two-pass; pass 1 writes ffmpeg2pass-0.log, pass 2 reads it. Same folder, in order.
    #   -an         pass 1 skips audio (doesn't affect bit distribution).
    #   -f null /dev/null   pass 1 discards the encode, keeps only the stats log. (Windows: NUL)
    # Gotcha: delete ffmpeg2pass-0.log afterward. Bitrate control ≠ CRF — don't mix -b:v and -crf.

## Burn-in (hardcode) subtitles

    ffmpeg -i input.mkv -vf "subtitles='path/to/subs.ass'" \
      -c:v libx264 -crf 18 -c:a aac -b:a 192k output.mp4
    #   -vf         video filter chain
    #   subtitles=  the libass burn-in filter; renders the .ass ONTO the pixels
    # Burn-in ALWAYS re-encodes video — you can never use -c:v copy with it.

    # Chain two tracks (dialogue + signs) — comma-separated, no spaces around comma:
    -vf "subtitles='subs/dialogue.ass',subtitles='subs/signs.ass'"

Escaping gotchas (these cost real time):
- Wrap the whole path in SINGLE quotes inside the double-quoted -vf value.
  That handles spaces AND square brackets: -vf "subtitles='a b [1080p].ass'"
- ONLY if the value is unquoted do brackets/spaces break the filtergraph parser
  ("Trailing garbage after a filter") and need backslash-escaping. Quoting avoids all that.
- Zero-headache fallback for gnarly names: copy the sub to a plain name first.
    cp "…[BudLightSubs]… .ass" sub.ass ; ffmpeg … -vf subtitles=sub.ass …

Russian sub-track folders: `Надписи` = signs/typesetting ONLY (episode titles,
on-screen text) — NOT dialogue. Full dialogue is the larger .ass sitting directly
in the release folder (e.g. Crunchyroll/ep.ass, not Crunchyroll/Надписи/ep.ass).

## Limit CPU usage (keep the machine usable during a long encode)

    nice -n 19 ffmpeg …                    # lowest scheduler priority; full speed when idle,
                                           # yields instantly under contention. Simplest option.
                                           # niceness −20 (greediest) … +19 (nicest); <0 needs root.
    taskset -c 0-7 ffmpeg …                # hard-pin to CPU cores 0–7 (OS-level, nothing escapes it)
    ffmpeg … -threads 8                    # cap the x264 encoder's threads (output option)
    ffmpeg … -filter_threads 2 -vf …       # cap FILTER threads (e.g. libass); global, before -vf

    # Combined belt-and-suspenders:
    nice -n 19 taskset -c 0-7 ffmpeg -i in.mkv -filter_threads 2 -vf subtitles=sub.ass \
      -c:v libx264 -crf 18 -preset medium -threads 8 -c:a aac -b:a 192k out.mp4
    # For a background batch, `nice -n 19` alone is usually enough — scheduler handles the rest.
    # x264 thread-scaling flattens past ~8 threads, so reserving a few costs little real speed.

## Hardware vs software encoding

- libx264 (CPU) gives the best quality-per-bit; use it for archival/anime line-art.
- GPU encoders trade quality for speed: h264_nvenc (NVIDIA), h264_amf (AMD), h264_qsv (Intel).
- Ryzen 5600G = APU, NO dGPU. h264_nvenc will NOT work here. h264_amf (Vega VCE) exists
  but looks worse than x264 at equal bitrate — stay on CPU x264 and manage load with nice/taskset.
