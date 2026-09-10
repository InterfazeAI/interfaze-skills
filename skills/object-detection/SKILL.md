---
name: object-detection
description: >
  Use Interfaze AI to detect, locate, and identify objects in images and PDFs with
  bounding box coordinates. Activate when the user wants to find objects, get their
  positions, count items, or locate specific things in an image — even if they say
  "find the car in this photo", "where is the logo", or "what objects are in this
  image" without explicitly mentioning object detection.
---

# Object Detection with Interfaze AI

Detect and locate objects in images with bounding box coordinates using Interfaze AI.

## When to use this skill

- The user wants to find or locate objects in an image
- The user needs bounding box coordinates for detected objects
- The user wants to count specific items in a photo
- The user says "find the X in this image", "where is the Y", "what objects are here"
- The task involves spatial understanding — positions, regions, or layout of objects
- The user wants both object positions and any visible text with coordinates
- The user wants to locate UI elements in a screenshot for computer-use or UI automation

## When not to use this skill

- The user only wants to read text from an image → use `ocr`
- The user wants to transcribe audio → use `speech-to-text`
- The input is not an image
- The user wants image classification without location data
- The user wants to scrape a URL → use `web-scraping`

## Workflow

1. Confirm the input is an image URL or base64-encoded image.
2. Determine what objects the user wants to detect, or if they want all objects.
3. Build a schema with object names and bounding box fields (`top_left_x`, `top_left_y`, `bottom_right_x`, `bottom_right_y`).
4. Call the chosen SDK with the image and a prompt describing what to detect.
5. Return the detected objects with their positions.

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

Supported sources: public HTTPS URLs and base64 data URLs.

| SDK | Image part |
| --- | --- |
| Interfaze / LangChain (TS & Py) | `{ type: "image_url", image_url: { url: "<url>" } }` |
| Vercel AI SDK | `{ type: "image", mediaType: "image/png", image: "<url-or-base64>" }` |

## Example: detect objects and text in an image

### TypeScript — Interfaze SDK

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

### TypeScript — Vercel AI SDK

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

const { output, response } = await generateText({
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

// @ts-expect-error precontext is not typed
console.log("Object Detection Results:", response.body?.precontext?.[0]?.result);
```

### TypeScript — LangChain SDK

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

### Python — Interfaze SDK

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

### Python — LangChain SDK

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

## Example: detect specific object categories

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

## Tips

- Be specific in the prompt about what to detect. "Find all vehicles" works better than "detect objects."
- Use `.describe()` / `Field(description=...)` on the name field to guide labelling.
- Add a `category` field to group objects by type.
- Coordinates are returned in pixels relative to the original image dimensions.
- Detection runs alongside OCR when the prompt asks for both, producing two `precontext` entries.

## Advanced: raw detection output

Run detection on its own for a fixed output shape without a custom schema. Faster and cheaper than a schema-driven call.

### TypeScript — Interfaze SDK

```ts
const result = await interfaze.tasks.objectDetection("https://example.com/construction.png");

console.log(result);
```

### TypeScript — Vercel AI SDK

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

### TypeScript — LangChain SDK

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

### Python — Interfaze SDK

```python
result = interfaze.tasks.object_detection("https://example.com/construction.png")

print(result)
```

### Python — LangChain SDK

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

## Advanced: GUI detection for computer use

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
