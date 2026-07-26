# What Now Doctor? — Code Overview

**A talking aftercare companion for cataract-surgery patients.**

You've just been told you need eye surgery. The consultant was lovely, the leaflet is somewhere in your bag, and at 9pm the questions start: *Will it hurt? When can I drive? What actually happens in there?*

What Now Doctor? gives patients a warm, spoken answer the moment they ask — grounded strictly in the clinic's own aftercare material — and quietly flags anything worrying to the surgeon. Patients hold a button and talk; the app talks back.

> This document is a guided tour of the interesting code. It contains **no API keys, secrets, or credentials** — the project itself keeps all provider access behind managed server-side proxies, so there are none in the source at all.

---

## The demo flow

1. Patient signs in through a simple demo gate (any name + a fixed demo password).
2. A spoken welcome greets them by name, with four common questions as tappable chips (pre-generated audio, instant playback).
3. They **hold the talk button and ask out loud** — or type. The answer comes back as text *and* a natural voice.
4. Say something concerning ("it's really painful", "my eye is bleeding") and the AI steps aside: the question is **flagged to Dr John immediately** with clear escalation guidance.

---

## Architecture

```
┌─────────────────────────────┐          ┌──────────────────────────────────┐
│  Patient portal (React)     │          │  API server (Express)            │
│  artifacts/apresdoc         │   /api   │  artifacts/api-server            │
│                             │ ───────► │                                  │
│  • hold-to-talk recorder    │          │  /chat  ── RAG + Claude          │
│  • live waveform (WebAudio) │          │  /voice ── ElevenLabs TTS + STT  │
│  • chips + typed questions  │          │  /flags ── safety escalations    │
│  • journey & leaflet views  │          │  /auth, /patients, /pathway      │
└─────────────────────────────┘          └────────────┬─────────────────────┘
                                                      │ managed proxies
                                                      ▼
                                     Anthropic Claude · ElevenLabs voice
                                     (credentials never touch this repo)
```

| Layer     | Choice                                                 |
| --------- | ------------------------------------------------------ |
| Frontend  | React 18 + Vite, framer-motion for the animated UI     |
| Backend   | Express 5 (TypeScript, ESM), pino structured logging   |
| AI        | Claude Sonnet, constrained by a retrieval-grounded prompt |
| Voice     | ElevenLabs — flash TTS for live answers, Scribe for STT |
| Knowledge | Curated cataract aftercare chunks (single TS module)   |

---

## 1. The voice loop

The heart of the app is a four-step round trip: **record → transcribe → answer → speak**. The frontend records with `MediaRecorder`, picking the best container the browser offers (Safari records `audio/mp4`, Chrome `audio/webm`):

```ts
const preferred = ["audio/webm;codecs=opus", "audio/webm", "audio/mp4"];
const mime = typeof MediaRecorder !== "undefined"
  ? preferred.find(m => MediaRecorder.isTypeSupported(m)) ?? ""
  : "";
const rec = mime ? new MediaRecorder(stream, { mimeType: mime }) : new MediaRecorder(stream);
```

Two small touches make it feel solid in the hand:

**Accidental taps are caught** — a clip under half a second (or under ~1.5 KB) is treated as a mis-press, with a gentle hint instead of a failed request:

```ts
if (duration < 500 || blob.size < 1500) {
  onError("Keep the button held down while you speak, then let go.");
}
```

**Letting go during the permission prompt is respected** — if the patient releases the button while the browser is still asking for mic access, we don't start a recording they're no longer holding for:

```ts
const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
// The user may have let go while the permission prompt was up —
// in that case don't start a recording they're no longer holding for.
if (!holdingRef.current) {
  stream.getTracks().forEach(t => t.stop());
  return;
}
```

The clip goes to `POST /api/voice/transcribe` as a raw body (no multipart ceremony), gets transcribed server-side, and the text follows the same path as a typed question.

---

## 2. Grounded answers, not hallucinations

Medical Q&A is exactly where an LLM must not improvise. The retrieval layer is deliberately simple — a curated knowledge base of aftercare chunks, each tagged with the phrases patients actually use:

```ts
export function getRelevantChunks(message: string): string | null {
  const lower = message.toLowerCase();
  const matched = cataracts.filter((chunk) =>
    chunk.topic.some((t) => lower.includes(t)),
  );
  if (matched.length === 0) return null;
  return matched.map((c) => c.content).join("\n\n");
}
```

What makes it safe is the contract in the system prompt. With matching chunks, Claude may **only** use them:

```
Answer using only the information provided below.
Do not add anything beyond what is written here.
```

And when *nothing* matches, the model is not allowed to guess. It hands the question to the humans — warmly, and in one prescribed shape:

```
"Leave that one with me — I've sent your question straight through to your
clinical team and you can expect to hear back within 2 hours. ..."
```

One rule outranks everything else in the prompt:

```
CRITICAL SAFETY RULE: If the patient describes anything that sounds acute or
potentially serious — sudden pain, increasing redness, loss of vision,
flashing lights, a dark shadow across their vision — always end with:
"If you are at all concerned, please use the emergency number you were given
when you left the clinic — do not wait."
```

---

## 3. The safety net runs *before* the AI

Trigger words never even reach the model. A plain, auditable word list short-circuits the chat route — no inference, no latency, no chance of a model deciding a bleeding eye can wait:

```ts
export const FLAG_TRIGGERS = [
  'pain', 'painful', 'hurts', 'hurting',
  'red', 'redness', 'bloodshot',
  "can't see", 'cannot see', 'lost sight',
  'bleeding', 'blood',
  'worse', 'worsening', 'deteriorating',
  'severe', 'unbearable', 'emergency', 'urgent',
  // ...
];
```

```ts
if (containsTrigger(message)) {
  patient.flagged = true;
  patient.flagMessage = message;
  patient.flaggedAt = new Date().toISOString();

  return res.json({ response: ESCALATION_RESPONSE, flagged: true });
}
```

The patient gets an honest, calm handover ("I have flagged this to Dr John right now…") including hard escalation guidance if symptoms worsen. A deterministic list is a feature here, not a shortcut: a clinician can read all of it in ten seconds and sign it off.

---

## 4. Making it speak like a human

Live answers are voiced by ElevenLabs. Three details in `lib/elevenlabs.ts` are worth showing.

**The screen and the voice get different spellings.** Patients should *read* the correct medical term, but TTS engines mangle "phacoemulsification". So the respelling happens only at the moment text becomes speech — display text is never touched:

```ts
/**
 * Respellings applied only to the text sent to the voice, never to what's
 * shown on screen: patients see the correct medical spelling, while the
 * voice gets a phonetic version it can pronounce ("fayco", not "p-hacko").
 */
const PRONUNCIATIONS: ReadonlyArray<[RegExp, string]> = [
  [/phaco-?emulsification/gi, "fayco emulsification"],
  [/\bphaco\b/gi, "fayco"],
];

export async function textToSpeech(text: string): Promise<Buffer> {
  const spokenText = forPronunciation(text);
  // ...
}
```

**Repeated answers cost one API call.** A small in-memory cache keyed on `voice | model | text` means replays and common questions don't re-bill:

```ts
const ttsCache = new Map<string, Buffer>();
const TTS_CACHE_MAX = 60;

const key = createHash("sha256")
  .update(`${voiceId}|${TTS_MODEL}|${spokenText}`)
  .digest("hex");
```

**Voice selection degrades gracefully.** Connected ElevenLabs keys are often permission-restricted, so the voice resolver tries the account's voice list and falls back to a premade voice that exists on every account — speech never breaks because a key couldn't *list* voices:

```ts
} catch (err) {
  logger.warn({ err }, "Voice list unavailable — using default premade voice");
  cachedVoiceId = DEFAULT_VOICE_ID; // "Lily" — warm, calm, on every account
  return cachedVoiceId;
}
```

The greeting and the four chip answers are pre-generated MP3s shipped as static assets — instant playback, zero API calls, and the demo's first impression never depends on a third-party being up.

---

## 5. The iPhone microphone problem

The hardest UX bug in the project: on iOS, embedded webviews (in-app browsers, preview panes) **auto-deny** `getUserMedia` without ever showing a prompt. To the patient it looks like the button simply doesn't work.

The fix is a classifier that turns each failure into the *right* piece of advice — including detecting the silent auto-deny by combining frame context with response time (a human can't read and dismiss a permission prompt in under ~400 ms):

```ts
function micErrorMessage(err: unknown, waitedMs: number): string {
  const name = err instanceof DOMException ? err.name : "";
  if (name === "NotFoundError" || name === "OverconstrainedError") {
    return "We couldn't find a microphone on this device. Tap one of the questions below, or type yours instead.";
  }
  if (name === "NotReadableError" || name === "AbortError") {
    return "Another app seems to be using your microphone. Close it, then hold the button and try again.";
  }
  if (name === "NotAllowedError" || name === "SecurityError") {
    if (isEmbedded() || waitedMs < 400) {
      return "The microphone was blocked before it could ask you. Open this page in your phone's web browser (like Safari) and try again — or tap one of the questions below.";
    }
    return "No problem — voice stays off until the microphone is allowed. ...";
  }
  return "We couldn't start the microphone. Tap one of the questions below, or type your question instead.";
}
```

Every branch ends the same way: the patient is always pointed at a path that still works (chips or typing). Voice is the delight, never a dead end.

---

## 6. Small details that keep the demo honest

**The waveform is decorative — and knows it.** The talk button shows a live level meter driven by a WebAudio analyser. If anything in that chain fails, recording carries on untouched:

```ts
function startMeter(stream: MediaStream) {
  try {
    const ctx = new Ctor();
    const source = ctx.createMediaStreamSource(stream);
    const analyser = ctx.createAnalyser();
    // ... RMS loop via requestAnimationFrame → levelRef
  } catch {
    /* metering is decorative — recording works without it */
  }
}
```

**Failures speak the patient's language.** Server errors come back as plain JSON with calm copy; the UI never shows a stack trace, and every error state offers the chips as a way forward.

**Structured logs, no `console.log`.** The server uses pino with per-request serializers, so every request — including AI latency and voice failures — is traceable in one line of JSON.

---

## Repo layout

```
artifacts/
  apresdoc/               # React patient portal (Vite)
    src/pages/            #   Entry (demo gate), PatientPortal, DoctorDashboard
    public/audio/         #   pre-generated greeting + chip answers (MP3)
  api-server/             # Express API
    src/routes/           #   chat, voice, flags, auth, patients, pathway
    src/lib/              #   elevenlabs.ts, rag.ts, flagTriggers.ts, logger
    src/data/             #   cataract-knowledge.ts, demo patients
```

---

## Security & privacy posture

- **No credentials in code, ever.** Anthropic and ElevenLabs are reached through server-side managed proxies; the browser never sees a provider token, and neither does this repository.
- **Demo data only.** The patient ("Margaret Holt") and her records are fictional seed data; state lives in memory and resets on demand.
- **Guardrails over vibes.** The model cannot answer outside the vetted aftercare content, trigger words bypass it entirely, and every escalation path ends in a human.

## Honest limitations (it's a hackathon build)

- Demo state is in-memory — flags and history reset when the server restarts.
- The sign-in is a demo gate, not real authentication.
- Voice transcription depends on the connected ElevenLabs key's permissions; the app degrades to chips + typing when it's unavailable.

---

*Built with Replit Agent. The full app is a monorepo of two services; this overview quotes the production code verbatim, lightly trimmed for reading.*
