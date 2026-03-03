# Changelog

## v0.2.0 (2026-03-03)

### Added
- **Multi-provider support** — Claude (Anthropic SDK), GPT (OpenAI SDK), Gemini (OpenAI-compatible)
  - 6 active models: opus, sonnet, haiku, codex, gpt-5.2, gemini-3-pro
  - Auto-routing by model ID prefix to correct SDK and API endpoint
  - Internal message format unified (anthropic canonical), auto-converted for OpenAI
- **Model aliases** — `/model opus`, `/model codex`, `/model gemini` quick switching
- **`/sync` command** — Fetch latest model list from aicodewith plugin GitHub repo
  - Parses `registry.ts`, filters deprecated models, updates `covenant.json`
  - Shows diff: new models, removed models
- **Multi-line input** — `prompt_toolkit` integration
  - Enter for newline, Alt+Enter (or Esc then Enter) to send
- **Model catalog in `/model`** — Lists all available models with provider, context window, max tokens

### Changed
- Default model: `claude-opus-4-6-20260205`
- Tool definitions: dual format (Anthropic + OpenAI function calling)
- Banner shows provider type

## v0.1.0 (2026-03-03)

### Initial release
- Direct Anthropic API streaming via `anthropic` SDK
- 4 core tools: `bash`, `read_file`, `write_file`, `edit_file`
- Tool call loop with auto-continuation
- Rich Markdown streaming output
- Soul memory system (`~/.soulcovenant/soul/*.md` → system prompt)
- Genesis custom system prompt (`~/.soulcovenant/genesis.md`)
- JSONL session persistence (`~/.soulcovenant/scrolls/`)
- Session resume (`--awaken` / `--scroll`)
- Interactive commands: `/rebirth`, `/ascend`, `/model`, `/scrolls`, `/soul`, `/help`
- Config priority: env vars > `covenant.json` > defaults
- Ctrl+C graceful interruption
