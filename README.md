# Second Brain

A personal knowledge system that captures thoughts from Slack, stores them with semantic embeddings in Supabase, and makes them retrievable through any AI tool via MCP.

## Architecture

```text
Slack message
    |
    v
Edge Function (ingest-thought)
    |
    +--> OpenRouter (embedding + metadata extraction)
    |
    v
Supabase (pgvector)
    |
    v
Edge Function (second-brain-mcp)
    |
    v
AI Clients (Claude, ChatGPT, Cursor, etc.)
```

## Documentation

- [Setup Guide](docs/second-brain-guide.md) -- step-by-step instructions for building the capture and retrieval systems
- [Companion Prompts](docs/second-brain-prompts.md) -- prompts for migrating existing knowledge, discovering capture opportunities, and running weekly reviews

## Privacy

All captured thoughts are sent to external APIs for processing:

- **Embeddings** are generated via OpenRouter using OpenAI's `text-embedding-3-small` model.
- **Metadata extraction** uses OpenAI's `gpt-4o-mini` via OpenRouter.

Your thought content transits OpenRouter and OpenAI servers. Do not capture information you would not send to a third-party API.
Review [OpenRouter's privacy policy](https://openrouter.ai/privacy) and
[OpenAI's API data usage policy](https://openai.com/enterprise-privacy/) before use.

## Prerequisites

- Supabase account (free tier works)
- Slack workspace where you can install apps
- OpenRouter API key
- Supabase CLI installed locally
- Terminal access

## Cost

All services run on free tiers. The only variable cost is OpenRouter API usage.

| Service | Cost |
|---|---|
| Slack | Free |
| Supabase (free tier) | $0 |
| Embeddings (text-embedding-3-small) | ~$0.02 / million tokens |
| Metadata extraction (gpt-4o-mini) | ~$0.15 / million input tokens |
| **20 thoughts/day** | **~$0.10--0.30/month** |

## About This Repo

This is a **documentation-only** repository containing the setup guide and companion prompts for building a Second Brain system.
The actual edge function code is provided inline in the guide -- you create the functions in your own Supabase project
by following the steps.

## FAQ

**Why not Notion, Obsidian, or another note-taking app?**
Second Brain is not a note-taking app. It is a context bus for AI tools. You capture in Slack (zero friction),
and the context surfaces automatically in any MCP-connected AI client. The value is in cross-tool retrieval, not capture.

**Why Supabase + edge functions + Slack + MCP?**
Each component exists for a reason. Supabase provides free hosted Postgres with pgvector. Edge functions are serverless
with no infrastructure to manage. Slack is zero-friction capture in a tool you already use. MCP is the universal AI client
integration protocol. The complexity is in the one-time setup, not the runtime.

**Why OpenRouter instead of calling OpenAI directly?**
OpenRouter provides a single API key for multiple model providers. If OpenAI changes pricing or deprecates a model, you can switch to an alternative without changing your code structure.

**Is my data safe on the free tier?**
Supabase free tier does not include automatic backups. See [Appendix C](docs/second-brain-guide.md#appendix-c-backup-and-export) in the setup guide for backup options.

## License

MIT
