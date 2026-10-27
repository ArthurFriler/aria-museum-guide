# Aria Museum Guide — a hands-free, real-time audio guide on Meta Project Aria glasses

> Look at an artwork, stop walking, and the guide starts talking. Walk away, and it pauses mid-sentence and picks up where it left off when you come back.

**Team:** Arthur Friler · Daniel Sitbon — Technion, CRML (Control Robotics & Machine Learning Lab)
**Supervisor:** Ben Kretzu
**Field test:** Tel Aviv Museum of Art, with 5 visitors (2026)

[![Demo video — click to watch on YouTube](https://img.youtube.com/vi/acS8IkUXweA/hqdefault.jpg)](https://youtu.be/acS8IkUXweA)

▶ **[Watch the demo](https://youtu.be/acS8IkUXweA)**

---

## What it does

A visitor wears Meta's Project Aria Gen 1 research glasses. The system streams the glasses' RGB camera and IMU to a laptop over Wi-Fi and runs a fully automatic loop:

1. **Detects that the visitor has stopped** in front of something (IMU-based walking detector).
2. **Recognises the artwork** from the camera frame with a vision-language model.
3. **Generates a spoken description** tailored to the visitor (language, age, art knowledge, preferred length).
4. **Speaks it sentence by sentence** while the rest is still being generated, so audio starts within ~3 s.
5. **Pauses as soon as the visitor walks away**, and on return either resumes the unheard part (same artwork) or starts the next one.

No buttons, no QR codes, no catalogue lookup.

## Architecture

```
 Aria glasses ──Wi-Fi──► RGB frames ─► [Vision worker] ─► sentence queue ─► [TTS worker] ─► audio queue ─► [Playback]
      │                                  gpt-5.4-mini                        tts-1 + local      pygame
      │                                  (streamed)                          clip cache
      └──────────────► IMU ─► [Walking detector] ── pause / resume events ──────────────────────────┘
```

Each stage is a thread connected by bounded queues (producer/consumer with back-pressure), so the first sentence is played while the model is still writing the next ones.

## Key engineering decisions

| Problem | Solution | Result |
|---|---|---|
| Latency between seeing an artwork and hearing something | Streamed VLM output parsed sentence by sentence; a pre-recorded opening line masks the remaining delay; model migration benchmarked | **2.98 s** to first sound, **7.75 s** to first real description (vs 4.58 s / 10.0 s with the previous model) |
| When should the guide stop talking? | IMU walking detector (accelerometer std over a sliding window) with hysteresis (enter 0.75 / exit 0.3 m/s²), minimum dwell time and a 3-sample debounce on walk onset — chosen over "artwork left the frame", which proved unreliable in testing | Pause is triggered at walk onset, not after the current TTS call returns |
| Stale results after the visitor moves on | Freshness filter: any frame captured before the last walk→still transition is skipped before reaching the vision model | Removes 10–20 s of wasted analysis when switching artworks |
| Same artwork described again and again | Normalised name matching with multilingual stop-word removal (EN/FR/ES) | Distinct works with similar titles no longer merge |
| Hundreds of TTS calls at every start-up | `pregenerate_audio.py` builds a content-addressed cache (SHA-1 of the sentence) for every language × age group × phrase pool | 288 clips generated once; only changed sentences are regenerated |
| One guide does not fit every visitor | `GuideProfile`: language (EN/FR/ES), age register, knowledge depth and length are independent axes injected into the system prompt and voice settings | Same pipeline serves a child novice and an adult expert |

The full development log (in French) is in [`docs/JOURNAL.md`](docs/JOURNAL.md).

## Repository layout

```
.
├── README.md
├── run.sh                    # one-command launch: visitor setup → Aria streaming → pipeline
├── requirements.txt
├── main.py                   # thread orchestration, pause/resume state machine
├── vision.py                 # artwork recognition + streamed description
├── tts.py                    # text-to-speech
├── audio.py                  # playback
├── motion.py                 # IMU walking detector
├── guide_profile.py          # visitor profile (language, age, knowledge, length)
├── guide_setup.py            # start-up questionnaire
├── pregenerate_audio.py      # builds tts_cache/
├── imu_debug.py              # live IMU threshold tuning
├── tts_cache/                # pre-generated clips (built by pregenerate_audio.py)
├── assets/                   # sample images for offline tests
├── docs/                     # development journal
├── third_party/aria_sdk_samples/  # Meta Project Aria Client SDK samples (Apache 2.0)
└── legacy/                   # early prototype (video-file simulation)
```

## Running it

Requirements: Project Aria Gen 1 glasses with the Aria Client SDK, macOS or Linux, Python 3.10, an OpenAI API key.

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
echo "OPENAI_API_KEY=sk-..." > .env
python pregenerate_audio.py           # once
./run.sh                               # Wi-Fi (default)  |  ./run.sh usb
```

Tip: institutional Wi-Fi tends to drop most RGB samples (DDS); a phone hotspot was the reliable setup in our tests.

## Limitations and next steps

- **Recognition is by name, not by image.** Artwork identity relies on the model's text answer; visual matching (CLIP embeddings or ORB features) would be more robust and work for uncatalogued pieces.
- **Cloud latency.** Description time is dominated by API round-trips; streaming TTS playback and an on-device model are the next levers.
- **Small-scale evaluation.** One museum, 5 visitors; a structured user study is the natural next step.

## Acknowledgements

Ben Kretzu and CRML (Technion) for supervision and hardware; the Tel Aviv Museum of Art for hosting the field test; Meta Reality Labs for the Project Aria research kit. Files under `third_party/aria_sdk_samples/` are © Meta Platforms, Inc., Apache License 2.0.
