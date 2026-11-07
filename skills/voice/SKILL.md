---
name: voice
description: Turn text, scripts and documents into natural speech, clone the user's own voice, or transcribe recordings, with Voice by Conversion Tools. Use when the user wants narration, a voice-over, an audio version of a document, a podcast or YouTube script read aloud, speech in their own cloned voice, help recording a voice sample for cloning (including a short sample script written for the topic and mood of what will be narrated), or a transcript of an MP3, WAV or FLAC recording. 66 built-in voices in 21 languages, MP3 or WAV output.
compatibility: Works with the Conversion Tools MCP server bundled in this plugin (claude.ai, Cowork, Claude Code and other MCP clients) and with the ctio CLI when the agent has shell access. Voice cloning itself happens on voice.conversiontools.io.
metadata:
  author: conversiontools
  version: "1.0"
  website: https://voice.conversiontools.io
---

# Voice: text to speech, voice cloning and transcription

Three jobs, one account:

- **Text to speech:** narrate a text, a script, or a whole PDF, Word or Markdown file, in one of 66 built-in voices or in a voice the user cloned.
- **Voice cloning:** the user records about 10 seconds of their own voice on voice.conversiontools.io and gets a Voice ID to narrate with. You help them record a sample that fits what they will narrate.
- **Speech to text:** transcribe an MP3, WAV or FLAC recording into a plain text file.

## How to run it

- **Shell available (Claude Code, Codex, terminals):** use the `ctio` CLI if it is installed (`ctio --version`). The `convert` skill covers installing and signing in.
- **No shell (claude.ai, Cowork) or no ctio:** use the Conversion Tools MCP tools from this plugin: `convert_file` with a `converter`, the input (`url`, or `file_id` from `request_upload_url`, or base64 `file_content` for small files) and `options`. On claude.ai and in Cowork the user connects the plugin's connector once first: **Customize > Plugins > Conversion Tools > Connectors > Connect**.

The result is a download link valid for about 30 seconds. Fetch it right away; if it expired, call `get_download_url` with the task id. Never produce the audio or the transcript some other way and present it as the result.

## Text to speech

| Converter | Input |
|---|---|
| `convert.ai_text_to_mp3` (also `convert.ai_text_to_speech`) | Plain text |
| `convert.ai_pdf_to_mp3` (also `convert.ai_pdf_to_speech`) | PDF |
| `convert.ai_docx_to_mp3` (also `convert.ai_docx_to_speech`) | Word |
| `convert.ai_md_to_mp3` (also `convert.ai_md_to_speech`) | Markdown (fenced code blocks are skipped) |

Options:

- `voice_tts`: a built-in voice id, or the Voice ID of a voice the user cloned (usable only from the account that cloned it).
- `language_tts`: the language the text is written in (38 codes). The voice reads the text as written, so pick a voice that speaks that language.
- `output_tts`: `mp3` or `wav`. With ctio the output file name picks it.

An unknown voice, language or format is an error that names the value, never a silent fallback.

**Choosing a voice.** List the voices before picking one: `ctio voice --format pretty` (built-in voices plus the user's own clones, each with its language), or `get_converter_info` with `convert.ai_text_to_mp3` over MCP (every `voice_tts` value with its label). Match the language first, then the style the user asked for.

```bash
# A script, narrated; the output name picks the format
ctio convert -t ai_text_to_speech script.txt narration.mp3 --option voice_tts=gb_jane_neutral --option language_tts=en

# A whole document
ctio convert -t ai_pdf_to_speech handbook.pdf handbook.wav --option voice_tts=fr_marie_neutral --option language_tts=fr

# In the user's own cloned voice
ctio convert -t ai_text_to_speech episode.txt episode.mp3 --option voice_tts=<Voice ID from the dashboard>
```

Over MCP, the same job is one `convert_file` call: `converter` `convert.ai_text_to_mp3`, the text file as `file_content`, `file_id` or `url`, and `options` `{"voice_tts": "gb_jane_neutral", "language_tts": "en", "output_tts": "mp3"}`.

**The audio.** Spoken-word loudness (about -16 LUFS), 24 kHz mono, ready to drop into a video or podcast without a loudness pass.

**Write for the ear before you convert.** The voice reads exactly what is on the page:

- Short sentences; one idea each.
- Spell out what should be said: "two thousand twenty-six", "per cent", "doctor", not codes or symbols the listener cannot see.
- Commas and full stops make the pauses. A paragraph break makes a longer one.
- Remove headings, footnote markers, URLs and table remnants that would be read aloud verbatim.

**Cost and length.** 10 AI credits per 1,000 characters. A free account gets 100 AI credits a month, about 10,000 characters, which is roughly 6 to 10 minutes of audio: voices narrate at about 1,000 to 1,600 characters a minute, and a clone keeps the pace of its sample. For a long document, run one paragraph first to check the voice, then the whole file. If a conversion fails on a credit or plan limit, say so and point to https://conversiontools.io/pricing instead of retrying.

## Clone the user's own voice

**Consent comes first.** Only clone the user's own voice, or a voice whose speaker has given explicit permission. The clone form asks the user to confirm exactly that. Do not help clone a celebrity, a public figure, or anyone who has not agreed, and do not write a sample script meant to imitate someone else.

**Two ways to clone.** In the browser on [voice.conversiontools.io](https://voice.conversiontools.io), which records the sample for the user (any surface, nothing to install), or with ctio from an audio file (shell available). Both create the same clone in the same account: it shows on the dashboard and in `ctio voice`.

**In the browser,** on a computer or a phone:

1. Sign in with the Conversion Tools account and open **Dashboard > Clone a voice**. **Record** is the default (**Upload** is next to it for a clip recorded elsewhere).
2. Tap the microphone and allow access. Read the sample script (below) while a timer runs: 3 seconds is the minimum, about 10 seconds works best, and it stops on its own at 30 seconds.
3. Listen to the take. If it does not sound right, **Record again**: the name, tone and consent are kept.
4. Name the voice, pick a tone label, confirm consent, **Clone voice**.

Each clone shows its **Voice ID** with a copy button; that ID is the `voice_tts` value for every narration above. If the microphone is blocked or missing, the page offers **Upload a clip instead**.

**With ctio,** from a recording the user already has:

```bash
ctio voice clone my-voice.m4a --name "My narrator" --tone calm --consent "I confirm this is my own voice, or I have the speaker's explicit permission to clone it."
```

- **Consent is the user's to give.** Show the user the statement, ask whether it is true for this recording, and add `--consent` only after they confirm. Never add it on your own. The statement must be word for word (`ctio voice` returns it as `consent_statement`); without it ctio stops before uploading anything and prints the exact flag.
- `--name` is required (up to 60 characters); `--tone` is the label (normal, excited, cheerful, confident, curious, calm, sad, angry, frustrated; default normal).
- The output carries the new Voice ID (`--format pretty` prints `Cloned "<name>" (<tone>). Voice ID: <id>`). Use it as `voice_tts`.
- `ctio voice` lists the built-in voices and the user's clones. `ctio voice delete <voice_id> --yes` deletes a clone for good: ask the user first.
- When all clone slots are used, the clone is refused with a message saying so: delete a clone or upgrade the Voice plan.

Over MCP there is no clone tool: send the user to the browser flow above.

**For an uploaded or ctio clip.** MP3, WAV, M4A, FLAC, OGG or WebM (the website upload also takes Opus), 3 seconds minimum, about 10 seconds of clear speech recommended, up to 512 KB on the free plan and 2 MB on paid plans (save as MP3 if a WAV is too big). A browser recording is far smaller than that. Clone slots: 1 on the free plan, 5 on Starter, 20 on Pro.

**The clip decides the mood.** The tone the user picks on the form is a label for telling clones apart; the clone copies the energy, mood and pace of how the sample was spoken. So:

- Record the sample in the mood the narration needs. A calm sample gives a calm narrator; an excited one gives an excited narrator.
- Record at the speed the narration should have. The clone keeps the sample's pace.
- Record in the language of the narration.
- Different moods need different clones: one calm clone for an audiobook, one cheerful clone for channel intros, each from its own sample.

**Recording tips to pass on.**

- Quiet room, soft furnishings, no music, no fan, no echo.
- One speaker only.
- Phone or microphone about a hand's width to a forearm's length away, the same distance for the whole take.
- Keep the script where it is easy to read without moving: on a second screen, or printed out when recording on the phone.
- Listen to each take before cloning and record again until it sounds the way the narration should.

### Write the sample script for the topic and mood

When the user wants to clone a voice, write the sample they will read. A sample written for their narration teaches the clone the right tone far better than reading something unrelated.

1. **Find out, or infer from the conversation:** what will be narrated (topic, audience, platform), the language, and the mood. Map the mood to one of the form's tone labels: normal, excited, cheerful, confident, curious, calm, sad, angry, frustrated.
2. **Write 2 to 4 sentences, 25 to 40 words**, which is 10 to 15 seconds at a natural pace. The sample must:
   - be in the language of the narration and use words from its topic;
   - carry the mood in the words themselves, so it is natural to say it that way;
   - mix sentence types: a statement, a question, a short pause, ideally one number said in words, so the clone hears varied intonation;
   - avoid tongue twisters, long lists of names, abbreviations, all caps and anything the user would not naturally say.
3. **Add delivery notes:** target pace, energy, where to pause, whether to smile while speaking (it is audible).
4. **Offer one alternative sample** in case the first does not feel natural to say, and remind the user to record a few takes.

Examples (English; write the sample in the narration's own language):

- **Calm, for a sleep story or meditation audiobook.** *"Let your shoulders soften, and take a slow breath in. There is nowhere else you need to be right now. Breathe out, and notice how the room grows quiet around you."* Delivery: slow, about 120 words a minute, long pauses at each full stop, low energy, no smile.
- **Cheerful, for a YouTube channel intro.** *"Hey everyone, welcome back to the channel! Today we're trying three tiny changes that make a big difference. Ready? Let's jump right in."* Delivery: upbeat, about 160 words a minute, smile while speaking, lift on the question.
- **Confident, for a training or product explainer.** *"In the next five minutes, you'll learn how to set up your first report. It takes three steps, and you won't need any spreadsheets. Let's start with the dashboard."* Delivery: steady, about 140 words a minute, clear consonants, firm sentence endings.
- **Curious, for a documentary or history podcast.** *"In eighteen eighty-three, a volcano erupted so loudly it was heard thousands of miles away. But what did people actually hear? And why did the sky turn red for months afterwards?"* Delivery: measured, about 130 words a minute, rising tone on the questions, a short pause before each.

**Then test the clone** before a long job: narrate one short paragraph of the real content with the new Voice ID. If it sounds too fast, too flat or too excited, record a new sample in the delivery that is wanted rather than adjusting the text.

## Speech to text

| Converter | Input |
|---|---|
| `convert.ai_mp3_to_text` | MP3 |
| `convert.ai_wav_to_text` | WAV |
| `convert.ai_flac_to_text` | FLAC |

```bash
ctio convert -t ai_mp3_to_text interview.mp3 interview.txt
```

Over MCP: `convert_file` with `converter` `convert.ai_mp3_to_text` and the recording as `url` or `file_id` (upload with `request_upload_url` first; recordings are usually too big for `file_content`).

The result is a plain text file with the full transcript. For a video, extract the audio first with `convert.mp4_to_mp3`, then transcribe the MP3. Transcription costs 5 AI credits per minute of audio, so the free 100 credits a month cover about 20 minutes.

## Links

- [Voice by Conversion Tools](https://voice.conversiontools.io): voice cloning and the voice dashboard
- [Text to speech converter](https://conversiontools.io/convert/ai-text-to-mp3)
- [ctio CLI docs](https://conversiontools.io/docs/ctio)
- [Pricing and AI credits](https://conversiontools.io/pricing)
