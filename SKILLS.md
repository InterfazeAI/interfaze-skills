# Interfaze AI — Skills Reference

Single-file reference combining all six Interfaze AI skills. Each capability also ships as an installable skill under `skills/<name>/SKILL.md`; this document concatenates them for reading in one place. The `## Setup` below is shared by every skill and is not repeated per section.

Interfaze AI is a proprietary multimodal model with first-party SDKs for TypeScript and Python. It also supports the Chat Completion API standard, so OpenAI-compatible clients work against its base URL.

- **Base URL:** `https://api.interfaze.ai/v1` (built into the first-party SDKs)
- **Model:** `interfaze` (built into the first-party SDKs)
- **Auth:** `Authorization: Bearer $INTERFAZE_API_KEY`

## Contents

- [Setup](#setup) — shared across all skills
- [OCR](#ocr--visual-ocr-with-interfaze-ai)
- [Speech-to-Text](#stt--speech-to-text-with-interfaze-ai)
- [Structured Output](#structured-output-with-interfaze-ai)
- [Object Detection](#object-detection-with-interfaze-ai)
- [Web Search](#web-search-with-interfaze-ai)
- [Web Scraping](#web-scraping-with-interfaze-ai)

## Setup

The same client setup applies to every skill below. Every SDK reads the API key from `INTERFAZE_API_KEY`.

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
npm install ai @interfaze-ai/ai-sdk
```

```ts
import { interfaze } from "@interfaze-ai/ai-sdk";
```

Requires `ai@^7.0.0`. The default `interfaze` export reads `INTERFAZE_API_KEY`; base URL and model name are built in. To configure it yourself, use `createInterfaze({ apiKey })`.

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

Interfaze supports the Chat Completion API standard, so point an existing OpenAI client at it and pass `model: "interfaze"` on every call.

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

For structured output on this path, use `zodResponseFormat(Schema, "schema_name")` from `openai/helpers/zod` in TypeScript, or a `json_schema` response format built from `Model.model_json_schema()` in Python.

## OCR — Visual OCR with Interfaze AI

Extract text and structured fields from images and scanned documents using Interfaze AI's vision OCR capability.

### When to use this skill

- The user has an image, screenshot, scan, or photo and wants text extracted from it
- The input is a receipt, invoice, ID card, form, menu, or label captured as an image
- The user wants structured fields (name, address, totals, line items) pulled from a visual source
- The user says "read this image", "extract the text", "what does this receipt say"
- The task involves layout-aware extraction where text position matters

### When not to use this skill

- The input is a native text file, JSON, or CSV — no vision needed
- The user wants to transcribe audio → use `speech-to-text`
- The task is locating objects without reading text → use `object-detection`
- The user wants to convert existing data to a schema → use `structured-output`
- The user wants to pull data from a live URL → use `web-scraping`

### Workflow

1. Confirm the input is an image URL, PDF URL, or base64-encoded file.
2. Decide whether you need plain text or structured fields.
3. For structured fields, define a Zod (TS) or Pydantic (Python) schema.
4. Pass the file in the message `content` array using the SDK's file/image part.
5. Match the SDK and language to the user's project.

### Input formats

Supported sources: public HTTPS URLs and base64 data URLs.

| SDK | Image part | PDF / document part |
| --- | --- | --- |
| Interfaze / LangChain (TS & Py) | `{ type: "image_url", image_url: { url: "<url>" } }` | `{ type: "file", file: { filename: "doc.pdf", file_data: "<url-or-base64>" } }` |
| Vercel AI SDK | `{ type: "image", mediaType: "image/jpeg", image: "<url-or-base64>" }` | `{ type: "file", data: "<url>", mediaType: "application/pdf" }` |

Size limits: 20 MB as a file part, 80 MB when the URL is written directly into the prompt text. For large documents, put the URL in the prompt instead — the model fetches it at inference time.

### Example: extract fields from an ID card

#### TypeScript — Interfaze SDK

```ts
import { responseFormat } from "interfaze";
import { z } from "zod";

const IDSchema = z.object({
  first_name: z.string().describe("First name on the ID"),
  last_name: z.string().describe("Last name on the ID"),
  dob: z.string().describe("Date of birth on the ID"),
  driver_licence_number: z.string().describe("Driver licence number on the ID"),
});

const response = await interfaze.chat.completions.create({
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Extract the details from this ID" },
        { type: "image_url", image_url: { url: "https://example.com/id.jpg" } },
      ],
    },
  ],
  response_format: responseFormat(z.toJSONSchema(IDSchema), "id_schema"),
});

console.log(JSON.parse(response.choices[0]?.message.content ?? "{}"));
console.log("OCR Results:", response.precontext?.[0]?.result);
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const IDSchema = z.object({
  first_name: z.string().describe("First name on the ID"),
  last_name: z.string().describe("Last name on the ID"),
  dob: z.string().describe("Date of birth on the ID"),
  driver_licence_number: z.string().describe("Driver licence number on the ID"),
});

const { output, providerMetadata } = await generateText({
  model: interfaze.chat("interfaze"),
  output: Output.object({ schema: IDSchema }),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Extract the details from this ID" },
        { type: "image", mediaType: "image/jpeg", image: "https://example.com/id.jpg" },
      ],
    },
  ],
});

console.log(output);

const precontext = providerMetadata?.interfaze?.precontext as { result?: unknown }[] | undefined;
console.log("OCR Results:", precontext?.[0]?.result);
```

#### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const IDSchema = z.object({
  first_name: z.string().describe("First name on the ID"),
  last_name: z.string().describe("Last name on the ID"),
  dob: z.string().describe("Date of birth on the ID"),
  driver_licence_number: z.string().describe("Driver licence number on the ID"),
});

const structuredModel = interfaze.withStructuredOutput(IDSchema, { includeRaw: true });

const response = await structuredModel.invoke([
  {
    role: "user",
    content: [
      { type: "text", text: "Extract the details from this ID" },
      { type: "image_url", image_url: { url: "https://example.com/id.jpg" } },
    ],
  },
]);

console.log(response.parsed);
console.log("OCR Results:", response.raw.response_metadata.precontext?.[0]?.result);
```

#### Python — Interfaze SDK

```python
from pydantic import BaseModel, Field

class IDSchema(BaseModel):
    first_name: str = Field(..., description="First name on the ID")
    last_name: str = Field(..., description="Last name on the ID")
    dob: str = Field(..., description="Date of birth on the ID")
    driver_licence_number: str = Field(..., description="Driver licence number on the ID")

response = interfaze.chat.completions.parse(
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Extract the details from this ID"},
                {"type": "image_url", "image_url": {"url": "https://example.com/id.jpg"}},
            ],
        }
    ],
    response_format=IDSchema,
)

print(response.choices[0].message.parsed)
print("OCR Results:", response.precontext[0].result if response.precontext else None)
```

#### Python — LangChain SDK

```python
from langchain_core.messages import HumanMessage
from pydantic import BaseModel, Field

class IDSchema(BaseModel):
    first_name: str = Field(..., description="First name on the ID")
    last_name: str = Field(..., description="Last name on the ID")
    dob: str = Field(..., description="Date of birth on the ID")
    driver_licence_number: str = Field(..., description="Driver licence number on the ID")

structured_llm = interfaze.with_structured_output(IDSchema, include_raw=True)

response = structured_llm.invoke([
    HumanMessage(content=[
        {"type": "text", "text": "Extract the details from this ID"},
        {"type": "image_url", "image_url": {"url": "https://example.com/id.jpg"}},
    ])
])

print(response["parsed"])
print("OCR Results:", response["raw"].response_metadata.get("precontext"))
```

### Example: receipt line items

Same call as the ID card above — only the schema changes.

```ts
const ReceiptSchema = z.object({
  items: z.array(z.object({ name: z.string(), price: z.string() })),
  total_cost: z.string(),
  tax: z.string(),
});
```
```python
class ReceiptItem(BaseModel):
    name: str
    price: str

class ReceiptSchema(BaseModel):
    items: list[ReceiptItem]
    total_cost: str
    tax: str
```

### Example: PDF document

Swap the image part for a file part. The call is otherwise identical to the ID card example.

```ts
{
  type: "file",
  file: { filename: "research-paper.pdf", file_data: "https://arxiv.org/pdf/1706.03762" },
}
```
```python
{
    "type": "file",
    "file": {"filename": "research-paper.pdf", "file_data": "https://arxiv.org/pdf/1706.03762"},
}
```

For the Vercel AI SDK, use `{ type: "file", data: "<url>", mediaType: "application/pdf" }`.

### Plain text (no schema)

Drop the `response_format` / `output` argument when you only need raw text.

```ts
import { generateText } from "ai";

const { text } = await generateText({
  model: interfaze.chat("interfaze"),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Extract all text from this image" },
        { type: "image", image: "https://example.com/scan.jpg" },
      ],
    },
  ],
});
```

### Reading local files

The Interfaze SDK's `inputs` helpers read a local file or raw bytes and build the content part.

```ts
import { inputs } from "interfaze";

inputs.image(await inputs.fromPath("./photo.png"));
inputs.file(await inputs.dataUrl(pdfBytes, "application/pdf"), { filename: "report.pdf" });
```
```python
from interfaze import inputs

inputs.image(inputs.from_path("./photo.png"))
inputs.file(inputs.data_url(pdf_bytes, "application/pdf"), filename="report.pdf")
```

### Tips

- Use `.describe()` / `Field(description=...)` when a field name alone is ambiguous.
- For multilingual text, name the target language in the prompt or field descriptions. 100+ languages, including mixed-language documents, are supported.
- For layout-aware extraction, include coordinate fields in your schema.
- The model corrects and normalizes text in the structured output. `precontext` holds the uncorrected raw OCR, which is what you want for auditing or confidence thresholds.

### Advanced: raw OCR output & bounding boxes

Run OCR on its own to get a fixed output shape with line-level bounding boxes and per-word confidence, skipping the full model. This is faster and cheaper than a schema-driven call.

#### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.ocr("https://example.com/id.jpg");

console.log(result);
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";

const { output } = await generateText({
  model: interfaze.chat("interfaze"),
  system: "<task>ocr</task>",
  output: Output.json(),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Extract all text from this ID" },
        { type: "image", mediaType: "image/jpeg", image: "https://example.com/id.jpg" },
      ],
    },
  ],
});

console.log(output);
```

#### TypeScript — LangChain SDK

```ts
import { HumanMessage, SystemMessage } from "@langchain/core/messages";

const response = await interfaze.invoke([
  new SystemMessage("<task>ocr</task>"),
  new HumanMessage({
    content: [
      { type: "text", text: "Extract all text from this ID" },
      { type: "image_url", image_url: { url: "https://example.com/id.jpg" } },
    ],
  }),
]);

const { result } = JSON.parse(response.content as string);
console.log(result);
```

#### Python — Interfaze SDK

```python
result = interfaze.tasks.ocr("https://example.com/id.jpg")

print(result)
```

#### Python — LangChain SDK

```python
import json
from langchain_core.messages import HumanMessage, SystemMessage

response = interfaze.invoke([
    SystemMessage(content="<task>ocr</task>"),
    HumanMessage(content=[
        {"type": "text", "text": "Extract all text from this ID"},
        {"type": "image_url", "image_url": {"url": "https://example.com/id.jpg"}},
    ]),
])

print(json.loads(response.content).get("result"))
```

Task mode rules: only one `<task>` runs per request, and the response format must be an empty or `any` schema — passing a real schema alongside a task returns a 400.

The result shape:

```json
{
  "name": "ocr",
  "result": {
    "extracted_text": "California\nUSA\nDRIVER LICENSE\n...",
    "sections": [
      {
        "text": "California\nUSA\n...",
        "lines": [
          {
            "text": "California",
            "bounds": {
              "top_left": { "x": 63, "y": 89 },
              "bottom_right": { "x": 268, "y": 129 },
              "width": 205,
              "height": 40
            },
            "average_confidence": 0.99,
            "words": [{ "text": "California", "confidence": 0.99, "bounds": {} }]
          }
        ]
      }
    ],
    "width": 698,
    "height": 525
  }
}
```

The same data is available on `precontext` for normal (non-task) calls, alongside your schema output.

## STT — Speech-to-Text with Interfaze AI

Transcribe audio files into text with speaker diarization, timestamps, translation, and audio analysis using Interfaze AI.

### When to use this skill

- The user has an audio file and wants it transcribed to text
- The input is a voice note, meeting recording, podcast, interview, or phone call
- The user wants speaker identification or diarization from an audio source
- The user asks for timestamps alongside the transcript
- The user says "transcribe this", "what does this audio say", "convert this recording to text"
- The user wants to translate audio between languages

### When not to use this skill

- The input is an image or screenshot → use `ocr`
- The user wants to generate speech from text (text-to-speech) — not supported
- The input is already text that needs reformatting — no transcription needed
- The user wants a video's visual content → use `ocr` or `object-detection`

### Workflow

1. Confirm the input is an audio file URL or file reference.
2. Determine what the user needs: plain transcript, speaker-labeled transcript, timestamps, or translation.
3. Build a schema matching the desired output (Zod for TypeScript, Pydantic for Python).
4. Pass the audio in the message `content` array. A URL can also be passed inline in the prompt text (slightly faster).

### Input formats

Sources: public HTTPS URLs and base64 data URLs.

| SDK | Audio part |
| --- | --- |
| Interfaze / LangChain (TS & Py) | `{ type: "file", file: { filename: "audio.mp3", file_data: "<url-or-base64>" } }` |
| Vercel AI SDK | `{ type: "file", data: "<url>", mediaType: "audio/mpeg" }` |

You can also pass the URL inline in the prompt across all SDKs — `{ role: "user", content: "Transcribe https://example.com/audio.mp3" }` — which is slightly faster and raises the size cap from 20 MB to 80 MB.

MIME types: MP3 → `audio/mpeg`, WAV → `audio/wav`, M4A → `audio/mp4`. Other formats use their canonical IANA MIME type.

The Interfaze SDK's `inputs.audio(url)` / `inputs.file(...)` helpers build these parts, and `inputs.fromPath()` / `inputs.from_path()` reads a local file.

### Example: basic transcription

#### TypeScript — Interfaze SDK

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

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const STTSchema = z.object({ text: z.string() });

const { output, providerMetadata } = await generateText({
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

const precontext = providerMetadata?.interfaze?.precontext as { result?: unknown }[] | undefined;
console.log("STT Results:", precontext?.[0]?.result);
```

#### TypeScript — LangChain SDK

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

#### Python — Interfaze SDK

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

#### Python — LangChain SDK

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

### Example: speaker diarization

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

### Example: audio translation

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

### Example: audio summary and intent

Add derived fields alongside the transcript and the model fills them in one pass.

```ts
const STTSchema = z.object({
  text: z.string(),
  summary: z.string().describe("two line summary of the audio"),
  intent: z.string().describe("3 word intent of the audio"),
});
```

### Plain transcription (no schema)

Drop the `response_format` / `output` argument when you only need raw text.

### Tips

- Use canonical MIME types: `audio/mpeg` (MP3), `audio/wav` (WAV), `audio/mp4` (M4A).
- Include "identify the speakers" in the prompt to trigger diarization.
- Specify the target language in the prompt for translation.
- Low-quality audio is de-noised automatically before transcription.
- 100+ languages are supported, including mixed-language audio.

### Advanced: long audio & word-level timestamps

For long audio (1hr+) or the lowest cost and latency, run transcription on its own. This activates only the audio part of the model — a 1hr 35min file transcribes in about 50 seconds. Ask for transcription "by word" to get word-level timestamps.

#### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.transcribe("https://example.com/audio.mp3");

console.log(result);
```

#### TypeScript — Vercel AI SDK

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

#### TypeScript — LangChain SDK

```ts
import { HumanMessage, SystemMessage } from "@langchain/core/messages";

const response = await interfaze.invoke([
  new SystemMessage("<task>speech_to_text</task>"),
  new HumanMessage("Transcribe the audio file https://example.com/audio.mp3"),
]);

const { result } = JSON.parse(response.content as string);
console.log(result);
```

#### Python — Interfaze SDK

```python
result = interfaze.tasks.transcribe("https://example.com/audio.mp3")

print(result)
```

#### Python — LangChain SDK

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

## Structured Output with Interfaze AI

Generate outputs into strict, schema-validated JSON using Zod (TypeScript) or Pydantic (Python) schemas.

### When to use this skill

- The user needs output in a specific JSON shape or typed format
- The task requires extracting structured fields from unstructured text
- The user says "give me JSON", "parse this into an object", "extract these fields"
- You need to validate AI output against a schema before using it downstream
- The output will be consumed by code, not displayed to a human

### When not to use this skill

- The user wants free-form text, summaries, or conversational responses
- The input is an image that needs OCR first → use `ocr`, then apply structured output
- The input is audio → use `speech-to-text` first, then apply structured output if needed

### Workflow

1. Identify the target output shape from the user's request.
2. Define a Zod (TypeScript) or Pydantic (Python) schema matching the expected structure.
3. Pass the schema via the SDK's structured-output mechanism:
   - Interfaze SDK: `responseFormat(z.toJSONSchema(Schema), "name")` (TS) or `completions.parse(response_format=Model)` (Python)
   - Vercel AI SDK: `generateText({ output: Output.object({ schema }) })`
   - LangChain SDK: `withStructuredOutput` / `with_structured_output`

### Rules that affect every schema

- Keep the schema root an `object`. A non-object root gets wrapped under a `result` key.
- With the Interfaze SDK in TypeScript and the OpenAI SDK, `message.content` comes back as a JSON string — parse it. The Python `parse()` helper, Vercel AI SDK, and LangChain all hand back a decoded object.
- `strict` defaults to `false`. Interfaze follows the OpenAI structured output specification.

### Example: generate a structured object from a prompt

#### TypeScript — Interfaze SDK

```ts
import { responseFormat } from "interfaze";
import { z } from "zod";

const CodeSchema = z.object({
  code: z.string(),
  sample_input: z.string(),
  sample_output: z.string(),
});

const response = await interfaze.chat.completions.create({
  messages: [
    { role: "user", content: "Write a Python script to check CPU type using subprocess." },
  ],
  response_format: responseFormat(z.toJSONSchema(CodeSchema), "code_schema"),
});

console.log(JSON.parse(response.choices[0]?.message.content ?? "{}"));
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const CodeSchema = z.object({
  code: z.string(),
  sample_input: z.string(),
  sample_output: z.string(),
});

const { output } = await generateText({
  model: interfaze.chat("interfaze"),
  output: Output.object({ schema: CodeSchema }),
  prompt: "Write a Python script to check CPU type using subprocess.",
});

console.log(output);
```

#### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const CodeSchema = z.object({
  code: z.string(),
  sample_input: z.string(),
  sample_output: z.string(),
});

const structuredModel = interfaze.withStructuredOutput(CodeSchema);

const response = await structuredModel.invoke(
  "Write a Python script to check CPU type using subprocess."
);

console.log(response);
```

#### Python — Interfaze SDK

```python
from pydantic import BaseModel

class CodeSchema(BaseModel):
    code: str
    sample_input: str
    sample_output: str

response = interfaze.chat.completions.parse(
    messages=[
        {"role": "user", "content": "Write a Python script to check CPU type using subprocess."},
    ],
    response_format=CodeSchema,
)

print(response.choices[0].message.parsed)
```

#### Python — LangChain SDK

```python
from pydantic import BaseModel

class CodeSchema(BaseModel):
    code: str
    sample_input: str
    sample_output: str

structured_llm = interfaze.with_structured_output(CodeSchema)

response = structured_llm.invoke("Write a Python script to check CPU type using subprocess.")

print(response)
```

### Example: array of structured objects

Wrap the list in an object rather than using an array at the root.

```ts
const LanguagesSchema = z.object({
  languages: z.array(
    z.object({
      name: z.string(),
      rank: z.number(),
      primary_use_case: z.string(),
    })
  ),
});
```
```python
class Language(BaseModel):
    name: str
    rank: int
    primary_use_case: str

class LanguagesSchema(BaseModel):
    languages: list[Language]
```

### Schema design tips

- Use `.nullable()` (Zod) or `Optional[T] = None` (Pydantic) for fields that may not be present.
- Use `.describe()` (Zod) or `Field(description=...)` (Pydantic) to clarify ambiguous fields — the model reads these descriptions.
- Keep schemas focused. Prefer multiple targeted calls over one massive schema.

### Schema patterns

Basic objects and arrays are shown above. Other common shapes:

#### Nullable fields

Use when a field may not exist in the source data.

```ts
z.object({
  name: z.string(),
  email: z.string().nullable(),
  phone: z.string().nullable(),
})
```
```python
from typing import Optional

class Contact(BaseModel):
    name: str
    email: Optional[str] = None
    phone: Optional[str] = None
```

#### Described fields

Use `.describe()` (Zod) or `Field(description=...)` (Pydantic) when the field name alone is ambiguous — the model reads these.

```ts
z.object({
  items: z.array(z.object({ name: z.string(), price: z.string() })),
  highlighted_items: z.array(
    z.object({ name: z.string(), price: z.string() })
  ).describe("items that are visually highlighted or marked in the source"),
})
```
```python
from pydantic import Field

class Item(BaseModel):
    name: str
    price: str

class ReceiptSchema(BaseModel):
    items: list[Item]
    highlighted_items: list[Item] = Field(
        ..., description="items that are visually highlighted or marked in the source"
    )
```

#### Nested coordinates (spatial extraction)

```ts
z.object({
  text_in_original_language: z.string(),
  bounds: z.array(
    z.object({
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ).describe("bounding box for each line"),
})
```
```python
from pydantic import Field

class Bound(BaseModel):
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class SpatialSchema(BaseModel):
    text_in_original_language: str
    bounds: list[Bound] = Field(..., description="bounding box for each line")
```

### Response handling

- **Interfaze SDK (TS):** parse `response.choices[0].message.content`.
- **Interfaze SDK (Python):** read `response.choices[0].message.parsed`.
- **Vercel AI SDK:** destructure `output` from `generateText`.
- **LangChain SDK:** the call returns the typed object directly. Pass `includeRaw: true` / `include_raw=True` to get `{ parsed, raw }` instead.

### Reading precontext alongside your schema

When Interfaze runs a task to answer the request — OCR, web search, transcription — the raw output of that task comes back as `precontext` next to your structured object. Use it for confidence scores, bounding boxes, timestamps, and source URLs.

```ts
// Interfaze SDK
response.precontext?.[0]?.result;

// Vercel AI SDK (from generateText) — JSON-typed, so cast it
(providerMetadata?.interfaze?.precontext as { result?: unknown }[] | undefined);

// LangChain SDK — requires includeRaw
response.raw.response_metadata.precontext;
```
```python
# Interfaze SDK
response.precontext[0].result if response.precontext else None

# LangChain SDK — requires include_raw=True
response["raw"].response_metadata.get("precontext")
```

### Streaming structured output

Use `streamText` with `partialOutputStream` (Vercel AI SDK) or `interfaze.stream(...)` (LangChain) when you want partial objects as they are produced.

## Object Detection with Interfaze AI

Detect and locate objects in images with bounding box coordinates using Interfaze AI.

### When to use this skill

- The user wants to find or locate objects in an image
- The user needs bounding box coordinates for detected objects
- The user wants to count specific items in a photo
- The user says "find the X in this image", "where is the Y", "what objects are here"
- The task involves spatial understanding — positions, regions, or layout of objects
- The user wants both object positions and any visible text with coordinates
- The user wants to locate UI elements in a screenshot for computer-use or UI automation

### When not to use this skill

- The user only wants to read text from an image → use `ocr`
- The user wants to transcribe audio → use `speech-to-text`
- The input is not an image
- The user wants image classification without location data
- The user wants to scrape a URL → use `web-scraping`

### Workflow

1. Confirm the input is an image URL or base64-encoded image.
2. Determine what objects the user wants to detect, or if they want all objects.
3. Build a schema with object names and bounding box fields (`top_left_x`, `top_left_y`, `bottom_right_x`, `bottom_right_y`).
4. Call the chosen SDK with the image and a prompt describing what to detect.
5. Return the detected objects with their positions.

### Input formats

Supported sources: public HTTPS URLs and base64 data URLs.

| SDK | Image part |
| --- | --- |
| Interfaze / LangChain (TS & Py) | `{ type: "image_url", image_url: { url: "<url>" } }` |
| Vercel AI SDK | `{ type: "image", mediaType: "image/png", image: "<url-or-base64>" }` |

### Example: detect objects and text in an image

#### TypeScript — Interfaze SDK

```ts
import { responseFormat } from "interfaze";
import { z } from "zod";

const DetectionSchema = z.object({
  objects: z.array(
    z.object({
      name: z.string().describe("describe the object in the image"),
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ),
  texts: z
    .array(
      z.object({
        text: z.string(),
        top_left_x: z.number(),
        top_left_y: z.number(),
        bottom_right_x: z.number(),
        bottom_right_y: z.number(),
      })
    )
    .describe("any alphabetic characters text in the image"),
});

const response = await interfaze.chat.completions.create({
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Get the position of the crane in the image and any text" },
        { type: "image_url", image_url: { url: "https://example.com/construction.png" } },
      ],
    },
  ],
  response_format: responseFormat(z.toJSONSchema(DetectionSchema), "detection_schema"),
});

console.log(JSON.parse(response.choices[0]?.message.content ?? "{}"));
console.log("Object Detection Results:", response.precontext?.[0]?.result);
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const DetectionSchema = z.object({
  objects: z.array(
    z.object({
      name: z.string().describe("describe the object in the image"),
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ),
  texts: z
    .array(
      z.object({
        text: z.string(),
        top_left_x: z.number(),
        top_left_y: z.number(),
        bottom_right_x: z.number(),
        bottom_right_y: z.number(),
      })
    )
    .describe("any alphabetic characters text in the image"),
});

const { output, providerMetadata } = await generateText({
  model: interfaze.chat("interfaze"),
  output: Output.object({ schema: DetectionSchema }),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Get the position of the crane in the image and any text" },
        { type: "image", mediaType: "image/png", image: "https://example.com/construction.png" },
      ],
    },
  ],
});

console.log(output);

const precontext = providerMetadata?.interfaze?.precontext as { result?: unknown }[] | undefined;
console.log("Object Detection Results:", precontext?.[0]?.result);
```

#### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const DetectionSchema = z.object({
  objects: z.array(
    z.object({
      name: z.string().describe("describe the object in the image"),
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ),
  texts: z
    .array(
      z.object({
        text: z.string(),
        top_left_x: z.number(),
        top_left_y: z.number(),
        bottom_right_x: z.number(),
        bottom_right_y: z.number(),
      })
    )
    .describe("any alphabetic characters text in the image"),
});

const structuredModel = interfaze.withStructuredOutput(DetectionSchema, { includeRaw: true });

const response = await structuredModel.invoke([
  {
    role: "user",
    content: [
      { type: "text", text: "Get the position of the crane in the image and any text" },
      { type: "image_url", image_url: { url: "https://example.com/construction.png" } },
    ],
  },
]);

console.log(response.parsed);
console.log("Object Detection Results:", response.raw.response_metadata.precontext?.[0]?.result);
```

#### Python — Interfaze SDK

```python
from typing import List
from pydantic import BaseModel, Field

class DetectedObject(BaseModel):
    name: str = Field(..., description="describe the object in the image")
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class DetectedText(BaseModel):
    text: str
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class DetectionSchema(BaseModel):
    objects: List[DetectedObject]
    texts: List[DetectedText] = Field(..., description="any alphabetic characters text in the image")

response = interfaze.chat.completions.parse(
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Get the position of the crane in the image and any text"},
                {"type": "image_url", "image_url": {"url": "https://example.com/construction.png"}},
            ],
        }
    ],
    response_format=DetectionSchema,
)

print(response.choices[0].message.parsed)
print("Object Detection Results:", response.precontext[0].result if response.precontext else None)
```

#### Python — LangChain SDK

```python
from typing import List
from langchain_core.messages import HumanMessage
from pydantic import BaseModel, Field

class DetectedObject(BaseModel):
    name: str = Field(..., description="describe the object in the image")
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class DetectedText(BaseModel):
    text: str
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class DetectionSchema(BaseModel):
    objects: List[DetectedObject]
    texts: List[DetectedText] = Field(..., description="any alphabetic characters text in the image")

structured_llm = interfaze.with_structured_output(DetectionSchema, include_raw=True)

response = structured_llm.invoke([
    HumanMessage(content=[
        {"type": "text", "text": "Get the position of the crane in the image and any text"},
        {"type": "image_url", "image_url": {"url": "https://example.com/construction.png"}},
    ])
])

print(response["parsed"])
print("Object Detection Results:", response["raw"].response_metadata.get("precontext"))
```

### Example: detect specific object categories

Same call — swap in a schema with a `category` field.

```ts
const TrafficSchema = z.object({
  objects: z.array(
    z.object({
      name: z.string(),
      category: z.string().describe("vehicle or pedestrian"),
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ),
});
```
```python
class TrafficObject(BaseModel):
    name: str
    category: str = Field(..., description="vehicle or pedestrian")
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class TrafficSchema(BaseModel):
    objects: list[TrafficObject]
```

### Tips

- Be specific in the prompt about what to detect. "Find all vehicles" works better than "detect objects."
- Use `.describe()` / `Field(description=...)` on the name field to guide labelling.
- Add a `category` field to group objects by type.
- Coordinates are returned in pixels relative to the original image dimensions.
- Detection runs alongside OCR when the prompt asks for both, producing two `precontext` entries.

### Advanced: raw detection output

Run detection on its own for a fixed output shape without a custom schema. Faster and cheaper than a schema-driven call.

#### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.objectDetection("https://example.com/construction.png");

console.log(result);
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";

const { output } = await generateText({
  model: interfaze.chat("interfaze"),
  system: "<task>object_detection</task>",
  output: Output.json(),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Detect objects in this image" },
        { type: "image", mediaType: "image/png", image: "https://example.com/construction.png" },
      ],
    },
  ],
});

console.log(output);
```

#### TypeScript — LangChain SDK

```ts
import { HumanMessage, SystemMessage } from "@langchain/core/messages";

const response = await interfaze.invoke([
  new SystemMessage("<task>object_detection</task>"),
  new HumanMessage({
    content: [
      { type: "text", text: "Detect objects in this image" },
      { type: "image_url", image_url: { url: "https://example.com/construction.png" } },
    ],
  }),
]);

const { result } = JSON.parse(response.content as string);
console.log(result);
```

#### Python — Interfaze SDK

```python
result = interfaze.tasks.object_detection("https://example.com/construction.png")

print(result)
```

#### Python — LangChain SDK

```python
import json
from langchain_core.messages import HumanMessage, SystemMessage

response = interfaze.invoke([
    SystemMessage(content="<task>object_detection</task>"),
    HumanMessage(content=[
        {"type": "text", "text": "Detect objects in this image"},
        {"type": "image_url", "image_url": {"url": "https://example.com/construction.png"}},
    ]),
])

print(json.loads(response.content).get("result"))
```

Task mode rules: only one `<task>` runs per request, and the response format must be an empty or `any` schema — passing a real schema alongside a task returns a 400.

The result shape:

```json
{
  "name": "object_detection",
  "result": {
    "detected_objects": [
      {
        "label": "crane",
        "bounds": {
          "top_left": { "x": 630, "y": 139 },
          "bottom_right": { "x": 769, "y": 225 },
          "width": 139,
          "height": 86
        }
      }
    ],
    "gui_elements": [
      { "type": "text", "content": "tower", "interactivity": false, "bounds": {} }
    ]
  }
}
```

### Advanced: GUI detection for computer use

To locate interactive UI elements in a screenshot — buttons, inputs, links, dropdowns — use the `gui_detection` task instead. It returns each element's type and bounding box, which is what you want for computer-use and UI automation agents.

```ts
const result = await interfaze.tasks.guiDetection("https://example.com/screenshot.jpg");
```
```python
result = interfaze.tasks.gui_detection("https://example.com/screenshot.jpg")
```

With other SDKs, set `<task>gui_detection</task>` as the system message and pass an empty schema, exactly as shown for `object_detection` above. The result shape:

```json
{
  "name": "gui_detection",
  "result": {
    "gui_elements": [
      {
        "type": "button",
        "top_left_x": 120,
        "top_left_y": 340,
        "bottom_right_x": 260,
        "bottom_right_y": 380
      }
    ]
  }
}
```

## Web Search with Interfaze AI

Search the web and retrieve current information using Interfaze AI's built-in web search index. No tool config required — send a prompt that needs current information and the model searches as needed.

### When to use this skill

- The user needs current or real-time information that isn't in the model's training data
- The user asks about prices, availability, news, events, or recent developments
- The user says "search for", "look up", "find out about", "what's the latest"
- The task requires verifying facts against current web sources
- The user provides a topic and needs comprehensive, up-to-date information

### When not to use this skill

- The question can be answered from the model's existing knowledge (e.g. "what is 2 + 2", "explain `let` vs `const`")
- The user wants to extract data from a specific known URL → use `web-scraping`
- The user has an image or audio to process → use `ocr`, `object-detection`, or `speech-to-text`

### Workflow

1. Identify what information the user needs from the web.
2. Formulate a clear, specific search prompt.
3. Use a structured-output call for typed results, or a plain text call for a natural-language summary.
4. The model returns results directly; raw hits are available via `precontext`.

### Example: basic web search (text summary)

#### TypeScript — Interfaze SDK

```ts
const response = await interfaze.chat.completions.create({
  messages: [{ role: "user", content: "Latest news on Nvidia" }],
});

console.log(response.choices[0]?.message.content);
console.log("Web Search Results:", response.precontext?.[0]?.result);
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText } from "ai";

const { text, providerMetadata } = await generateText({
  model: interfaze.chat("interfaze"),
  prompt: "Latest news on Nvidia",
});

console.log(text);

const precontext = providerMetadata?.interfaze?.precontext as { result?: unknown }[] | undefined;
console.log("Web Search Results:", precontext?.[0]?.result);
```

#### TypeScript — LangChain SDK

```ts
const response = await interfaze.invoke("Latest news on Nvidia");

console.log(response.content);
console.log("Web Search Results:", response.response_metadata.precontext?.[0]?.result);
```

#### Python — Interfaze SDK

```python
response = interfaze.chat.completions.create(
    messages=[{"role": "user", "content": "Latest news on Nvidia"}],
)

print(response.choices[0].message.content)
print("Web Search Results:", response.precontext[0].result if response.precontext else None)
```

#### Python — LangChain SDK

```python
response = interfaze.invoke("Latest news on Nvidia")

print(response.content)
print("Web Search Results:", response.response_metadata.get("precontext"))
```

### Example: structured web search

#### TypeScript — Interfaze SDK

```ts
import { responseFormat } from "interfaze";
import { z } from "zod";

const NvidiaNewsSchema = z.object({
  summary: z.string(),
  current_stock_price: z.number(),
  links: z.array(z.string()),
});

const response = await interfaze.chat.completions.create({
  messages: [{ role: "user", content: "Latest news on Nvidia" }],
  response_format: responseFormat(z.toJSONSchema(NvidiaNewsSchema), "nvidia_news_schema"),
});

console.log(JSON.parse(response.choices[0]?.message.content ?? "{}"));
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const NvidiaNewsSchema = z.object({
  summary: z.string(),
  current_stock_price: z.number(),
  links: z.array(z.string()),
});

const { output } = await generateText({
  model: interfaze.chat("interfaze"),
  output: Output.object({ schema: NvidiaNewsSchema }),
  prompt: "Latest news on Nvidia",
});

console.log(output);
```

#### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const NvidiaNewsSchema = z.object({
  summary: z.string(),
  current_stock_price: z.number(),
  links: z.array(z.string()),
});

const structuredModel = interfaze.withStructuredOutput(NvidiaNewsSchema, { includeRaw: true });

const response = await structuredModel.invoke("Latest news on Nvidia");

console.log(response.parsed);
```

#### Python — Interfaze SDK

```python
from typing import List
from pydantic import BaseModel

class NvidiaNewsSchema(BaseModel):
    summary: str
    current_stock_price: float
    links: List[str]

response = interfaze.chat.completions.parse(
    messages=[{"role": "user", "content": "Latest news on Nvidia"}],
    response_format=NvidiaNewsSchema,
)

print(response.choices[0].message.parsed)
```

#### Python — LangChain SDK

```python
from typing import List
from pydantic import BaseModel

class NvidiaNewsSchema(BaseModel):
    summary: str
    current_stock_price: float
    links: List[str]

structured_llm = interfaze.with_structured_output(NvidiaNewsSchema, include_raw=True)

response = structured_llm.invoke("Latest news on Nvidia")

print(response["parsed"])
```

For structured queries, Interfaze often follows the search with a scrape pass over the top results to get higher-quality data. That shows up as an extra `precontext` entry.

### Example: people and company enrichment

Same call — use a schema shaped for the entity.

```ts
const PersonSchema = z.object({
  summary: z.string(),
  company: z.string().optional(),
  emails: z.array(z.string()).describe("all unmasked email addresses").optional(),
  location: z.string().describe("City and country").optional(),
});

// prompt: "Who is Yoeven D Khemlani, his company, his email and where is he based now?"
```
```python
from typing import List, Optional

class PersonSchema(BaseModel):
    summary: str
    company: Optional[str] = None
    emails: Optional[List[str]] = Field(None, description="all unmasked email addresses")
    location: Optional[str] = Field(None, description="City and country")
```

### Raw results via precontext

Raw search hits are returned on the response's `precontext` array. Each hit has `title`, `description`, `content`, `snippets`, and `url`.

```ts
// Interfaze SDK
response.precontext?.[0]?.result;

// Vercel AI SDK (from generateText) — JSON-typed, so cast it
(providerMetadata?.interfaze?.precontext as { result?: unknown }[] | undefined)?.[0]?.result;

// LangChain SDK
response.response_metadata.precontext?.[0]?.result;
```

### Advanced: search-only, no model response

Run the search on its own to skip the model's summarization pass. Faster and cheaper when you just want the hits.

#### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.webSearch("GLP-1 research paper");

console.log("Web Search Results:", result);
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";

const { output } = await generateText({
  model: interfaze.chat("interfaze"),
  system: "<task>web_search</task>",
  output: Output.json(),
  prompt: "GLP-1 research paper",
});

console.log(output);
```

#### TypeScript — LangChain SDK

```ts
import { HumanMessage, SystemMessage } from "@langchain/core/messages";

const response = await interfaze.invoke([
  new SystemMessage("<task>web_search</task>"),
  new HumanMessage("GLP-1 research paper"),
]);

const { result } = JSON.parse(response.content as string);
console.log("Web Search Results:", result);
```

#### Python — Interfaze SDK

```python
result = interfaze.tasks.web_search("GLP-1 research paper")

print("Web Search Results:", result)
```

#### Python — LangChain SDK

```python
import json
from langchain_core.messages import HumanMessage, SystemMessage

response = interfaze.invoke([
    SystemMessage(content="<task>web_search</task>"),
    HumanMessage(content="GLP-1 research paper"),
])

print("Web Search Results:", json.loads(response.content).get("result"))
```

Task mode rules: only one `<task>` runs per request, and the response format must be an empty or `any` schema — passing a real schema alongside a task returns a 400.

The result shape:

```json
{
  "name": "web_search",
  "result": [
    {
      "title": "Latest News | NVIDIA Newsroom",
      "description": "NVIDIA and Emerald AI today announced ...",
      "content": "Latest News\nMedia Contacts Stay Informed\n...",
      "url": "https://nvidianews.nvidia.com/news/latest"
    }
  ]
}
```

### Coverage

Interfaze's web search covers research indexes (biology, finance, medicine, law, arxiv), social platforms (LinkedIn, X, personal sites), financial data (stocks, forex, crypto, commodities, SEC filings), and general search (news, events, products, company info).

### Tips

- Be specific. "Find the price of X on Amazon" works better than "look up X."
- Use structured output when you need machine-readable results.
- A web search is triggered automatically when the prompt requires current information.
- To normalize results further, combine with `structured-output`.

## Web Scraping with Interfaze AI

Extract structured data from web pages by including a URL in the prompt and defining a schema for the desired output. Interfaze has built-in residential proxies, auto-scaling browser infrastructure, and bot-block handling — no separate scraping library required.

### When to use this skill

- The user provides a specific URL and wants data extracted from it
- The task involves pulling product listings, prices, profiles, or structured content from a webpage
- The user says "scrape this page", "get the data from this URL", "extract products from this site"
- The task requires pulling data from an e-commerce site, job board, directory, or social profile

### When not to use this skill

- The user wants to search the web without a specific URL → use `web-search`
- The user has an image → use `ocr` or `object-detection`
- The user has audio → use `speech-to-text`
- The user wants to reshape data they already have → use `structured-output`
- The URL points to a downloadable file rather than a web page

### Workflow

1. Confirm the target URL.
2. Define a schema (Zod for TypeScript, Pydantic for Python) matching the data the user wants.
3. Include the URL inline in the prompt; the model fetches and parses the page.
4. Return the structured data.

### Example: scrape product listings

The URL goes inline in the prompt — there is no separate URL parameter.

#### TypeScript — Interfaze SDK

```ts
import { responseFormat } from "interfaze";
import { z } from "zod";

const ProductSchema = z.object({
  price: z.number(),
  listing_name: z.string(),
});

const response = await interfaze.chat.completions.create({
  messages: [
    {
      role: "user",
      content: "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL",
    },
  ],
  response_format: responseFormat(z.toJSONSchema(ProductSchema), "product_schema"),
});

console.log(JSON.parse(response.choices[0]?.message.content ?? "{}"));
console.log("Web Scrape Results:", response.precontext?.[0]?.result);
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const ProductSchema = z.object({
  price: z.number(),
  listing_name: z.string(),
});

const { output, providerMetadata } = await generateText({
  model: interfaze.chat("interfaze"),
  output: Output.object({ schema: ProductSchema }),
  prompt: "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL",
});

console.log(output);

const precontext = providerMetadata?.interfaze?.precontext as { result?: unknown }[] | undefined;
console.log("Web Scrape Results:", precontext?.[0]?.result);
```

#### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const ProductSchema = z.object({
  price: z.number(),
  listing_name: z.string(),
});

const structuredModel = interfaze.withStructuredOutput(ProductSchema, { includeRaw: true });

const response = await structuredModel.invoke(
  "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL"
);

console.log(response.parsed);
console.log("Web Scrape Results:", response.raw.response_metadata.precontext?.[0]?.result);
```

#### Python — Interfaze SDK

```python
from pydantic import BaseModel

class ProductSchema(BaseModel):
    price: float
    listing_name: str

response = interfaze.chat.completions.parse(
    messages=[
        {
            "role": "user",
            "content": "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL",
        }
    ],
    response_format=ProductSchema,
)

print(response.choices[0].message.parsed)
print("Web Scrape Results:", response.precontext[0].result if response.precontext else None)
```

#### Python — LangChain SDK

```python
from pydantic import BaseModel

class ProductSchema(BaseModel):
    price: float
    listing_name: str

structured_llm = interfaze.with_structured_output(ProductSchema, include_raw=True)

response = structured_llm.invoke(
    "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL"
)

print(response["parsed"])
print("Web Scrape Results:", response["raw"].response_metadata.get("precontext"))
```

### Example: scrape a social profile

Same call — swap in the schema.

```ts
const LinkedInProfileSchema = z.object({
  first_name: z.string(),
  last_name: z.string(),
  location: z.string(),
  latest_education: z.string(),
  current_job: z.string(),
  followers: z.number(),
});
// prompt: "Extract information from https://www.linkedin.com/in/example-user/"
```
```python
class LinkedInProfileSchema(BaseModel):
    first_name: str
    last_name: str
    location: str
    latest_education: str
    current_job: str
    followers: int
```

### Example: scrape an array of listings

Wrap the list in an object — a non-object schema root gets wrapped under a `result` key.

```ts
const ArticlesSchema = z.object({
  articles: z.array(
    z.object({
      title: z.string(),
      url: z.string(),
      points: z.number().nullable(),
    })
  ),
});
// prompt: "Extract all article titles, URLs, and points from https://news.ycombinator.com"
```
```python
class Article(BaseModel):
    title: str
    url: str
    points: Optional[int] = None

class ArticlesSchema(BaseModel):
    articles: list[Article]
```

### Tips

- Pass the URL inline in the prompt. No separate URL parameter is needed.
- Be specific: "get all product names and prices" works better than "scrape this page".
- Use `.describe()` / `Field(description=...)` for ambiguous field names.
- Residential proxies, browser infrastructure, and bot-block handling are built in — no extra setup.
- The scraped page content lands in `precontext` under `scraped_content`, `scraped_elements`, and `meta`, including the CSS selectors used for each field. Use it to debug a field the model got wrong.

### Advanced: raw scraping output

Run the scraper on its own to skip the model's extraction pass and get the page content in a fixed shape. Faster and cheaper when you don't need a custom schema.

#### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.scrape("https://news.ycombinator.com");

console.log("Web Scrape Results:", result);
```

#### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";

const { output } = await generateText({
  model: interfaze.chat("interfaze"),
  system: "<task>scraper</task>",
  output: Output.json(),
  prompt: "Extract post titles and points from https://news.ycombinator.com",
});

console.log(output);
```

#### TypeScript — LangChain SDK

```ts
import { HumanMessage, SystemMessage } from "@langchain/core/messages";

const response = await interfaze.invoke([
  new SystemMessage("<task>scraper</task>"),
  new HumanMessage("Extract post titles and points from https://news.ycombinator.com"),
]);

const { result } = JSON.parse(response.content as string);
console.log("Web Scrape Results:", result);
```

#### Python — Interfaze SDK

```python
result = interfaze.tasks.scrape("https://news.ycombinator.com")

print("Web Scrape Results:", result)
```

#### Python — LangChain SDK

```python
import json
from langchain_core.messages import HumanMessage, SystemMessage

response = interfaze.invoke([
    SystemMessage(content="<task>scraper</task>"),
    HumanMessage(content="Extract post titles and points from https://news.ycombinator.com"),
])

print("Web Scrape Results:", json.loads(response.content).get("result"))
```

Task mode rules: only one `<task>` runs per request, and the response format must be an empty or `any` schema — passing a real schema alongside a task returns a 400.

The result shape:

```json
{
  "name": "ai_scraper",
  "result": {
    "scraped_content": { "title": ["...", "..."], "points": ["...", "..."] },
    "scraped_elements": [
      {
        "selector": "span.titleline > a",
        "key": "title",
        "results": [{ "html": "...", "text": "...", "attributes": [{ "name": "href", "value": "..." }] }]
      }
    ],
    "meta": { "title": "Hacker News" }
  }
}
```

Note that `scraped_content` values are arrays of strings keyed by your field names. Scrapes that happen mid-request as part of a larger answer appear in `precontext` under the name `web_extract` with the same result shape.
