# Second Brain

A personal knowledge system that captures thoughts from Slack, stores them with semantic embeddings in Supabase, and makes them retrievable through any AI tool via MCP.

## Architecture

```
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

## License

MIT
