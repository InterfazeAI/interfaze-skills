---
name: web-scraping
description: >
  Use Interfaze AI to scrape and extract structured data from specific web pages and
  URLs. Activate when the user provides a URL and wants to pull products, listings,
  prices, profiles, or any structured content from that page — even if they say
  "get the data from this page", "pull the listings from this URL", or "extract
  products from this site" without explicitly mentioning scraping.
---

# Web Scraping with Interfaze AI

Extract structured data from web pages by including a URL in the prompt and defining a schema for the desired output. Interfaze has built-in residential proxies, auto-scaling browser infrastructure, and bot-block handling — no separate scraping library required.

## When to use this skill

- The user provides a specific URL and wants data extracted from it
- The task involves pulling product listings, prices, profiles, or structured content from a webpage
- The user says "scrape this page", "get the data from this URL", "extract products from this site"
- The task requires pulling data from an e-commerce site, job board, directory, or social profile

## When not to use this skill

- The user wants to search the web without a specific URL → use `web-search`
- The user has an image → use `ocr` or `object-detection`
- The user has audio → use `speech-to-text`
- The user wants to reshape data they already have → use `structured-output`
- The URL points to a downloadable file rather than a web page

## Workflow

1. Confirm the target URL.
2. Define a schema (Zod for TypeScript, Pydantic for Python) matching the data the user wants.
3. Include the URL inline in the prompt; the model fetches and parses the page.
4. Return the structured data.

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

Requires `ai@^6.0.0` and `@ai-sdk/openai@^3.0.0`.

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

Point an existing OpenAI client at Interfaze and pass `model: "interfaze-beta"` on every call.

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

## Example: scrape product listings

The URL goes inline in the prompt — there is no separate URL parameter.

### TypeScript — Interfaze SDK

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

### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const ProductSchema = z.object({
  price: z.number(),
  listing_name: z.string(),
});

const { output, response } = await generateText({
  model: interfaze.chat("interfaze-beta"),
  output: Output.object({ schema: ProductSchema }),
  prompt: "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL",
});

console.log(output);

// @ts-expect-error precontext is not typed
console.log("Web Scrape Results:", response.body?.precontext?.[0]?.result);
```

### TypeScript — LangChain SDK

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

### Python — Interfaze SDK

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

### Python — LangChain SDK

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

## Example: scrape a social profile

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

## Example: scrape an array of listings

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

## Tips

- Pass the URL inline in the prompt. No separate URL parameter is needed.
- Be specific: "get all product names and prices" works better than "scrape this page".
- Use `.describe()` / `Field(description=...)` for ambiguous field names.
- Residential proxies, browser infrastructure, and bot-block handling are built in — no extra setup.
- The scraped page content lands in `precontext` under `scraped_content`, `scraped_elements`, and `meta`, including the CSS selectors used for each field. Use it to debug a field the model got wrong.

## Advanced: raw scraping output

Run the scraper on its own to skip the model's extraction pass and get the page content in a fixed shape. Faster and cheaper when you don't need a custom schema.

### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.scrape("https://news.ycombinator.com");

console.log("Web Scrape Results:", result);
```

### TypeScript — Vercel AI SDK

```ts
import { generateText, Output } from "ai";
import { z } from "zod";

const { output } = await generateText({
  model: interfaze.chat("interfaze-beta"),
  system: "<task>scraper</task>",
  output: Output.object({ schema: z.any() }),
  prompt: "Extract post titles and points from https://news.ycombinator.com",
});

console.log(output);
```

### TypeScript — LangChain SDK

```ts
import { HumanMessage, SystemMessage } from "@langchain/core/messages";

const response = await interfaze.invoke([
  new SystemMessage("<task>scraper</task>"),
  new HumanMessage("Extract post titles and points from https://news.ycombinator.com"),
]);

const { result } = JSON.parse(response.content as string);
console.log("Web Scrape Results:", result);
```

### Python — Interfaze SDK

```python
result = interfaze.tasks.scrape("https://news.ycombinator.com")

print("Web Scrape Results:", result)
```

### Python — LangChain SDK

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
