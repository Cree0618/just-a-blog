---
date: 2026-08-19
title: "A local deployment system for speech, not a model"
description: "English YouTube captions to Czech on an i5: Argos, Piper Jirka, a service worker, hardware that refuses OmniVoice, and a GPU path documented but not wired."
tags:
  - local ML
  - Chrome extension
  - TTS
  - product
---

On-device speech systems are usually described as a model problem: pick a translator, pick a TTS, measure real-time factor. That description misses the layer that actually fails in a living room. The model is Piper’s Czech male “Jirka,” a ~60 MB ONNX file. The product is a deployment system around it: captions in, ducked soundtrack out, nothing uploaded, on a few-year-old Windows i5 with 12–16 GB of RAM sitting next to Chrome.

The product is called Dabing. I develop it on a Mac. The Mac is irrelevant. `CONTEXT.md` says a Mac is a nicer development box, not the machine that has to work in the living room. If Dabing only speaks Czech on my desk, it is a demo.

This post focuses on that deployment system: the Chrome local-network rule, hardware capability gating, and a worker that stays ahead of the playhead. ASR, speaker cloning, and a rented GPU path are related and out of scope because they are not what he runs. OmniVoice exists in the repo as an optional engine. Modal is a note in `docs/modal.md`. Neither is the living-room path.

I explicitly mention “deployment system” because the layer between captions and the ear is as important as the checkpoint. A harness here is small: merge, translate, speak, mix. It still decides how the model observes, acts, and stops.

## The path that has to survive the living room

```
YouTube English captions
        ↓
merge cues into 8–20 s windows
        ↓
Argos English → Czech
        ↓
Piper, Czech male “Jirka”
        ↓
WAV chunks over a ducked soundtrack
```

Nothing in that chain needs a GPU. Nothing uploads the video. The original soundtrack stays in the YouTube tab and gets turned down, not replaced in a file. A fixed Czech male voice is the point. You always know who is talking. You do not get a legal and moral mess for a family install.

Three tempting features are missing on purpose.

1. **No speech recognition.** If YouTube has no English captions, Dabing stops. The injector reports `no_english_captions`. The server turns an empty cue list into a job with `error="no_usable_captions"`. ASR on that laptop, live, next to Chrome, would be a second project.
2. **No live OmniVoice on his machine.** OmniVoice wants NVIDIA CUDA or Apple MPS and more RAM than he has if you also want to stay ten to thirty seconds ahead of the playhead.
3. **No rented GPU in the path he uses.** `docs/modal.md` exists so I would not re-derive the fallback later; nobody has implemented it yet. The README line is “Modal GPU (documented, not wired).”

## Design patterns

### Pattern 1: The page is public; the server is localhost

Dabing is a Chrome Manifest V3 extension and a local Python server on `127.0.0.1:8765`. Windows setup is `scripts\setup.bat` then `scripts\start.bat`. He leaves that window open. Chrome loads the unpacked `extension` folder.

They must not meet the way a normal web app meets a backend. Chrome’s local-network rules treat a `fetch("http://127.0.0.1:8765/...")` from the YouTube tab as a public site asking for a private address, and they block it.

So the YouTube page never fetches localhost. The MAIN-world script that hooks timedtext does not know the server exists. The isolated content script does not call `127.0.0.1` either. Every health check, every job POST, every WAV, goes through the service worker.

```javascript
chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  if (message?.type !== "dabing-http") return;
  fetch(message.url, {
    method: message.method || "GET",
    headers: message.body ? { "content-type": "application/json" } : {},
    body: message.body ? JSON.stringify(message.body) : undefined,
    cache: "no-store",
  }).then(/* json or base64 WAV */).catch((err) => sendResponse({ error: String(err) }));
  return true;
});
```

`extension/lib/api.js` wraps that. The worker is the process that holds `host_permissions` for `http://127.0.0.1:8765/*`. The page origin never becomes the HTTP client.

That detail decides everything: it separates “works in my unpacked extension on a Mac” from “works in Chrome on his laptop after the next Chrome local-network prompt.” Commit `f850a56` is titled for this.

### Pattern 2: Hardware that says no

I do not want a 3 GB TTS model to start loading because I left `DABING_TTS=auto` and the import happened to succeed. The first thing the server should know is what the box actually is.

```python
def probe() -> Hardware:
    ram = _ram_gb()
    gpus = _gpus()
    cuda, mps = _torch_flags()
    if cuda and ram >= 24:
        tts, translate = "omnivoice", "opus"
    else:
        tts, translate = "piper", "argos"
```

On Windows, RAM comes from `Win32_ComputerSystem` and GPUs from `Win32_VideoController`, plus `nvidia-smi` if it exists. OmniVoice is recommended only when there is CUDA **and** at least 24 GB of RAM. His laptop fails both checks. Intel or AMD laptop graphics do not count.

`python -m dabing diagnose` prints the probe. `server/tests/test_hardware.py` asserts that a box under 20 GB without CUDA must come back with `recommended_tts == "piper"`. I do not want a future commit to “improve” the default onto a model that will swap his machine to death.

OmniVoice still exists in `server/dabing/tts.py`. The `full` extra pins `omnivoice==0.2.1`. I have used that path on a Mac when I wanted to hear the nicer voice. I have not put it on his laptop. Even there, voice design is an instruction string — `male, middle-aged, low pitch` — not a clip of the YouTuber.

### Pattern 3: Persistent work as files, scheduled against the playhead

YouTube cues are short. A second and a half of “Hello everyone and” is a bad TTS request. `merge_cues` packs them into windows: minimum 8 s, target 14 s, maximum 20 s (`server/dabing/config.py`; `CONTEXT.md` still says 8–18). Split on a 1.15 s scene gap, prefer a sentence-final period, skip `[music]`. Those numbers are the size of work the worker can finish while the video is still playing.

The worker is one background thread. It does not synthesize the whole video before playback. “Useful” is defined by the playhead the extension posts every 700 ms.

```python
def priority(seg: Segment) -> tuple[int, float]:
    if seg.end < now - 1.0:
        return (2, seg.start)  # already passed, do later
    if seg.start <= now + 45:
        return (0, seg.start)
    return (1, seg.start)
```

If he skips ahead, we speak near the new time first. If he already passed a line, it drops to the back of the queue. Ready segments are `0000.wav`, `0001.wav` under the cache. The booth’s gold strip is lead time: seconds of Czech in front of `video.currentTime`. Ten seconds and up is gold. Under three is red.

I do not have a saved real-time-factor log from his i5. The design assumption, written in the README, is a short wait on the first clip and then Piper staying ahead. If the strip stays red, the video is faster than the CPU. Pause, or pick a slower talker. That is an honest failure mode. It is also why Modal is written down.

Czech from Piper tends to run long. `PiperSynthesizer` shortens phoneme duration with `length_scale`, never below 0.62. If the WAV is still more than 6% longer than the caption window, it interpolates down. The player will raise Czech `playbackRate` as far as 1.38. None of that is elegant. All of it is what you do when you will not pause his video for a perfect take.

When he clicks **Zapnuto**, the YouTube `<video>` volume goes to 15%. Czech WAV plays in a separate `Audio` element. If YouTube shows an ad (`#movie_player.ad-showing`), Czech pauses. I am not going to dub the pre-roll. The MP4 on Google’s CDN is untouched.

{{< responsive-image src="images/dabing-pipeline.png" alt="Captions through service worker to local Piper server to ducked video" maxWidth="720px" >}}

*The YouTube page never fetches localhost. The service worker is the HTTP client.*


## Case study: captions or nothing

The MAIN-world script, `extension/injected.js`, tries, in order: a transcript already on screen, the English caption track (preferring a human track over YouTube’s `kind: "asr"`), a hook on `/api/timedtext`, Innertube `get_transcript`, then a click on “Show transcript” / “Zobrazit přepis” and a DOM scrape.

`kind: "asr"` is YouTube’s label for an auto-caption track. Dabing does not transcribe audio. If there is no English track and the scrape is empty, the error is `no_english_captions`. If a track exists but every fetch comes back blocked, `captions_blocked`. Live video is `live_unsupported`. The booth text is the contract: *Tohle video nemá anglické titulky.*

I would rather he see a red lamp than hear a hallucinated Czech paragraph over a cooking video with no text.

Tests speak a tone. `server/tests/conftest.py` forces mock engines before the app import. The mock translator prefixes `[cs]`. The mock TTS writes a soft 196 Hz tone. Re-run 2026-08-19: `python -m pytest server/tests` → **46 passed** in 0.54 s — less a listening test than a promise that the job state machine, the WAV path, and the hardware default can be broken in CI without a GPU.

## What this cut does not do

`docs/modal.md` exists because I can already see the video where Piper loses: dense, fast English, gold strip stuck red. The shape is: his Chrome still reads captions; those few kilobytes of JSON go to a token-gated Modal endpoint; opus-mt and OmniVoice run there; WAV chunks come back; the booth UI does not change. Do not send the video. Do not set `min_containers=1` and keep a T4 warm all month. The arithmetic in that file — about ten cents for a 40-minute video at a conservative RTF 0.2 on a T4 — is planning arithmetic from published Modal rates as of August 2026, short of an actual billed run.

Until that is wired, “we can always burst to a GPU” would be a slide.

| In | Out |
|---|---|
| Captioned English YouTube | Videos without English captions (no ASR) |
| Piper Jirka on `127.0.0.1:8765` | Live streams; any site that is not YouTube |
| Hardware probe that selects Piper on 12–16 GB / no CUDA | Cloning the original speaker |
| Mock-engine tests (46 passed) | Live OmniVoice on dad’s i5 |
| GPL-3.0 Piper; OmniVoice weights CC-BY-NC, private family install | Modal (documented, not implemented) |

There is leftover Mac copy in the tree — the manifest still says “generated on your Mac.” The setup he is supposed to double-click is `setup.bat`. Sample WAVs of Jirka in `samples/` prove the voice exists. They are not a benchmark of his CPU.

## Implications

The work I care about here is smaller than a model card. A content script that will not fetch localhost. A worker that will. A probe that will not load the model that does not fit. A pipeline that prefers the next 45 seconds over the scene he already skipped. A red lamp when YouTube gave us nothing to say.

Licenses for a house, not a store. Piper-tts is GPL-3.0. OmniVoice’s code is Apache-2.0; the weights are CC-BY-NC. That mix is acceptable for a private family install; a paid dubbing SaaS wrapped around OmniVoice would step outside it.

If you only remember one thing: I did not put a cloud dubbing stack on my dad’s i5. I put Jirka on `127.0.0.1:8765`, and I stopped where the captions stop.
