# Attune — a two-way bridge for neurodivergent & neurotypical communication

Built for **IncludAI — The Neurodiversity Hackathon** (IncludEDU × Stanford NNEA), Track 2: *AI for Connection & Wellbeing*.

## The gap this addresses

Most existing tools that touch neurodivergent communication train the neurodivergent person to mask: soften your tone, add small talk, guess the unstated subtext, perform neurotypical norms so you're not seen as blunt, rude, or "too much." That labor runs one direction, and it's exhausting.

Separately, many neurodivergent people — especially autistic people — experience **alexithymia**: real difficulty translating a body sensation into an emotion word. And unfamiliar situations (a fire drill, a new routine, an assembly) are a well-known source of anticipatory anxiety with too little preparation support.

Attune is six connected tools, none of which ask the neurodivergent person to change who they are:

1. **The Bridge** — a two-way, two-model translation pipeline (see architecture below) that preserves a message's real meaning while adjusting only the framing needed for a neurotypical reader to parse the intended tone — and decodes indirect neurotypical phrasing into its plain, literal meaning, so inference labor doesn't only sit with the neurodivergent listener.
2. **Name It** — a sensation-to-emotion mapper for alexithymia that runs two independent reasoning passes and only calls an emotion "cross-validated" when both agree, instead of presenting a single guess as fact.
3. **Story Builder** — a multimodal "Social Story" generator (Carol Gray's established previewing technique) that scripts, illustrates, and narrates a 4-step walkthrough of an upcoming situation the person is anxious about.
4. **Rehearse** — a conversation simulator for a genuinely hard exchange (asking a teacher for more time, telling a friend you need space). One model roleplays the other person; a second, independent model coaches only your side, out of view of the roleplay, after every line.
5. **Notes** — a self-advocacy message generator: describe a need in your own words, get a short version for a text/chat message and a longer version for an email, in your voice rather than flattened corporate ask-speak.
6. **Battery** — a quiet, two-tap social/sensory capacity log — no streaks, no guilt copy, deliberately not AI.

A seventh panel, **Insights**, runs no model at all: it's plain client-side statistics (rolling battery trend, a same-session anomaly check, a time-window correlation between battery dips and confirmed emotions) computed live from what you've actually logged, kept deliberately free of AI inference so it never claims more than the numbers support.

## Architecture — this is not a single prompt in a text box

Attune runs entirely client-side on **[Puter.js](https://developer.puter.com)**, which proxies free, keyless access to Gemini 3 models (Puter's "User-Pays" model — each visitor authenticates with a free Puter account and their usage doesn't cost the developer anything or require any backend).

Three different jobs are routed to three different models:

| Model | Job | Why this model |
|---|---|---|
| `gemini-3.6-flash` | Drafts 3 candidate phrasings per Bridge translation | Fast and cheap — most candidates get discarded, so speed matters more than depth here |
| `gemini-3-pro-preview` | Scores each candidate for meaning-preservation vs. tone-fit, picks the best, writes the subtext explanation; runs the two-pass emotion cross-validation in Name It; writes the Social Story script | Needs real reasoning depth and reliable structured (JSON) output |
| `gemini-3.1-flash-image-preview` + Gemini TTS | Illustrates each Social Story panel; narrates Bridge output and the story aloud | Multimodal generation — this is the part a single text model can't do |

**The Bridge pipeline**, concretely: `gemini-3.6-flash` drafts 3 phrasings at directness levels `d-1, d, d+1`. `gemini-3-pro-preview` then independently scores every candidate for two separate axes (does it keep the original request/boundary intact vs. does it match the intended tone), picks the best-scoring one, and explains *why* — a verifier pattern, not a single generate-and-return call. The full scoring is shown to the user in an expandable "Show how this was built" panel, so the reasoning isn't hidden.

**Name It's ensemble**: two independent passes are run in parallel — one weighting the physical sensations most heavily, one weighting the stated situational context most heavily — and results are merged. An emotion that both passes land on independently is marked "cross-validated" with an averaged confidence; an emotion only one pass surfaces is shown with a discounted confidence. This treats single-model output as a hypothesis to cross-check, not a verdict.

**Story Builder** chains three modalities: `gemini-3-pro-preview` writes a 4-step script following Social Stories conventions (first-person, descriptive, affirmative, non-directive), the image model illustrates each step from a per-step prompt in a consistent flat, calm, uncluttered style, and Gemini TTS narrates the finished story aloud.

**Rehearse** runs two models per turn, in parallel, with different jobs and different views of the conversation: `gemini-3.6-flash` plays the other person in the scenario (realistic, not a pushover, not a caricature), while `gemini-3-pro-preview` — never shown the partner's reply — independently coaches only the practicing person's latest line. Keeping the coach blind to the partner's response is deliberate: it stops the coaching from just reacting to how the roleplay happened to go, and keeps it grounded in what was actually said.

**Notes** asks `gemini-3-pro-preview` for two lengths of the same self-advocacy message in one structured call, tuned by audience and tone, explicitly instructed not to over-apologize or pathologize the need.

**Two free, keyless browser APIs are used alongside Puter.js:**
- The native **Web Speech API** (`SpeechRecognition`) powers the microphone button next to the emotion-context field, for anyone who finds typing harder than speaking in the moment. No key, no network call beyond the browser's own engine.
- The native **Web Speech Synthesis API** is the offline fallback for read-aloud if Gemini TTS is unavailable.

**Graceful degradation is real, not decorative.** Every AI feature has a transparent, rule-based local fallback (`localTranslate`, `localEmotionMap`, `localSocialStory`, `localArt` in `index.html`) so the app is always demoable even with no network, no Puter account, or a declined sign-in — and the status pill always tells you which mode is active.

## Running it

This is a single static file. No build step, no API key to configure.

```bash
python3 -m http.server 8000
# visit http://localhost:8000
```

Or just open `index.html` directly in a browser. The first AI call will prompt a free, quick Puter sign-in (handled entirely by the `js.puter.com` script) — decline it and the app keeps working in local fallback mode.

## Project structure

```
attune/
├── index.html    # entire app: markup, styles, and logic
├── README.md
├── SUBMISSION_DRAFT.md   # starting point for the Devpost project description
└── DEMO_SCRIPT.md        # 3-minute demo video script
```

## Judging criteria — how this was designed against them

- **Impact on neurodivergent youth (30%)** — targets three specific, well-documented pain points (masking labor in communication; alexithymia in emotion recognition; anticipatory anxiety before unfamiliar situations) rather than a generic "AI study buddy." Coping suggestions are small and concrete, not prescriptive therapy.
- **Innovation in AI application (25%)** — draft-then-verify model routing in the Bridge, two-pass ensemble cross-validation in Name It, a three-modality generation chain in Story Builder, and a dual-model roleplay-plus-blind-coach architecture in Rehearse are all genuine multi-step AI architectures, not a single wrapped API call. Insights deliberately uses *no* model at all, to show a considered line between what should be inferred and what should just be counted. The app is explicit at every step about which reasoning is live and which is a local fallback.
- **Technical execution (10%)** — fully working, keyless, degrades gracefully at every failure point, includes real state (phrasebook, vocabulary, battery log), and uses two free native browser APIs (speech recognition, speech synthesis) alongside the model pipeline.
- **Presentation quality (10%)** — this README, the demo script, and the in-app "How it works" section explain the architecture honestly, including what's a fallback and what's live model reasoning.

## Honest gap: real neurodivergent user involvement

The hackathon rules require genuine engagement with a real neurodivergent user in design or testing, described in the submission — **that part has to come from you, not from an AI assistant**, and it should be true. This build gives you a working prototype to actually put in front of someone today. Write down what they said, what confused them, and what you changed because of it — then put that, honestly, in `SUBMISSION_DRAFT.md`. Don't submit fabricated testing; judges are specifically scoring for whether this was actually designed *with* someone, not just *for* them.

## Roadmap (if this continues past the hackathon)

- Persist phrasebook/vocabulary/battery log across sessions (deliberately left in-memory-only for this prototype).
- Let a trusted contact (teacher, parent, partner) opt into seeing battery check-ins with consent.
- Expand the sensation vocabulary, emotion set, and Social Story library with input from an occupational therapist or the NNEA advisory group.
- Cache generated Social Story illustrations so a repeated situation (e.g. a recurring fire drill) doesn't re-generate art each time.
