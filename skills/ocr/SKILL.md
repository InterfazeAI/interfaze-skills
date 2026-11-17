---
name: ocr
description: >
  Use Interfaze AI for OCR and visual text extraction from images, screenshots,
  scans, PDFs, invoices, receipts, IDs, forms, menus, and labels with complex
  layouts. Activate when the user wants to extract text, fields, or structured
  data from an image or scanned document — even if they say "read this", "what
  does this say", or "pull the data from this photo" without mentioning OCR.
---

# OCR — Visual OCR with Interfaze AI

Extract text and structured fields from images and scanned documents using Interfaze AI's vision OCR capability.

## When to use this skill

- The user has an image, screenshot, scan, or photo and wants text extracted from it
- The input is a receipt, invoice, ID card, form, menu, or label captured as an image
- The user wants structured fields (name, address, totals, line items) pulled from a visual source
- The user says "read this image", "extract the text", "what does this receipt say"
- The task involves layout-aware extraction where text position matters

## When not to use this skill

- The input is a native text file, JSON, or CSV — no vision needed
- The user wants to transcribe audio → use `speech-to-text`
- The task is locating objects without reading text → use `object-detection`
- The user wants to convert existing data to a schema → use `structured-output`
- The user wants to pull data from a live URL → use `web-scraping`

## Workflow

1. Confirm the input is an image URL, PDF URL, or base64-encoded file.
2. Decide whether you need plain text or structured fields.
3. For structured fields, define a Zod (TS) or Pydantic (Python) schema.
4. Pass the file in the message `content` array using the SDK's file/image part.
5. Match the SDK and language to the user's project.

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

## Input formats

Supported sources: public HTTPS URLs and base64 data URLs.

| SDK | Image part | PDF / document part |
| --- | --- | --- |
| Interfaze / LangChain (TS & Py) | `{ type: "image_url", image_url: { url: "<url>" } }` | `{ type: "file", file: { filename: "doc.pdf", file_data: "<url-or-base64>" } }` |
| Vercel AI SDK | `{ type: "image", mediaType: "image/jpeg", image: "<url-or-base64>" }` | `{ type: "file", data: "<url>", mediaType: "application/pdf" }` |

Size limits: 20 MB as a file part, 80 MB when the URL is written directly into the prompt text. For large documents, put the URL in the prompt instead — the model fetches it at inference time.

## Example: extract fields from an ID card

### TypeScript — Interfaze SDK

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

### TypeScript — Vercel AI SDK

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

### TypeScript — LangChain SDK

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

### Python — Interfaze SDK

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

### Python — LangChain SDK

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

## Example: receipt line items

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

## Example: PDF document

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

## Plain text (no schema)

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

## Reading local files

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

## Tips

- Use `.describe()` / `Field(description=...)` when a field name alone is ambiguous.
- For multilingual text, name the target language in the prompt or field descriptions. 100+ languages, including mixed-language documents, are supported.
- For layout-aware extraction, include coordinate fields in your schema.
- The model corrects and normalizes text in the structured output. `precontext` holds the uncorrected raw OCR, which is what you want for auditing or confidence thresholds.

## Advanced: raw OCR output & bounding boxes

Run OCR on its own to get a fixed output shape with line-level bounding boxes and per-word confidence, skipping the full model. This is faster and cheaper than a schema-driven call.

### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.ocr("https://example.com/id.jpg");

console.log(result);
```

### TypeScript — Vercel AI SDK

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

### TypeScript — LangChain SDK

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

### Python — Interfaze SDK

```python
result = interfaze.tasks.ocr("https://example.com/id.jpg")

print(result)
```

### Python — LangChain SDK

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
