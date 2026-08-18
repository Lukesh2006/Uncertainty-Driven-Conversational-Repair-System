# ContextBridge AI

**Context-Aware Real-Time Multilingual Conversational Interpreter**

Two people, each with their own phone and earbuds, speaking different
languages. A laptop between them does the interpreting.

The system does not just translate. It decides whether it *understood*
first, and repairs the conversation when it did not — by asking the
speaker to repeat, or by asking which of two things they meant.

```
UNDERSTAND  →  EVALUATE  →  REPAIR IF NECESSARY  →  TRANSLATE  →  SPEAK
```

---

## 1. How to run it

Three things, in this order.

### The server (the intermediate laptop)

```
python run_server.py
```

It prints two addresses:

```
Console : https://127.0.0.1:8000/
Phones  : https://10.239.136.242:8000/
```

**HTTPS is the default, and it is not optional for the phones.** Chrome
grants microphone access only to a "secure context" — HTTPS, or
localhost. Over plain HTTP the client loads, shows its join screen, and
then fails the moment it asks for the mic, and there is no permission
setting that changes that.

The certificate is generated on first run and self-signed, so each
phone shows **"Your connection is not private"** once. Tap
*Advanced → Proceed*. That is expected. After that the origin counts as
secure and the microphone works normally.

```
python run_server.py --http              # no mic on phones; localhost testing only
python run_server.py --new-certificate   # after changing Wi-Fi network
```

The certificate lists the laptop's current LAN address as an IP entry in
its Subject Alternative Names. Chrome validates the address you typed
against that list, and a mismatch is rejected outright rather than
merely warned about — so moving to a different network needs a new
certificate. The server detects this and regenerates automatically.

### Mobile App A and Mobile App B

Open the **Phones** address in each phone's browser — note `https://`.
Both phones must be on the same Wi-Fi as the laptop.

On each phone:

1. Enter the **same session code** (the console's *Create* button
   generates one).
2. Choose **Person A** on one phone, **Person B** on the other.
3. Choose the language that person speaks.
4. Tap **Join conversation**, then **Start conversation**.

Then just talk. Capture is automatic — the client detects when someone
starts and stops speaking.

> **Use earbuds.** Without them each phone's microphone hears the
> translated voice coming out of its own speaker and sends it back as
> new speech, and the two phones translate each other in a loop.

### The desktop console (optional)

**In a second terminal window** — the server holds the first one, and
Ctrl+C to get a prompt back kills it.

```
python app.py
```

Shows the live conversation, confidence, repair state and the context
engine's working memory.

**The console captures no audio.** It has no microphone and is not a
third participant. Server plus two phones is the complete system;
nothing is lost by never running the console.

---

## 2. Why the phone client is a web app

The specification asked for `mobile/app_a/` and `mobile/app_b/`. What is
built instead is one web client at `mobile/client/`, served by the
server and opened in each phone's browser.

The reasons:

- A native Android/iOS app cannot be written in this project's language
  or toolchain, and would need a build pipeline, a signing identity and
  a device deployment step before a single sentence could be translated.
- The web client runs on **both** platforms today, over the LAN, with
  no install.
- One codebase serves both people. The role is a parameter, so App A and
  App B are symmetrical *by construction* rather than by remembering to
  change both.

Everything the specification asked App A and App B to do — capture,
connect, choose languages, play translated audio, show original and
translated text, show connection status — the client does.

The trade-off, stated plainly: it needs the browser tab open and
foregrounded. A native app could capture with the screen off.

---

## 3. Final architecture

```
   PERSON A  (Tamil)                                  PERSON B  (Spanish)
        │                                                     │
   🎧 Earbuds A                                          🎧 Earbuds B
        │  Bluetooth                                          │  Bluetooth
   📱 Mobile App A                                       📱 Mobile App B
        │  WebSocket over Wi-Fi                               │
        └──────────────►  💻 CENTRAL AI SERVER  ◄─────────────┘
                              (this laptop)

                    ┌─────────────────────────────┐
                    │  audio quality measurement  │
                    │  original audio archive     │
                    │  speech recognition         │
                    │  ASR confidence             │
                    │  translation (+ pivot)      │
                    │  translation confidence     │
                    │  context engine             │
                    │  repair engine  L1 / L2 / L3│
                    │  text to speech             │
                    │  conversation log           │
                    └─────────────────────────────┘
```

One pipeline serves both directions. There is no separate A→B and B→A
implementation — the direction is `(source_language, target_language)`,
read from the session.

---

## 4. Files

### Created

| File | What it does |
|---|---|
| `run_server.py` | Starts the server; prints the LAN address for the phones |
| `server/certificates.py` | Generates the self-signed TLS certificate that unlocks the phone microphone |
| `server/server.py` | FastAPI app: WebSocket endpoint, REST, serves the phone client |
| `server/session_manager.py` | Sessions, speaker slots, connection routing |
| `server/pipeline.py` | The interpretation pipeline — the core of the system |
| `modules/asr.py` | Server-side speech recognition from uploaded audio |
| `modules/translation.py` | Concurrent forward + pivot + back-translation |
| `modules/context_engine.py` | Conversation memory and reference resolution |
| `modules/confidence_engine.py` | Audio, ASR and translation reliability scoring |
| `modules/repair_engine.py` | Level 2 / Level 3 state machine |
| `modules/audio_storage.py` | Optional original-audio archive |
| `database/conversation.py` | Session and utterance persistence (thread-safe) |
| `mobile/client/index.html` | Phone client markup |
| `mobile/client/app.js` | Capture, streaming, playback, display |
| `mobile/client/style.css` | Phone-sized dark interface |
| `gui/interpreter.py` | Desktop console window |
| `gui/client.py` | Console's background WebSocket thread |
| `gui/conversation_view.py` | Pooled transcript widgets |
| `gui/statusbar.py` | Header, context panel, repair banner |
| `tests/test_contextbridge.py` | Tests 4–14 with stubbed AI services |
| `tests/test_live_services.py` | Tests 1–3 and latency, against live services |
| `tests/test_end_to_end.py` | Two simulated phones through the real server |
| `tests/test_widget_colours.py` | Guard against the `color is None` bug |
| `docs/CONTEXTBRIDGE.md` | This document |

### Modified

| File | Change | Why |
|---|---|---|
| `config.py` | Renamed the app; added interpreter languages, ASR locales, confidence thresholds, context bounds, server and audio-storage settings | One place to configure the system; no hard-coded language pairs |
| `app.py` | Launches the console instead of the login window | The old multi-screen app is gone |
| `modules/tts.py` | Added `synthesize()` returning MP3 bytes | The server needs audio as data to send over a socket; the existing class only plays locally through pygame |
| `requirements.txt` | Added `fastapi`, `uvicorn`, `websockets` | The server |
| `ui/theme.py` | Reads settings directly instead of via the deleted `SettingsController` | Removed a dependency on a deleted file |
| `ui/components/__init__.py` | Dropped chart/table/skeleton exports | Those components were deleted |
| `ui/components/cards.py` | `sparkline` argument accepted but no longer drawn | Chart components were deleted; the argument stays so existing calls do not raise |
| `tests/test_widget_contracts.py` | Points at the surviving and new widgets | Kept a useful test alive instead of deleting it |

### Deleted

Out of scope per the specification:

`gui/ocr_window.py`, `gui/pdf_window.py`, `gui/export_dialog.py`,
`gui/chatbot.py`, `modules/ocr.py`, `modules/pdf_translator.py`,
`modules/exporter.py`, `ui/components/charts.py`,
`ui/components/table.py`, `ui/components/skeleton.py`,
`app/analytics.py`, `tests/test_analytics.py`

The old multi-screen app, replaced by the console:

`gui/home.py`, `gui/login.py`, `gui/auth.py`, `gui/register.py`,
`gui/screens/` (dashboard, translate, history, reports, profile,
settings), `ui/shell.py`, `ui/sidebar.py`, `app/controllers.py`

Superseded modules:

`modules/translator.py` → `modules/translation.py`,
`modules/speech.py` → `modules/asr.py`,
`modules/ai_assistant.py`, `modules/clipboard.py`,
`modules/security.py`, `modules/languages.py`

`database/database.py` was **kept**. Nothing in ContextBridge uses it,
but it holds the users and history tables from the previous app, this
project is not under version control, and the specification did not ask
for it to go.

---

## 5. Data flow

### Mobile A → Server → Mobile B

```
1  Person A speaks Tamil into earbuds A
2  App A detects speech start (RMS above threshold)
3  App A downsamples to 16 kHz mono PCM16 and streams chunks
      → {"type": "audio_start"} … binary … {"type": "audio_end"}
4  Server buffers the utterance under session ABC123, speaker A
5  Pipeline:  ta → es   (from the session, not from the message)
6  Server sends to App B:  {"type": "translation", own: false, …}
7  Server sends to App B:  {"type": "tts_audio", …} + binary MP3
8  Server sends to App A:  {"type": "translation", own: true, …}
9  App B plays the MP3 into earbuds B
```

### Mobile B → Server → Mobile A

Identical, with the roles exchanged. Step 5 becomes `es → ta`. Same
code, same pipeline, same session — the conversation context built by
A's turn is available to B's turn.

---

## 6. The AI pipeline

```
                       PCM from the phone
                              │
                    ┌─────────▼──────────┐
                    │  measure audio     │   RMS, peak, clipping, silence
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  archive original  │   only if the session enabled it
                    └─────────┬──────────┘
                              │
                     usable? ─┴─ no ──────────►  LEVEL 2  RE-LISTEN
                              │ yes                    (no ASR call spent)
                    ┌─────────▼──────────┐
                    │ speech recognition │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  ASR confidence    │   model score, or stated heuristic
                    └─────────┬──────────┘
                              │
                   reliable? ─┴─ no ──────────►  LEVEL 2  RE-LISTEN
                              │ yes
                    ┌─────────▼──────────────────────────┐
                    │  translate  ─┬─ forward  (spoken)  │  concurrent:
                    │              ├─ pivot     (context)│  one round trip,
                    │              └─ back      (score)  │  not three
                    └─────────┬──────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │ context analysis   │   on the English pivot
                    └─────────┬──────────┘
                              │
                  meaning ────┴─ ambiguous ───►  LEVEL 3  CLARIFICATION
                     clear?   │                        (utterance parked)
                    ┌─────────▼──────────┐
                    │ LEVEL 1 resolution │   silent, if a pronoun resolved
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  text to speech    │   failure here keeps the text
                    └─────────┬──────────┘
                              │
                         deliver to the listener
```

**Why this order.** Audio quality is measured before recognition so an
obviously dead clip costs zero network round trips. ASR confidence is
checked before translation for the same reason — and because there is
no point resolving the meaning of a sentence that was probably misheard.
**Level 2 must precede Level 3.**

---

## 7. Context engine

`modules/context_engine.py`

Language-independent by design: it never reads Tamil or Spanish. Every
utterance is also translated to an **English pivot**, and all entity
tracking and reference resolution happen there. That is what lets a name
introduced by the Tamil speaker resolve a pronoun used later by the
Spanish speaker — both sides contribute to one shared memory.

It tracks:

- **people** — capitalised tokens mid-sentence; sentence-initial tokens
  only when followed by a subject marker (`Raj is coming` names a
  person, `Meet me at nine` does not)
- **objects** — determiner + noun, stored *with* the determiner so
  substitution stays grammatical (`send it` → `send the report`, not
  `send report`)
- **topic** — most repeated content words
- **turns** — the last `CONTEXT_WINDOW_TURNS` utterances

It is **deterministic**, not an LLM call. Two reasons: it sits directly
in the latency path and runs in microseconds, and a clarification
question that appeared at random would be worse than none at all.

**Bounded.** The turn history is a `deque(maxlen=…)`, entities expire
after `CONTEXT_ENTITY_TTL_TURNS` without a mention, and the topic
counter is trimmed. Test 13 asserts all three.

---

## 8. Confidence engine

`modules/confidence_engine.py`

Every number traces to a measurement. Nothing is invented to make the UI
look confident, and a missing signal produces `None` — which propagates
as "unknown", not as `1.0`.

| Signal | Source | Fallback |
|---|---|---|
| Audio quality | RMS, peak, clipping and silence measured from the PCM | none needed — always measurable |
| ASR reliability | Google's own `confidence` on the top alternative (**real**) | stated heuristic: alternative disagreement × speaking rate × audio quality |
| Translation | round-trip agreement — translate back and compare | pass-through ratio only, reported as `heuristic` |

The report says which was used (`asr_source: "model"` vs `"heuristic"`),
so a reader can tell a measured score from an estimated one.

Round-trip back-translation is a real signal and costs no extra
wall-clock time, because it runs concurrently with the forward
translation.

Thresholds live in `config.py`:

```python
ASR_CONFIDENCE_FLOOR         = 0.55   # below this → Level 2
ASR_CONFIDENCE_GOOD          = 0.75   # below this → flagged as medium
TRANSLATION_CONFIDENCE_FLOOR = 0.60
MAX_RELISTEN_ATTEMPTS        = 2
```

---

## 9. The three repair levels

### Level 1 — context reinterpretation (silent)

The meaning is ambiguous but context resolves it *uniquely*. No
interruption.

```
Earlier:  "I spoke to Ravi yesterday."
Now:      "What did he say?"
Sent:     "What did Ravi say?"
```

The listener's phone shows a small badge: `"he" → Ravi`. The
conversation does not stop.

When Level 1 rewrites the sentence, the forward translation is redone
from the resolved English — otherwise the listener would still hear
"him". That extra round trip happens only on turns where a pronoun
actually resolved.

### Level 2 — re-listen

The words themselves are not trustworthy: the audio was unusable, or
ASR confidence fell below the floor.

```
🟡  RE-LISTENING
    "That wasn't clear. Could you say it again?"
```

Bounded to `MAX_RELISTEN_ATTEMPTS`. After that the best attempt is
passed through — an interpreter stuck saying "please repeat" is worse
than one that occasionally mistranslates. The budget is **per utterance
and per speaker**: A being asked to repeat does not consume B's budget,
and one bad patch of audio does not disable re-listen for the rest of
the conversation.

The listener is told the other side is being re-prompted, so their
screen does not just sit silent — but they receive no half-understood
text.

### Level 3 — targeted clarification

The words were heard correctly; the *meaning* is ambiguous. Two or more
candidates exist, and guessing would put words in the speaker's mouth in
a language they cannot check.

```
🟠  CLARIFICATION
    When you said "him", did you mean Ravi or Raj?
    [ Ravi ]  [ Raj ]
```

Asked **in the speaker's own language** — asking a Tamil speaker in
English defeats the point. Answers are tappable buttons where the
ambiguity has a closed set.

Triggers implemented:

- **person reference** — a pronoun with ≥2 candidate antecedents
- **ambiguous clock time** — `at nine` / `at 9` with no AM/PM marker.
  `at 9 AM`, `at 9 in the morning` and `at 14:00` do not trigger.

The utterance is parked, not dropped. After the answer: context is
updated, the sentence is resolved, translated, spoken and delivered.
Parked utterances expire after 45 seconds so a speaker who ignores the
prompt is not answering a question from a minute ago.

**Context is only committed for utterances that reached the listener** —
an abandoned clarification does not poison the conversation memory.

---

## 10. Original audio storage

Off by default. Recording someone's voice is not a neutral default.

```
output/sessions/
    session_ABC123/
        A_0001.wav
        B_0001.wav
        manifest.jsonl
```

`manifest.jsonl` records, per clip: session id, speaker, language,
timestamp, duration, byte count and the reference. The same reference
is stored on the `conversation` row.

- Toggled per session from either phone or the console; the recording
  state is visible on every screen while it is on.
- Callers never see filesystem paths — they get an opaque reference like
  `session_ABC123/A_0003.wav`, and a pattern check stops `../` from
  escaping the storage root.
- Session folders older than `AUDIO_RETENTION_DAYS` (default 7) are
  pruned when the server starts.
- A failed archive write never interrupts a live conversation.

---

## 11. Latency

### Measured

`tests/test_live_services.py::test_latency_against_target`, on this
machine, with clean synthesised audio:

```
samples : 3
min     : 1946 ms
average : 2163 ms
max     : 2503 ms
target  : 5000-6000 ms
verdict : within target
```

Full two-phone path over the real `wss://` protocol
(`tests/test_end_to_end.py`):

```
Tamil   → Spanish   2574 ms  /  5526 ms   (+ 15.5 KB of MP3 delivered)
Spanish → Tamil     2511 ms  /  4940 ms
```

Two numbers because **run-to-run variance is large and it is not ours**.
The same AI-only benchmark measured 2163 ms average on one run and
3181 ms average on another with no code change in between, so the spread
comes from Google's service response times, not from TLS or from the
server. TLS costs a handshake per connection, not per utterance.

Individual utterances across all live tests ranged **1946–5526 ms** —
inside the 5–6 s target, but the upper end is close enough to it that a
slow day on Google's side will exceed it. Nothing in this design can
prevent that; the fix would be local models, which is a different
project.

### Budget

| Stage | Cost |
|---|---|
| Upload + buffer | 0.1 – 0.3 s (overlaps speech) |
| Speech recognition | 0.8 – 2.0 s |
| Translation bundle | 0.4 – 1.0 s (three calls, concurrent) |
| Context analysis | < 0.01 s (local) |
| Re-translation after Level 1 | 0 or ~0.4 s |
| Speech synthesis | 0.5 – 1.5 s |
| **Typical total** | **2.5 – 5.0 s** |

### What was done to get there

1. **Concurrent translation.** Forward, pivot and back-translation run
   in a thread pool. Sequentially they would add roughly a second to
   every turn.
2. **Streaming upload.** The phone sends PCM while the person is still
   speaking, so the upload finishes almost as soon as the sentence does.
3. **Raw PCM, not opus.** MediaRecorder would give webm/opus, needing
   ffmpeg and a transcode step server-side.
4. **Early rejection.** Audio quality is measured locally before any
   network call, so a dead clip costs 0 ms instead of a wasted ASR
   round trip.
5. **Deterministic context.** Microseconds, not an LLM round trip.
6. **Re-translate only when needed.** The extra call happens only on
   turns where Level 1 actually rewrote the sentence.
7. **Shared clients.** Recogniser and thread pool are built once for
   the process, not per utterance.

### Honest limits

- Every AI stage is a network call to Google. On a slow or congested
  link the figures above will be worse, and nothing in this design can
  fix that — the fix would be local models, which is a different
  project.
- The measured numbers use clean synthesised audio. A real room adds
  capture time and, at Bluetooth mic quality, more re-listens.
- Turns that hit Level 3 take as long as the person takes to answer.
  That is the feature working, not a latency failure.

---

## 12. Wire protocol

`ws://host:8000/ws?session=CODE&role=A|B|console&language=ta`

**Client → server**

```json
{"type": "audio_start"}
                                    ← binary frames: 16 kHz mono PCM16
{"type": "audio_end"}
{"type": "set_language", "speaker": "A", "language": "ta"}
{"type": "swap_languages"}
{"type": "clarification_answer", "answer": "Ravi"}
{"type": "set_audio_saving", "enabled": true}
{"type": "pause"}   {"type": "resume"}   {"type": "reset"}
```

**Server → client**

```json
{"type": "status", "session_id": "ABC123", "languages": {...}, "connected": ["A","B"]}
{"type": "translation", "speaker": "A", "source_language": "ta",
 "target_language": "es", "original_text": "...", "translated_text": "...",
 "confidence": {...}, "resolutions": [...], "latency_ms": 2574}
{"type": "tts_audio", "bytes": 15552}   ← followed by a binary MP3 frame
{"type": "relisten", "prompt": {"level": 2, "message": "...", "attempt": 1}}
{"type": "clarification", "prompt": {"level": 3, "question": "...", "options": [...]}}
{"type": "context_update", "context": {...}}
{"type": "error", "error": "..."}
```

Every message carries `session_id`, `speaker`, `source_language` and
`target_language` context. Speaker identity comes from the connection,
not from the audio — no voice diarisation, which is both more reliable
and exactly right when each person has their own phone.

---

## 13. Database

`database/conversation.py`, in the same SQLite file, with its own
connection, its own lock and WAL mode — the GUI reads while pipeline
worker threads write.

**`sessions`** — session id, started/ended, both languages, audio flag.

**`conversation`** — session id, timestamp, speaker, source and target
language, original text, resolved text, translated text, decision,
confidence label, ASR score, translation score, repair level, repair
detail, latency, audio reference.

Indexed on `(session_id, id)`.

---

## 14. Performance and GUI safety

- All AI work happens in a thread pool via `run_in_executor`. The event
  loop is never blocked, so one slow conversation cannot stall another.
- The console's socket lives on a background thread that **never**
  touches a widget. Messages go onto a `queue.Queue`, drained by a
  single `root.after(120, …)` tick. That tick is the only place widgets
  are updated.
- Transcript rows are a **fixed pool**, refilled in place. No widget is
  created per sentence.
- Every `after` id the window owns is tracked and cancelled at teardown,
  before widgets are destroyed. That ordering is the fix for
  `invalid command name "...update"` — those errors come from Tk firing
  a scheduled callback against an already-destroyed widget.
- Recogniser, translation pool and TTS are constructed once per process.
- Per-session `asyncio.Lock`: utterances in one conversation are ordered,
  different conversations stay fully parallel.

---

## 15. Bugs from the report

**`Database.get_history() missing 1 required positional argument`** —
the two call sites (`app/controllers.py:323`, `app/analytics.py:341`)
both passed `user_id` correctly; the fault was already resolved in the
code as it stood. Both files have since been deleted with the old
screens, so `get_history` now has no callers at all. `database.py`
itself was kept.

**`ValueError: color is None`** — originated in the dashboard, reports
and translate screens, which passed tone values looked up from
dictionaries that could miss. Those screens are deleted.
`tests/test_widget_colours.py` was added so it cannot return: it asserts
no design token is `None`, builds every surviving and new widget, drives
`UtteranceRow` through every confidence decision *including an unknown
one*, and checks the optional `color=None` arguments substitute a token
rather than forwarding `None`.

**`invalid command name "...update"` / `"...check_dpi_scaling"` /
`"..._click_animation"`** — the lifecycle fix is in
`InterpreterWindow.close()`: cancel tracked `after` jobs, then drain
`ui.animation`'s pending callbacks, then stop the socket thread, then
destroy. Verified by closing the console mid-traffic (Test 14).

---

## 16. Testing

```
python -m pytest tests/ --ignore=tests/test_end_to_end.py -q   # offline
node tests/test_vad.js                                         # phone-client capture
python run_server.py                                           # then, separately:
python -m pytest tests/test_end_to_end.py -q -s                # two-phone path
python -m pytest tests/test_live_services.py -q -s             # Tests 1-3 + latency
```

`tests/test_vad.js` runs the real `app.js` in a Node `vm` context
against synthetic audio frames. Voice-activity detection decides what
the recogniser ever hears, so a bug there presents as "the AI is
inaccurate" rather than as a bug — worth testing directly. 12 cases:
pre-roll, bounded buffering, silence handling, mid-sentence pauses,
threshold adaptation, and echo suppression during playback.

| Spec test | Where | Result |
|---|---|---|
| 1 Tamil → Spanish | `test_live_services.py` | pass — `¿Hay una reunión mañana?` |
| 2 Spanish → Tamil | `test_live_services.py` | pass — `ஆம், காலை 10 மணிக்கு` |
| 3 English → Tamil | `test_live_services.py` | pass |
| 4 `he` → Ravi | `test_4_*` | pass, incl. cross-language |
| 5 Low-quality audio → Level 2 | `test_5_*` | pass, incl. bounded retries |
| 6 Ravi vs Raj → Level 3 | `test_6_*` | pass, incl. answering |
| 7 9 AM vs 9 PM → Level 3 | `test_7_*` | pass, incl. non-triggers |
| 8 Alternating speakers | `test_8_*` | pass |
| 9 Audio storage ON | `test_9_*` | pass |
| 10 Audio storage OFF | `test_10_*` | pass, incl. path traversal |
| 11 Network interruption | `test_end_to_end.py` | pass |
| 12 TTS failure keeps text | `test_12_*` | pass |
| 13 Long conversation bounded | `test_13_*` | pass |
| 14 Close during processing | manual + teardown path | pass |

**Known failure, pre-existing and unrelated:**
`tests/test_design_system.py::test_radii_are_monotonic_and_within_brief`
asserts `Radius.XL <= 24` while `ui/tokens.py` defines it as `28`. That
assertion predates this work and `ui/tokens.py` was not modified. It is
left as found rather than quietly adjusted.

---

## 17. Known limitations

1. **Bluetooth microphone quality.** Capturing from earbuds forces the
   headset profile — mono, narrowband — which is measurably worse for
   speech recognition than the phone's own mic. If accuracy disappoints,
   test with the phone mic first to separate recognition errors from
   audio-quality errors.

2. **Everything needs the internet.** ASR, translation and TTS are all
   Google services. Offline, nothing works. The failure is reported
   rather than hidden.

3. **The English pivot loses nuance.** Context tracking on an English
   rendering is what makes the engine language-independent, but Tamil →
   English → context is lossy for languages structurally distant from
   English. The forward translation the listener hears is always direct
   source → target; only the *context analysis* uses the pivot.

4. **Entity extraction is heuristic.** It relies on English
   capitalisation in the pivot. An uncommon name that Google
   lower-cases will not be tracked; an unusual sentence-initial verb may
   be mistaken for a name. The cost of the latter is a spurious
   clarification, which is why the subject-marker test exists.

5. **Ambiguity detection covers two categories** — person references and
   clock times. Real conversation has many more.

5b. **Voice activity detection is energy-based.** It compares loudness
   against an adaptive noise floor. Sustained noise *louder* than
   someone speaking is indistinguishable from speech by that measure —
   a television at conversational volume will open utterances. The
   threshold adapts upward to compensate, but only so far
   (`MAX_START_RMS`). A quiet room is where this works best.

6. **No speaker diarisation.** Speaker identity comes from which socket
   the audio arrived on. Two people sharing one phone appear as one
   speaker.

7. **Encrypted, but not authenticated.** Traffic is TLS over `wss://`,
   so audio is encrypted across the Wi-Fi. But the certificate signs
   itself, so nothing proves the server is the laptop you think it is —
   that is what the browser warning is telling you, and tapping through
   it accepts exactly that risk. There is also **no session
   authentication**: anyone on the same network who guesses a
   6-character code can join a conversation. Session authentication is
   the obvious next step.

8. **Consent for recording is a policy, not a mechanism.** The system
   shows recording state clearly and defaults to off, but nothing
   enforces that the *other* speaker agreed. In many jurisdictions
   recording a conversation requires both parties' consent.

9. **Browser tab must stay foregrounded** — see §2.

10. **The console is observer-only.** It cannot speak into a
    conversation. That is deliberate, not missing.
