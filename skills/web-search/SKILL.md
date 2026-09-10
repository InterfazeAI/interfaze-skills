---
name: web-search
description: >
  Use Interfaze AI to search the web and retrieve current information, facts, prices,
  news, or any real-time data. Activate when the user needs up-to-date information
  from the internet, wants to look something up, or asks a question that requires
  current knowledge — even if they say "look this up", "find out about", or "what's
  the latest on" without explicitly mentioning web search.
---

# Web Search with Interfaze AI

Search the web and retrieve current information using Interfaze AI's built-in web search index. No tool config required — send a prompt that needs current information and the model searches as needed.

## When to use this skill

- The user needs current or real-time information that isn't in the model's training data
- The user asks about prices, availability, news, events, or recent developments
- The user says "search for", "look up", "find out about", "what's the latest"
- The task requires verifying facts against current web sources
- The user provides a topic and needs comprehensive, up-to-date information

## When not to use this skill

- The question can be answered from the model's existing knowledge (e.g. "what is 2 + 2", "explain `let` vs `const`")
- The user wants to extract data from a specific known URL → use `web-scraping`
- The user has an image or audio to process → use `ocr`, `object-detection`, or `speech-to-text`

## Workflow

1. Identify what information the user needs from the web.
2. Formulate a clear, specific search prompt.
3. Use a structured-output call for typed results, or a plain text call for a natural-language summary.
4. The model returns results directly; raw hits are available via `precontext`.

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

## Example: basic web search (text summary)

### TypeScript — Interfaze SDK

```ts
const response = await interfaze.chat.completions.create({
  messages: [{ role: "user", content: "Latest news on Nvidia" }],
});

console.log(response.choices[0]?.message.content);
console.log("Web Search Results:", response.precontext?.[0]?.result);
```

### TypeScript — Vercel AI SDK

```ts
import { generateText } from "ai";

const { text, response } = await generateText({
  model: interfaze.chat("interfaze"),
  prompt: "Latest news on Nvidia",
});

console.log(text);

// @ts-expect-error precontext is not typed
console.log("Web Search Results:", response.body?.precontext?.[0]?.result);
```

### TypeScript — LangChain SDK

```ts
const response = await interfaze.invoke("Latest news on Nvidia");

console.log(response.content);
console.log("Web Search Results:", response.response_metadata.precontext?.[0]?.result);
```

### Python — Interfaze SDK

```python
response = interfaze.chat.completions.create(
    messages=[{"role": "user", "content": "Latest news on Nvidia"}],
)

print(response.choices[0].message.content)
print("Web Search Results:", response.precontext[0].result if response.precontext else None)
```

### Python — LangChain SDK

```python
response = interfaze.invoke("Latest news on Nvidia")

print(response.content)
print("Web Search Results:", response.response_metadata.get("precontext"))
```

## Example: structured web search

### TypeScript — Interfaze SDK

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

### TypeScript — Vercel AI SDK

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

### TypeScript — LangChain SDK

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

### Python — Interfaze SDK

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

### Python — LangChain SDK

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

## Example: people and company enrichment

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

## Raw results via precontext

Raw search hits are returned on the response's `precontext` array. Each hit has `title`, `description`, `content`, `snippets`, and `url`.

```ts
// Interfaze SDK
response.precontext?.[0]?.result;

// Vercel AI SDK
// @ts-expect-error precontext is not typed
response.body?.precontext?.[0]?.result;

// LangChain SDK
response.response_metadata.precontext?.[0]?.result;
```

## Advanced: search-only, no model response

Run the search on its own to skip the model's summarization pass. Faster and cheaper when you just want the hits.

### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.webSearch("GLP-1 research paper");

console.log("Web Search Results:", result);
```

### TypeScript — Vercel AI SDK

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

### TypeScript — LangChain SDK

```ts
import { HumanMessage, SystemMessage } from "@langchain/core/messages";

const response = await interfaze.invoke([
  new SystemMessage("<task>web_search</task>"),
  new HumanMessage("GLP-1 research paper"),
]);

const { result } = JSON.parse(response.content as string);
console.log("Web Search Results:", result);
```

### Python — Interfaze SDK

```python
result = interfaze.tasks.web_search("GLP-1 research paper")

print("Web Search Results:", result)
```

### Python — LangChain SDK

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

## Coverage

Interfaze's web search covers research indexes (biology, finance, medicine, law, arxiv), social platforms (LinkedIn, X, personal sites), financial data (stocks, forex, crypto, commodities, SEC filings), and general search (news, events, products, company info).

## Tips

- Be specific. "Find the price of X on Amazon" works better than "look up X."
- Use structured output when you need machine-readable results.
- A web search is triggered automatically when the prompt requires current information.
- To normalize results further, combine with `structured-output`.
