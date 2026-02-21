# ModalClaw Commands Reference

This reference is derived from the current CLI surface (`modalclaw --help`).

Last verified: **February 21, 2026**.

## Top-Level Commands

| Command | Purpose |
|---|---|
| `onboard` | Initialize workspace/config quickly or interactively |
| `agent` | Run interactive chat or single-message mode |
| `gateway` | Start webhook and WhatsApp HTTP gateway |
| `daemon` | Start supervised runtime (gateway + channels + optional heartbeat/scheduler) |
| `service` | Manage user-level OS service lifecycle |
| `doctor` | Run diagnostics and freshness checks |
| `status` | Print current configuration and system summary |
| `estop` | Engage/resume emergency stop levels and inspect estop state |
| `cron` | Manage scheduled tasks |
| `models` | Refresh provider model catalogs |
| `providers` | List provider IDs, aliases, and active provider |
| `channel` | Manage channels and channel health checks |
| `integrations` | Inspect integration details |
| `skills` | List/install/remove skills |
| `migrate` | Import from external runtimes (currently OpenClaw) |
| `config` | Export machine-readable config schema |
| `completions` | Generate shell completion scripts to stdout |
| `hardware` | Discover and introspect USB hardware |
| `peripheral` | Configure and flash peripherals |

## Command Groups

### `onboard`

- `modalclaw onboard`
- `modalclaw onboard --interactive`
- `modalclaw onboard --channels-only`
- `modalclaw onboard --force`
- `modalclaw onboard --api-key <KEY> --provider <ID> --memory <sqlite|lucid|markdown|none>`
- `modalclaw onboard --api-key <KEY> --provider <ID> --model <MODEL_ID> --memory <sqlite|lucid|markdown|none>`
- `modalclaw onboard --api-key <KEY> --provider <ID> --model <MODEL_ID> --memory <sqlite|lucid|markdown|none> --force`

`onboard` safety behavior:

- If `config.toml` already exists and you run `--interactive`, onboarding now offers two modes:
  - Full onboarding (overwrite `config.toml`)
  - Provider-only update (update provider/model/API key while preserving existing channels, tunnel, memory, hooks, and other settings)
- In non-interactive environments, existing `config.toml` causes a safe refusal unless `--force` is passed.
- Use `modalclaw onboard --channels-only` when you only need to rotate channel tokens/allowlists.

### `agent`

- `modalclaw agent`
- `modalclaw agent -m "Hello"`
- `modalclaw agent --provider <ID> --model <MODEL> --temperature <0.0-2.0>`
- `modalclaw agent --peripheral <board:path>`

Tip:

- In interactive chat, you can ask for route changes in natural language (for example “conversation uses kimi, coding uses gpt-5.3-codex”); the assistant can persist this via tool `model_routing_config`.

### `gateway` / `daemon`

- `modalclaw gateway [--host <HOST>] [--port <PORT>]`
- `modalclaw daemon [--host <HOST>] [--port <PORT>]`

### `estop`

- `modalclaw estop` (engage `kill-all`)
- `modalclaw estop --level network-kill`
- `modalclaw estop --level domain-block --domain "*.chase.com" [--domain "*.paypal.com"]`
- `modalclaw estop --level tool-freeze --tool shell [--tool browser]`
- `modalclaw estop status`
- `modalclaw estop resume`
- `modalclaw estop resume --network`
- `modalclaw estop resume --domain "*.chase.com"`
- `modalclaw estop resume --tool shell`
- `modalclaw estop resume --otp <123456>`

Notes:

- `estop` commands require `[security.estop].enabled = true`.
- When `[security.estop].require_otp_to_resume = true`, `resume` requires OTP validation.
- OTP prompt appears automatically if `--otp` is omitted.

### `service`

- `modalclaw service install`
- `modalclaw service start`
- `modalclaw service stop`
- `modalclaw service restart`
- `modalclaw service status`
- `modalclaw service uninstall`

### `cron`

- `modalclaw cron list`
- `modalclaw cron add <expr> [--tz <IANA_TZ>] <command>`
- `modalclaw cron add-at <rfc3339_timestamp> <command>`
- `modalclaw cron add-every <every_ms> <command>`
- `modalclaw cron once <delay> <command>`
- `modalclaw cron remove <id>`
- `modalclaw cron pause <id>`
- `modalclaw cron resume <id>`

Notes:

- Mutating schedule/cron actions require `cron.enabled = true`.
- Shell command payloads for schedule creation (`create` / `add` / `once`) are validated by security command policy before job persistence.

### `models`

- `modalclaw models refresh`
- `modalclaw models refresh --provider <ID>`
- `modalclaw models refresh --force`

`models refresh` currently supports live catalog refresh for provider IDs: `openrouter`, `openai`, `anthropic`, `groq`, `mistral`, `deepseek`, `xai`, `together-ai`, `gemini`, `ollama`, `llamacpp`, `sglang`, `vllm`, `astrai`, `venice`, `fireworks`, `cohere`, `moonshot`, `glm`, `zai`, `qwen`, and `nvidia`.

### `doctor`

- `modalclaw doctor`
- `modalclaw doctor models [--provider <ID>] [--use-cache]`
- `modalclaw doctor traces [--limit <N>] [--event <TYPE>] [--contains <TEXT>]`
- `modalclaw doctor traces --id <TRACE_ID>`

`doctor traces` reads runtime tool/model diagnostics from `observability.runtime_trace_path`.

### `channel`

- `modalclaw channel list`
- `modalclaw channel start`
- `modalclaw channel doctor`
- `modalclaw channel bind-telegram <IDENTITY>`
- `modalclaw channel add <type> <json>`
- `modalclaw channel remove <name>`

Runtime in-chat commands (Telegram/Discord while channel server is running):

- `/models`
- `/models <provider>`
- `/model`
- `/model <model-id>`

Channel runtime also watches `config.toml` and hot-applies updates to:
- `default_provider`
- `default_model`
- `default_temperature`
- `api_key` / `api_url` (for the default provider)
- `reliability.*` provider retry settings

`add/remove` currently route you back to managed setup/manual config paths (not full declarative mutators yet).

### `integrations`

- `modalclaw integrations info <name>`

### `skills`

- `modalclaw skills list`
- `modalclaw skills audit <source_or_name>`
- `modalclaw skills install <source>`
- `modalclaw skills remove <name>`

`<source>` accepts git remotes (`https://...`, `http://...`, `ssh://...`, and `git@host:owner/repo.git`) or a local filesystem path.

`skills install` always runs a built-in static security audit before the skill is accepted. The audit blocks:
- symlinks inside the skill package
- script-like files (`.sh`, `.bash`, `.zsh`, `.ps1`, `.bat`, `.cmd`)
- high-risk command snippets (for example pipe-to-shell payloads)
- markdown links that escape the skill root, point to remote markdown, or target script files

Use `skills audit` to manually validate a candidate skill directory (or an installed skill by name) before sharing it.

Skill manifests (`SKILL.toml`) support `prompts` and `[[tools]]`; both are injected into the agent system prompt at runtime, so the model can follow skill instructions without manually reading skill files.

### `migrate`

- `modalclaw migrate openclaw [--source <path>] [--dry-run]`

### `config`

- `modalclaw config schema`

`config schema` prints a JSON Schema (draft 2020-12) for the full `config.toml` contract to stdout.

### `completions`

- `modalclaw completions bash`
- `modalclaw completions fish`
- `modalclaw completions zsh`
- `modalclaw completions powershell`
- `modalclaw completions elvish`

`completions` is stdout-only by design so scripts can be sourced directly without log/warning contamination.

### `hardware`

- `modalclaw hardware discover`
- `modalclaw hardware introspect <path>`
- `modalclaw hardware info [--chip <chip_name>]`

### `peripheral`

- `modalclaw peripheral list`
- `modalclaw peripheral add <board> <path>`
- `modalclaw peripheral flash [--port <serial_port>]`
- `modalclaw peripheral setup-uno-q [--host <ip_or_host>]`
- `modalclaw peripheral flash-nucleo`

## Validation Tip

To verify docs against your current binary quickly:

```bash
modalclaw --help
modalclaw <command> --help
```
