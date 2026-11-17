# Interfaze Skills

Installable [Agent Skills](https://agentskills.io) for [Interfaze AI](https://interfaze.ai). This repo follows the open Agent Skills format and is compatible with the `skills` CLI.

## What is Interfaze AI?

Interfaze AI is a proprietary multimodal AI model accessible through a chat-completions compatible API. It handles vision OCR, speech-to-text, structured output, object detection, web search, web scraping, and document processing through a single unified endpoint — no tool routing or multi-provider setup required.

## Included skills

| Skill                                          | Purpose                                                                                |
| ---------------------------------------------- | -------------------------------------------------------------------------------------- |
| [ocr](skills/ocr/)                             | OCR and visual text extraction from images, scans, PDFs, invoices, receipts, and forms |
| [speech-to-text](skills/speech-to-text/)       | Speech-to-text transcription from audio files, voice notes, and recordings             |
| [structured-output](skills/structured-output/) | Convert model responses into strict JSON or schema-constrained structured objects      |
| [object-detection](skills/object-detection/)   | Detect and locate objects in images with bounding box coordinates                      |
| [web-search](skills/web-search/)               | Search the web for current information, facts, prices, and news                        |
| [web-scraping](skills/web-scraping/)           | Extract structured data from specific web pages and URLs                               |

## Installation

Install all skills:

```bash
npx skills add JigsawStack/interfaze-skills
```

Install a single skill:

```bash
npx skills add JigsawStack/interfaze-skills --skill ocr
```

List available skills without installing:

```bash
npx skills add JigsawStack/interfaze-skills --list
```

## Setup

Set `INTERFAZE_API_KEY` in your environment, then install the SDK for your stack.

```bash
npm install interfaze          # TypeScript
pip install interfaze          # Python
```

```ts
import { Interfaze } from "interfaze";

const interfaze = new Interfaze();
```

```python
from interfaze import Interfaze

interfaze = Interfaze()
```

The base URL and model name are built in, and the client reads `INTERFAZE_API_KEY` automatically.

### Other SDKs

| SDK | Package | Entry point |
| --- | --- | --- |
| LangChain (TypeScript) | `@interfaze-ai/langchain` | `new ChatInterfaze()` |
| LangChain (Python) | `interfaze-langchain` | `ChatInterfaze()` |
| Vercel AI SDK | `ai` + `@interfaze-ai/ai-sdk` | `interfaze.chat("interfaze")` |
| OpenAI SDK | `openai` | `new OpenAI({ baseURL, apiKey })` |

The OpenAI SDK talks to Interfaze over the Chat Completion API standard, so it needs the base URL `https://api.interfaze.ai/v1` and the model name `interfaze`. The Vercel AI SDK uses the native `@interfaze-ai/ai-sdk` provider, which has the base URL built in and exposes `precontext` on `providerMetadata.interfaze`. Each skill's `## Setup` section has the full snippet for every SDK.

## When to use AGENTS.md vs SKILL.md

- **AGENTS.md** contains repo-wide rules and conventions that apply to all skills. Agents load this as passive context.
- **SKILL.md** files contain specialized instructions for a single capability. Agents load these on-demand when a task matches the skill description.

## Format

This repo follows the [Agent Skills open specification](https://agentskills.io/specification). Skills work with 18+ AI agents including Claude Code, GitHub Copilot, Cursor, Cline, and others.

## License

MIT — see [LICENSE](LICENSE).
