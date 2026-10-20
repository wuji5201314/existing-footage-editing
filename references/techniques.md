# Techniques — Existing Footage Editing

Concrete recipes behind the workflow in `SKILL.md`. All lessons below were earned on the 孙宇晨访谈切片 project (2026-10) unless noted.

## 1. Transcription (single source of truth)
- Tool: faster-whisper, `medium` model, venv at `~/workspace/venv-stt/`. The user insists on accuracy over speed.
- Output: `transcript.json` (segments with start/end) + a plain-text copy. Every later step — selection, cut points, subtitles, copy — reads from this file, never from memory of the audio.

## 2. Segment selection standard
- A shippable clip = subject + premise + conclusion. Read the transcript around the quote: if the sentence before it is the premise ("AI 时代所有东西都会变得很便宜" before "年轻就是财富"), the clip starts at the premise.
- Quote clips: 12–15s floor. Theory/explainer segments: as long as the argument needs (40–55s observed).
- Anti-pattern (rejected by the user, 2026-10-17): 6–11s bare punchlines. They read as context-free slogans and had to be re-cut.

## 3. Accurate cutting
```bash
python3 ~/workspace/custom-skills/local-ffmpeg-skill/scripts/cut.py <src> --start <s> --end <e> --accurate --crf 18
```
- After cutting, watch/listen to the first and last second. End-leak is a real failure mode: a clip once shipped with the speaker opening his mouth for the next sentence ("因为你老了…") in the final half-second — trim the tail to the last complete breath.

## 4. Export matrix
| Version | Purpose | Notes |
|---|---|---|
| GIF | chat/messaging loops | silent, small |
| MP4 480×270 | easy forwarding | with sound |
| MP4 source-res (e.g. 1026×576, name suffix `_576p`) | platform upload | highest available; never upscale |

- Filenames carry the resolution suffix so the user can never grab the wrong one.
- Keep clean masters; processed variants (e.g. `_bgm`) are additional files, never overwrites.

## 5. BGM premix recipe
Why premix: on Douyin, stacking BGM in the multi-select/photo editor squeezed each clip to ~6s — speech never finished. Premixed files upload with original-sound only and play full length.

Before mixing, **verify the BGM recording's waveform in sections** (user screen-recordings have shipped with ~50s of silence at the head). Find a stable section first (the Facing The Sun recording: stable from ~63s).

```bash
ffmpeg -i <clip.mp4> -ss <bgm_offset> -i <bgm_source> -filter_complex \
"[1:a]atrim=0:<dur>,asetpts=PTS-STARTPTS,volume=0.15,afade=t=in:st=0:d=1,afade=t=out:st=<dur-1.5>:d=1.5[bgm];[0:a][bgm]amix=inputs=2:duration=first:normalize=0,alimiter=limit=0.98[a]" \
-map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k <clip_bgm.mp4>
```
- Gotcha: `amix` defaults to `normalize=1`, which ducks the voice by ~6 dB. Always `normalize=0`, then `alimiter=limit=0.98`.
- The user once asked for BGM at 75% — that buries speech. Ship 15%, say so plainly, offer to adjust after they listen.
- Verify after mixing: `ffmpeg -i <file> -af volumedetect -f null -` on clean vs mixed — mean volume should differ by ≲1 dB, max ≤ ~-0.8 dB (no clipping).

## 6. Concatenating clips into one post
```bash
printf "file '<abs path 1>'\nfile '<abs path 2>'\n" > /tmp/concat_list.txt
ffmpeg -f concat -safe 0 -i /tmp/concat_list.txt -c copy <merged.mp4>
```
- Use **absolute paths** in the list — relative entries resolve against the list file's directory, not the cwd (bit us once, 2026-10-20).
- Stream-copy concat only works when codec/resolution/audio params match (true when both clips came from the same pipeline).

## 7. Publishing kit template
Per clip, delivered together in chat (file first, then copy):
- **File**: the full-res `_576p`-style MP4 (or `_bgm` variant when premixed).
- **Title**: the quote itself, e.g. 「孙宇晨：年轻就是财富，因为我百分百相信AI」.
- **Hashtags**: 3–4, e.g. `#孙宇晨 #AI #认知 #搞钱`.
- **Pinned comment** (user posts it first): an opinion question, e.g. 「你觉得 AI 时代，年轻人最大的优势是什么？」
- **Cover**: big-text quote over the frame where the quote lands.
- **Checklist**: sound on, high-quality upload checked, original sound kept (no second BGM in-app).
- **Schedule**: one per day, evening window (19:00–21:00). Track actual vs planned in the plan file; when the user merges/skips entries, re-lay the remaining schedule instead of nagging about the old one.
- Note for the user (once, not per clip): pure re-uploads risk platform throttling; adding even a 3s original intro/commentary makes it safer long-term.

## 8. Three-dimensional analysis, sync check, hallucination filter
（来源：znyupup/ai-video-editing-skill 的"听/看/量"三维分析；cutlist 的输入校验）
Analyse every source on three axes before selecting anything:
- **Hear** — the timestamped transcript (§1).
- **See** — frames. Extract a contact sheet around candidate moments (and the first 3s of the source): `ffmpeg -i <src> -vf "fps=1,scale=270:-1,tile=6x5" -frames:v 1 sheet.jpg`. Reject candidates whose picture contradicts the words: speaker off-camera, face hidden, blank slide, loading screen, glitch frame. Note the speaker/frame focus for vertical reframing.
- **Measure** — audio energy. `ffmpeg -i <src> -af astats` / a volume scan per transcript segment; loud runs (laughs, emphasis) become **leads** to cross-check against the candidate list, never candidates on their own.

Sync check (when captions come as a separate file): last caption time vs video duration. Captions running past the video end = mismatched exports → stop, ask for a re-export from the same timeline. Video running past the last word = outro / end card / trailing silence — ask the user which, never assume an offset.

Whisper hallucination filter: on pure music / ambient stretches Whisper invents text. Any transcript segment whose audio mean volume is below ≈ −40dB is suspect — listen or drop it before it becomes a "quote" in a plan.

Long-source proxy: for sources over ~30 min, make a 360p proxy for frame review and contact sheets; all cuts and exports still come from the original file.

## 9. Scoring rubric
（来源：cutlist 的 6 维打分 + Melvynx `/clips` 的 5 维打分，合并并按本用户的语义完整标准调整）
Score each candidate 0–2 per row; keep **≥8/12**:

| Test | 2 | 1 | 0 |
|---|---|---|---|
| Self-contained | Full sense with zero context (subject + premise + conclusion) | Needs one line of setup | Needs the episode |
| Hook in 2s | First sentence is a claim, number, question or contradiction | Hook arrives by 5s | Slow start |
| One idea | Exactly one point, one payoff | Two points | Rambles |
| Payoff | Ends on a punchline, answer or instruction | Ends fine | Trails off |
| Energy | Laugh / emphasis / loud run confirms it | Neutral | Flat |
| Speaker value (2+ speakers) | The guest/expert says the important part | Shared | Host only (cap at 2 per batch) |

Single-speaker sources: skip the last row, keep ≥7/10.
Automatic rejects regardless of score: cold-start on an unintroduced "he / it / this", any lean on the long video ("as I said earlier…"), inside jokes, crosstalk, >1.5s dead air inside the clip, results promises ("you will make $X") the clip does not actually support.

## 10. Plan file template
（来源：cutlist 的 clip_plan + Melvynx `/clips` 的 repurpose plan，合并）
One plan file per source, written **before** cutting, updated as publishing actually happens:

```markdown
# <source title>
Source: <file/link> · <duration> · transcribed <date>
Inside: 4 claims, 2 stories, 6 numbers, 3 mistakes, 5 how-tos, 1 answer
Cold open: "<strongest complete sentence>" (<start>–<end>, ≤10s)

| # | Slug | Start → end | Length | Opens on (exact words) | Score |
|---|---|---|---|---|---|
| 1 | young-is-wealth | 1525.5 → 1540.7 | 15.2s | "AI时代所有东西都会变得很便宜…" | 10/12 |

### 1. young-is-wealth
**Why it stands alone:** premise + claim + conclusion in one go.
**Speaker focus:** guest, centre frame.
**Kit:** title / hashtags / pinned comment / cover line (see §7, §12).
**Status:** planned → cut → delivered → posted <date> (mirror in the published log)
```

Publish order: strongest first, then alternate topics. The cold-open moment may double as clip 1. Ideas too spread out to clip get one line each under `## Re-record candidates` — they are not clips and are never forced into the table.

## 11. Word-snapped cutting, truncation validation, clean joins
（来源：video-use 的词边界落刀 + 30ms 淡化 + 切点自检；znyupup 的语音截断校验；shorts-cutter 的停顿压缩）
1. Rough-cut with ~0.5s margin on both sides (caption/transcript times are approximate), then snap in/out points to the **quietest point nearest the word boundary** — never mid-syllable.
2. **Truncation validation (programmatic, not eyeballing):** for each clip, check every transcript segment overlapping [start, end]: if a segment starts < start − 0.3s or ends > end + 0.5s, the cut truncates speech. Auto-fix: start → segment start − 0.1s; end → segment end + 0.2s. Then **re-run the check**; a clip ships only with a clean second pass. (Timestamp error is ±0.2–0.3s — that is what the tolerance is for.)
3. **30ms audio fades** at every cut head and tail (`afade=t=in:st=0:d=0.03`, `afade=t=out:st=<dur-0.03>:d=0.03`) so joins never click or pop.
4. Optional tightening for talking-head cuts only: pauses >0.6s compress to ≈0.3s. Preserve laughs and deliberate beats — never tighten theory/explainer segments this way.
5. Self-evaluate the rendered file at **every cut boundary**: play the first and last second of each clip and each join in a concatenated export before delivering.

## 12. Cover trifecta and legibility test
（来源：cutlist 的 thumbnail-trifecta + Melvynx `/publish` 的封面选帧）
- **Trifecta:** cover text (2–4 big words from the quote) + title + first description line must all restate the **cold open's** promise. Test by reading them in sequence — same promise three times, or fix the weakest one.
- **Frame picking:** contact sheet of the first ~3s (`fps=10, scale=270:-1, tile=6x5`), choose the frame where the speaker who says the line is on camera, eyes open, face expressive (mid-word fine, blink not), subject on a left/right third, text on the opposite side, nothing ugly in frame (hands over face, overlays, glitches). Never default to frame 0.
- **Legibility test:** render the cover, shrink to ≈320px wide (feed size) and read it. If the words do not read, **cut words** — never shrink the font to fit more in.
- Keep text away from platform UI zones (bottom area on Douyin/TikTok, bottom-right duration badge on YouTube).

## 13. Sources — what was taken, what was rejected (2026-10-21)
Taken:
- **itsryanlenk/cutlist** — plan-first workflow, input sync check, 6-point scoring, audio-energy leads, contact-sheet frame check, thumbnail trifecta, publish-order rule.
- **Melvynx/video-agent-skills** — `/clips`: content inventory taxonomy, cold-open pick, 0.5s cut margins, "too spread out → new script" outlet; `/publish`: cover-from-contact-sheet, published-log, "already posted → stop" duplicate check.
- **browser-use/video-use** — filler/dead-space awareness, word-boundary cuts, 30ms fades, colour-neutral clean output, self-evaluation at cut boundaries, `project.md`-style session memory.
- **znyupup/ai-video-editing-skill** — hear/see/measure three-dimensional analysis, speech-truncation validation + auto-fix + second pass, −40dB hallucination filter, BGM 10–15% under speech (matches the established 15% here).
- **eliftforlife/claude-video-editing-skills (shorts-cutter)** — pause compression (>0.6s → 0.3s), end-on-payoff, one-focal-point caption restraint.
- **luoluoluo22/jianying-editor-skill** — mandatory acceptance checklist before declaring done, low-res proxy for long-video analysis, markers/plan handoff for editor fine-cutting, keep working scripts out of the skill directory.

Rejected, deliberately:
- **Auto-publishing / scheduling tools** (Melvynx's Treg route): the user posts manually in the Douyin app; nothing posts without their tap.
- **Cloud/library BGM recommendation** (jianying-editor-skill): violates the standing rule — BGM only from audio the user provided.
- **Plan-only, never-edit stance** (cutlist): this workflow cuts for real via ffmpeg-skill; the plan is a step, not the product.
- **Vertical-first reframing as default** (most of the above): this user's interview clips publish at source ratio on Douyin; reframe only when a platform or the user asks, with speaker focus noted.
- **Filler-word removal as a default pass** (video-use): right for vlog takes, wrong for interview answers where pauses carry meaning; kept as opt-in tightening (§11.4) only.
