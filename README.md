# Omni Adapter — AI setup guide

Connect your own program to a ProtoPie **Connect Omni** stage with help from your AI assistant.
You don't need to pick a language first: give your assistant this guide, and it chooses the right
Omni Adapter package for your project, installs it, and writes the connection code.

## Any AI assistant

Give your assistant this link:

```text
https://raw.githubusercontent.com/ProtoPie/omni-adapter-ai/main/llms.txt
```

For example:

> Read https://raw.githubusercontent.com/ProtoPie/omni-adapter-ai/main/llms.txt, then connect this
> project to my Connect Omni stage with the Omni Adapter.

If your assistant can't open links, download [`llms.txt`](llms.txt) and attach it to the chat.

## Claude Code

Install the skill as a plugin:

```text
/plugin marketplace add ProtoPie/omni-adapter-ai
/plugin install omni-adapter@studioxid
```

- **Updates:** automatic updates are off by default for this marketplace. Turn them on in
  `/plugin` → **Marketplaces** → `studioxid` → **Enable auto-update**, or update by hand with
  `/plugin marketplace update studioxid`.
- **No GitHub SSH key?** Set `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1` in your environment before
  starting Claude Code, so it clones over HTTPS without trying SSH first.
- **Without plugins:** copy [`skills/using-omni-adapter/`](skills/using-omni-adapter/) into your
  project's `.claude/skills/`.

## What you need

- A Connect Omni stage.
- An **adapter token** (`cst_…`). An Editor or Owner of the stage creates it: open the stage →
  **Share → Adapter keys → Create key**, then **Copy**. The token is shown only once. Keep it in an
  environment variable, never in source code.

## Packages

Your assistant picks one of these for you:

| Language | Package |
| --- | --- |
| TypeScript / JavaScript / React | npm `@studioxid/omni-adapter` |
| Python 3.10+ | PyPI `omni-adapter` |
| C# / .NET / Unity | NuGet `StudioXid.OmniAdapter` |
| Kotlin / JVM / Android | Maven `com.studioxid:omni-adapter` |
| Swift | SwiftPM `https://github.com/ProtoPie/omni-adapter-swift` |
| Dart / Flutter | pub.dev `omni_adapter` |

## What's in this repository

- [`llms.txt`](llms.txt) — the full guide for AI assistants, all six languages
- [`skills/using-omni-adapter/`](skills/using-omni-adapter/) — the Claude Code skill
- [`docs/getting-started.md`](docs/getting-started.md) — the first connection, step by step, for people
- [`docs/COMPATIBILITY.md`](docs/COMPATIBILITY.md) — which features each language and server version supports

These files are copied here from the Omni Adapter source on each release. Changes made directly in
this repository are overwritten.

## License

Apache License 2.0 — see [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).
