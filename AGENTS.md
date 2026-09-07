# Interfaze AI — Agent Guidelines

This repo contains installable Agent Skills for Interfaze AI. Follow these rules when working with or contributing to these skills.

## General rules

- Interfaze AI is a proprietary multimodal model. It has first-party SDKs and also supports the Chat Completion API standard, so OpenAI-compatible clients work against its base URL.
- API base URL: `https://api.interfaze.ai/v1`. Model name: `interfaze-beta`. The first-party SDKs have both built in, so neither is configured when using them.
- Interfaze supports the Chat Completion API standard, not the Responses API standard.
- Always require `INTERFAZE_API_KEY` as an environment variable. The first-party SDKs read it automatically.
- Only document how to use the API — inputs, outputs, and patterns.

## Supported SDKs and languages

Show all supported SDK + language combinations for the **primary** example of each skill. For secondary examples, show just the schema (or prompt) and note that the call shape matches the primary example — do not repeat near-identical blocks across every SDK. The combinations the docs cover:

- **TypeScript**: Interfaze SDK (`interfaze`), Vercel AI SDK (`ai` + `@ai-sdk/openai`), LangChain SDK (`@interfaze-ai/langchain`).
- **Python**: Interfaze SDK (`interfaze`), LangChain SDK (`interfaze-langchain`).

Use these section labels consistently inside each example, in this order:

1. `### TypeScript — Interfaze SDK`
2. `### TypeScript — Vercel AI SDK`
3. `### TypeScript — LangChain SDK`
4. `### Python — Interfaze SDK`
5. `### Python — LangChain SDK`

The Interfaze SDK is the default recommendation — lead with it. The OpenAI SDK still works by pointing `baseURL` at Interfaze and passing `model: "interfaze-beta"`; document that once in each skill's `## Setup` section rather than repeating it as a sixth tab on every example.

## Skill authoring

- Keep skill descriptions precise and trigger-oriented. The `description` field is the primary routing surface.
- Prefer narrow, capability-specific skills over broad catch-all skills.
- Each skill is a single self-contained `SKILL.md` — setup, input formats, all examples, and advanced `<task>` modes live there. Do not split content into `references/`; keep one source of truth per skill.
- Do not create empty scaffold directories (e.g. `assets/`, `references/`). Add a directory only when it holds real files.
- Maintain naming consistency: folder names must match the `name` field in frontmatter.

## Writing style

- Use direct, practical language.
- Avoid marketing language, hype, or superlatives.
- Lead with what the user needs to do, not what the product can do.
- Prefer imperative instructions: "Use this when..." not "This skill provides..."

## API conventions

### Interfaze SDK (TypeScript and Python)

- Install `interfaze`. Construct with `new Interfaze()` / `Interfaze()` — no base URL, no model name. `AsyncInterfaze` is the Python async client.
- Use `interfaze.chat.completions.create({ messages: [...] })`.
- For structured output:
  - TypeScript: `response_format: responseFormat(z.toJSONSchema(Schema), "schema_name")`, importing `responseFormat` from `interfaze`. `message.content` is a JSON string, so parse it.
  - Python: `interfaze.chat.completions.parse(response_format=Model)` with a `pydantic.BaseModel`, then read `message.parsed`. `response_format(schema_dict, "schema_name")` with `create()` is the lower-level alternative.
- Pass images as `{ type: "image_url", image_url: { url: "<url>" } }` and audio/PDF/documents as `{ type: "file", file: { filename: "<name>", file_data: "<url-or-base64>" } }`.
- `inputs.image()`, `inputs.file()`, and `inputs.audio()` build those parts. `inputs.fromPath()` / `inputs.from_path()` reads a local file and `inputs.dataUrl()` / `inputs.data_url()` encodes raw bytes.
- Run a single built-in task with a `tasks.*` helper: `tasks.ocr`, `tasks.objectDetection`, `tasks.guiDetection`, `tasks.webSearch`, `tasks.scrape`, `tasks.transcribe`, `tasks.translate`, `tasks.forecast` (snake_case in Python).
- `precontext` is typed on the response as `response.precontext`.

### Vercel AI SDK (TypeScript)

- Requires `ai@^6.0.0` and `@ai-sdk/openai@^3.0.0`. Create the provider with `createOpenAI({ baseURL, apiKey })` and select the model with `interfaze.chat("interfaze-beta")`.
- Use `generateText` with `output: Output.object({ schema })` for structured output, and read the result off `output`. Do not use `generateObject`.
- Use plain `generateText` for text, and `streamText` (with `partialOutputStream` for structured data) for streaming.
- Define schemas with Zod. Use `.describe()` on fields when the field name alone is ambiguous.
- Pass images as `{ type: "image", image: "<url>" }` in message content arrays. Add `mediaType` (e.g. `"image/jpeg"`, `"image/png"`) when the URL extension is ambiguous.
- Pass audio/PDF as `{ type: "file", data: "<url>", mediaType: "<mime>" }`. Use canonical MIME types: `audio/mpeg` for MP3, `audio/wav`, `audio/mp4` for M4A, `application/pdf` for PDF.
- Tools use `tool({ description, inputSchema, execute })` with `stopWhen: stepCountIs(n)`.
- `precontext` is untyped here — read it off `response.body?.precontext`.

### LangChain SDK (TypeScript and Python)

- Install `@interfaze-ai/langchain` (TypeScript) or `interfaze-langchain` (Python) and use the `ChatInterfaze` chat model. Do not use `ChatOpenAI` from `@langchain/openai` / `langchain-openai`.
- `ChatInterfaze` needs no base URL or model name. In TypeScript, `@langchain/core`, `@langchain/openai`, and `interfaze` are peer dependencies.
- Use `interfaze.withStructuredOutput(Schema)` / `interfaze.with_structured_output(Model)` for structured output, and `interfaze.invoke([...])` for plain text.
- Pass `includeRaw: true` / `include_raw=True` when the caller also needs `precontext`; the parsed object is on `.parsed` and the message on `.raw`.
- Pass images and files the same way as the Interfaze SDK (`image_url` and `file`). Video uses an Interfaze-specific `{ type: "video", url }` block.
- `precontext`, `reasoning`, and `vcache` are exposed on `response.response_metadata`.

### Shared request options

- `<task>name</task>` in the system message runs one built-in task without the full model. `response_format` must be an empty/`any` schema, only the first `<task>` tag is parsed, and a non-empty schema alongside it returns a 400.
- `<guard>S1, S2</guard>` in the system message applies safety guardrails. A blocked request returns a normal completion whose content is the string `unsafe S1`.
- Reasoning is off by default; enable it with `reasoning_effort` (`low`, `medium`, `high`).
- Client options `showAdditionalInfo`, `bypassMoA`, and `bypassCache` map to the `x-show-additional-info`, `x-interfaze-bypass-moa`, and `x-interfaze-bypass-cache` headers.

### Limits

- Max file size is 20 MB as a base64 or binary file part, and 80 MB when the URL is passed in the prompt text. Prefer URLs in the prompt for large files.
- Context window 1M tokens, max output 32k tokens, 5 minute max request duration, 50 requests/sec.

## Source of truth

- The Interfaze API behavior is the source of truth. If a skill contradicts observed API behavior, update the skill.
- The docs in the `interfaze-landing` repo (`pages/docs/**`) are the reference for SDK surface and call shapes. The skills published under its `public/.well-known/agent-skills/` are generated from this repo, so never treat them as an upstream source.
- Do not invent capabilities that have not been tested against the live API.
