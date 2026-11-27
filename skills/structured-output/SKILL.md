---
name: structured-output
description: >
  Use Interfaze AI to produce strict JSON or schema-constrained structured objects
  from any prompt. Activate when the user needs typed, machine-readable output —
  JSON objects, arrays, validated fields, or Zod/Pydantic-style extraction. Use this
  when the task requires a specific output shape, data normalization, or converting
  unstructured text into structured records, even if the user just says "give me this
  as JSON" or "parse this into fields."
---

# Structured Output with Interfaze AI

Generate outputs into strict, schema-validated JSON using Zod (TypeScript) or Pydantic (Python) schemas.

## When to use this skill

- The user needs output in a specific JSON shape or typed format
- The task requires extracting structured fields from unstructured text
- The user says "give me JSON", "parse this into an object", "extract these fields"
- You need to validate AI output against a schema before using it downstream
- The output will be consumed by code, not displayed to a human

## When not to use this skill

- The user wants free-form text, summaries, or conversational responses
- The input is an image that needs OCR first → use `ocr`, then apply structured output
- The input is audio → use `speech-to-text` first, then apply structured output if needed

## Workflow

1. Identify the target output shape from the user's request.
2. Define a Zod (TypeScript) or Pydantic (Python) schema matching the expected structure.
3. Pass the schema via the SDK's structured-output mechanism:
   - Interfaze SDK: `responseFormat(z.toJSONSchema(Schema), "name")` (TS) or `completions.parse(response_format=Model)` (Python)
   - Vercel AI SDK: `generateText({ output: Output.object({ schema }) })`
   - LangChain SDK: `withStructuredOutput` / `with_structured_output`

## Rules that affect every schema

- Keep the schema root an `object`. A non-object root gets wrapped under a `result` key.
- With the Interfaze SDK in TypeScript and the OpenAI SDK, `message.content` comes back as a JSON string — parse it. The Python `parse()` helper, Vercel AI SDK, and LangChain all hand back a decoded object.
- `strict` defaults to `false`. Interfaze follows the OpenAI structured output specification.

## Setup

Every SDK reads the API key from `INTERFAZE_API_KEY`, and the SDK examples below need an API key. If `INTERFAZE_API_KEY` is unset, the Vercel AI SDK and OpenAI SDK clients read `OPENAI_API_KEY` instead and send it to Interfaze.

To try Interfaze without an API key, send a direct API request from server-side code and leave out the `Authorization` header. Interfaze limits keyless requests from each IP address and does not accept them from a browser. See [Try Interfaze without an API key](https://interfaze.ai/docs#try-interfaze-without-an-api-key) and [keyless limits](https://interfaze.ai/docs/limits#keyless-access).

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

Point an existing OpenAI client at Interfaze and pass `model: "interfaze"` on every call. Use `zodResponseFormat(Schema, "schema_name")` from `openai/helpers/zod` in TypeScript, or a `json_schema` response format built from `Model.model_json_schema()` in Python.

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

## Example: generate a structured object from a prompt

### TypeScript — Interfaze SDK

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

### TypeScript — Vercel AI SDK

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

### TypeScript — LangChain SDK

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

### Python — Interfaze SDK

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

### Python — LangChain SDK

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

## Example: array of structured objects

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

## Schema design tips

- Use `.nullable()` (Zod) or `Optional[T] = None` (Pydantic) for fields that may not be present.
- Use `.describe()` (Zod) or `Field(description=...)` (Pydantic) to clarify ambiguous fields — the model reads these descriptions.
- Keep schemas focused. Prefer multiple targeted calls over one massive schema.

## Schema patterns

Basic objects and arrays are shown above. Other common shapes:

### Nullable fields

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

### Described fields

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

### Nested coordinates (spatial extraction)

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

## Response handling

- **Interfaze SDK (TS):** parse `response.choices[0].message.content`.
- **Interfaze SDK (Python):** read `response.choices[0].message.parsed`.
- **Vercel AI SDK:** destructure `output` from `generateText`.
- **LangChain SDK:** the call returns the typed object directly. Pass `includeRaw: true` / `include_raw=True` to get `{ parsed, raw }` instead.

## Reading precontext alongside your schema

When Interfaze runs a task to answer the request — OCR, web search, transcription — the raw output of that task comes back as `precontext` next to your structured object. Use it for confidence scores, bounding boxes, timestamps, and source URLs.

```ts
// Interfaze SDK
response.precontext?.[0]?.result;

// Vercel AI SDK — untyped, read off the raw body
// @ts-expect-error precontext is not typed
response.body?.precontext;

// LangChain SDK — requires includeRaw
response.raw.response_metadata.precontext;
```
```python
# Interfaze SDK
response.precontext[0].result if response.precontext else None

# LangChain SDK — requires include_raw=True
response["raw"].response_metadata.get("precontext")
```

## Streaming structured output

Use `streamText` with `partialOutputStream` (Vercel AI SDK) or `interfaze.stream(...)` (LangChain) when you want partial objects as they are produced.
