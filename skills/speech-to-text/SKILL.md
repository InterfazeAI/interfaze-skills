---
name: speech-to-text
description: >
  Use Interfaze AI for speech-to-text transcription from audio files, voice notes,
  recordings, podcasts, and meeting audio. Activate when the user wants to transcribe
  audio to text, identify speakers, detect language, or extract timestamps — even if
  they say "what did they say in this recording" or "convert this audio to text"
  without explicitly mentioning transcription.
---

# STT — Speech-to-Text with Interfaze AI

Transcribe audio files into text with speaker diarization, timestamps, translation, and audio analysis using Interfaze AI.

## When to use this skill

- The user has an audio file and wants it transcribed to text
- The input is a voice note, meeting recording, podcast, interview, or phone call
- The user wants speaker identification or diarization from an audio source
- The user asks for timestamps alongside the transcript
- The user says "transcribe this", "what does this audio say", "convert this recording to text"
- The user wants to translate audio between languages

## When not to use this skill

- The input is an image or screenshot → use `ocr`
- The user wants to generate speech from text (text-to-speech) — not supported
- The input is already text that needs reformatting — no transcription needed
- The user wants a video's visual content → use `ocr` or `object-detection`

## Workflow

1. Confirm the input is an audio file URL or file reference.
2. Determine what the user needs: plain transcript, speaker-labeled transcript, timestamps, or translation.
3. Build a schema matching the desired output (Zod for TypeScript, Pydantic for Python).
4. Pass the audio in the message `content` array. A URL can also be passed inline in the prompt text (slightly faster).

## Setup

Every SDK reads the API key from `INTERFAZE_API_KEY`.

### TypeScript — Interfaze SDK

```bash
npm install interfaze
```

```ts
import { Interfaze } from "interfaze";

const interfaze = new Interfaze();
```

No base URL or model name to configure — both are built in.

### TypeScript — Vercel AI SDK

```bash
npm install ai @ai-sdk/openai
```

```ts
import { createOpenAI } from "@ai-sdk/openai";

const interfaze = createOpenAI({
  baseURL: "https://api.interfaze.ai/v1",
  apiKey: process.env.INTERFAZE_API_KEY,
});
```

Requires `ai@^7.0.0` and `@ai-sdk/openai@^4.0.0`.

### TypeScript — LangChain SDK

```bash
npm install @interfaze-ai/langchain
```

```ts
import { ChatInterfaze } from "@interfaze-ai/langchain";

const interfaze = new ChatInterfaze({ apiKey: process.env.INTERFAZE_API_KEY });
```

### Python — Interfaze SDK

```bash
pip install interfaze
```

```python
from interfaze import Interfaze

interfaze = Interfaze()
```

### Python — LangChain SDK

```bash
pip install interfaze-langchain
```

```python
import os
from interfaze_langchain import ChatInterfaze

interfaze = ChatInterfaze(api_key=os.environ["INTERFAZE_API_KEY"])
```

### Already using the OpenAI SDK?

Point an existing OpenAI client at Interfaze and pass `model: "interfaze"` on every call.

```ts
import OpenAI from "openai";

const interfaze = new OpenAI({
  baseURL: "https://api.interfaze.ai/v1",
  apiKey: process.env.INTERFAZE_API_KEY,
});
```

```python
from openai import OpenAI

interfaze = OpenAI(base_url="https://api.interfaze.ai/v1", api_key="<your-api-key>")
```

## Input formats

Sources: public HTTPS URLs and base64 data URLs.

| SDK | Audio part |
| --- | --- |
| Interfaze / LangChain (TS & Py) | `{ type: "file", file: { filename: "audio.mp3", file_data: "<url-or-base64>" } }` |
| Vercel AI SDK | `{ type: "file", data: "<url>", mediaType: "audio/mpeg" }` |

You can also pass the URL inline in the prompt across all SDKs — `{ role: "user", content: "Transcribe https://example.com/audio.mp3" }` — which is slightly faster and raises the size cap from 20 MB to 80 MB.

MIME types: MP3 → `audio/mpeg`, WAV → `audio/wav`, M4A → `audio/mp4`. Other formats use their canonical IANA MIME type.

The Interfaze SDK's `inputs.audio(url)` / `inputs.file(...)` helpers build these parts, and `inputs.fromPath()` / `inputs.from_path()` reads a local file.

## Example: basic transcription

### TypeScript — Interfaze SDK

```ts
import { responseFormat } from "interfaze";
import { z } from "zod";

const STTSchema = z.object({ text: z.string() });

const response = await interfaze.chat.completions.create({
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Transcribe the audio file" },
        {
          type: "file",
          file: { filename: "voice-note.mp3", file_data: "https://example.com/voice-note.mp3" },
        },
      ],
    },
  ],
  response_format: responseFormat(z.toJSONSchema(STTSchema), "stt_schema"),
});

console.log(JSON.parse(response.choices[0]?.message.content ?? "{}"));
console.log("STT Results:", response.precontext?.[0]?.result);
```

### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const STTSchema = z.object({ text: z.string() });

const { output, response } = await generateText({
  model: interfaze.chat("interfaze"),
  output: Output.object({ schema: STTSchema }),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Transcribe the audio file" },
        { type: "file", data: "https://example.com/voice-note.mp3", mediaType: "audio/mpeg" },
      ],
    },
  ],
});

console.log(output);

// @ts-expect-error precontext is not typed
console.log("STT Results:", response.body?.precontext?.[0]?.result);
```

### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const STTSchema = z.object({ text: z.string() });

const structuredModel = interfaze.withStructuredOutput(STTSchema, { includeRaw: true });

const response = await structuredModel.invoke([
  {
    role: "user",
    content: [
      { type: "text", text: "Transcribe the audio file" },
      {
        type: "file",
        file: { filename: "voice-note.mp3", file_data: "https://example.com/voice-note.mp3" },
      },
    ],
  },
]);

console.log(response.parsed);
console.log("STT Results:", response.raw.response_metadata.precontext?.[0]?.result);
```

### Python — Interfaze SDK

```python
from pydantic import BaseModel

class STTSchema(BaseModel):
    text: str

response = interfaze.chat.completions.parse(
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Transcribe the audio file"},
                {
                    "type": "file",
                    "file": {
                        "filename": "voice-note.mp3",
                        "file_data": "https://example.com/voice-note.mp3",
                    },
                },
            ],
        }
    ],
    response_format=STTSchema,
)

print(response.choices[0].message.parsed)
print("STT Results:", response.precontext[0].result if response.precontext else None)
```

### Python — LangChain SDK

```python
from langchain_core.messages import HumanMessage
from pydantic import BaseModel

class STTSchema(BaseModel):
    text: str

structured_llm = interfaze.with_structured_output(STTSchema, include_raw=True)

response = structured_llm.invoke([
    HumanMessage(content=[
        {"type": "text", "text": "Transcribe the audio file"},
        {
            "type": "file",
            "file": {
                "filename": "voice-note.mp3",
                "file_data": "https://example.com/voice-note.mp3",
            },
        },
    ])
])

print(response["parsed"])
print("STT Results:", response["raw"].response_metadata.get("precontext"))
```

## Example: speaker diarization

Same call as basic transcription — swap in this schema and prompt `"Transcribe and identify the speakers in the audio file"`. `speaker_id` is a string; up to 50 speakers are supported. Add analysis fields per chunk if needed, e.g. `sentiment: z.enum(["positive", "negative", "neutral"])`.

```ts
const DiarizationSchema = z.object({
  full_text: z.string(),
  chunks: z.array(
    z.object({
      speaker_id: z.string(),
      text: z.string(),
      start_time: z.number(),
      end_time: z.number(),
    })
  ),
  number_of_speakers: z.number(),
});
```
```python
class Chunk(BaseModel):
    speaker_id: str
    text: str
    start_time: float
    end_time: float

class DiarizationSchema(BaseModel):
    full_text: str
    chunks: list[Chunk]
    number_of_speakers: int
```

## Example: audio translation

Include the target language in the prompt (e.g. "Transcribe the audio file and translate it to Chinese") and use this schema. Both the transcription and the translation show up as separate `precontext` entries.

```ts
const TranslationSchema = z.object({
  translated_text: z.string().describe("translated text"),
  original_language_code: z.string(),
  translated_language_code: z.string(),
});
```
```python
class TranslationSchema(BaseModel):
    translated_text: str = Field(..., description="translated text")
    original_language_code: str
    translated_language_code: str
```

## Example: audio summary and intent

Add derived fields alongside the transcript and the model fills them in one pass.

```ts
const STTSchema = z.object({
  text: z.string(),
  summary: z.string().describe("two line summary of the audio"),
  intent: z.string().describe("3 word intent of the audio"),
});
```

## Plain transcription (no schema)

Drop the `response_format` / `output` argument when you only need raw text.

## Tips

- Use canonical MIME types: `audio/mpeg` (MP3), `audio/wav` (WAV), `audio/mp4` (M4A).
- Include "identify the speakers" in the prompt to trigger diarization.
- Specify the target language in the prompt for translation.
- Low-quality audio is de-noised automatically before transcription.
- 100+ languages are supported, including mixed-language audio.

## Advanced: long audio & word-level timestamps

For long audio (1hr+) or the lowest cost and latency, run transcription on its own. This activates only the audio part of the model — a 1hr 35min file transcribes in about 50 seconds. Ask for transcription "by word" to get word-level timestamps.

### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.transcribe("https://example.com/audio.mp3");

console.log(result);
```

### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";

const { output } = await generateText({
  model: interfaze.chat("interfaze"),
  system: "<task>speech_to_text</task>",
  output: Output.json(),
  prompt: "Transcribe the audio file https://example.com/audio.mp3",
});

console.log(output);
```

### TypeScript — LangChain SDK

```ts
import { HumanMessage, SystemMessage } from "@langchain/core/messages";

const response = await interfaze.invoke([
  new SystemMessage("<task>speech_to_text</task>"),
  new HumanMessage("Transcribe the audio file https://example.com/audio.mp3"),
]);

const { result } = JSON.parse(response.content as string);
console.log(result);
```

### Python — Interfaze SDK

```python
result = interfaze.tasks.transcribe("https://example.com/audio.mp3")

print(result)
```

### Python — LangChain SDK

```python
import json
from langchain_core.messages import HumanMessage, SystemMessage

response = interfaze.invoke([
    SystemMessage(content="<task>speech_to_text</task>"),
    HumanMessage(content="Transcribe the audio file https://example.com/audio.mp3"),
])

print(json.loads(response.content).get("result"))
```

Task mode rules: only one `<task>` runs per request, and the response format must be an empty or `any` schema — passing a real schema alongside a task returns a 400.

The result shape:

```json
{
  "name": "speech_to_text",
  "result": {
    "text": "I just started a round of amoxicillin and I wanted to ask ...",
    "chunks": [
      { "timestamp": [0.28, 4], "text": "I just started a round of amoxicillin and I wanted to ask" },
      { "timestamp": [4, 7.72], "text": "if it was safe to take that with my current spironolactone prescription." }
    ]
  }
}
```

For normal (non-task) calls the same data arrives on `precontext` under the name `stt`, alongside a `translate` entry when translation was requested.
