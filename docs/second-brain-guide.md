# Second Brain Setup Guide

A step-by-step guide to building a personal knowledge capture and retrieval system using Slack, Supabase, OpenRouter, and MCP.

**Capture** turns Slack messages into semantically searchable thoughts. **Retrieval** exposes those thoughts to any AI tool through an MCP server.

---

## Before You Start

Print or copy this credential tracker. Fill in each value as you complete the corresponding step.

```text
SECOND BRAIN -- CREDENTIAL TRACKER
Keep this file. Fill in as you go.
--------------------------------------

SUPABASE
  Account email:        ________________
  Account password:     ________________
  Database password:    ________________  <- Step 1
  Project name:         ________________
  Project ref:          ________________  <- Step 1
  Project URL:          ________________  <- Step 3
  Service role key:     ________________  <- Step 3

OPENROUTER
  Account email:        ________________
  Account password:     ________________
  API key:              ________________  <- Step 4

SLACK
  Workspace name:       ________________
  Workspace URL:        ________________
  Channel name:         ________________
  Channel ID:           ________________  <- Step 5
  Bot OAuth Token:      ________________  <- Step 6

GENERATED DURING SETUP
  Edge Function URL:    ________________  <- Step 7
  MCP Access Key:       ________________  <- Step 10
  MCP Server URL:       ________________  <- Step 11
  MCP Connection URL:   ________________  <- Step 11

--------------------------------------
```

---

## Part 1: Capture System

### Step 1: Create a Supabase Project

1. Go to [supabase.com/dashboard](https://supabase.com/dashboard) and sign in (or create an account).
2. Click **New Project**.
3. Name: `second-brain` (or whatever you prefer).
4. Set a strong database password. **Save it in your credential tracker.**
5. Select your preferred region.
6. Click **Create new project**.
7. Wait for provisioning to complete (takes about a minute).

Copy the **Project ref** from the URL bar (the string after `https://supabase.com/dashboard/project/`) and save it.

---

### Step 2: Set Up the Database

Open the **SQL Editor** in the Supabase dashboard (left sidebar). Run each block below in order.

#### Enable pgvector

```sql
create extension if not exists vector with schema extensions;
```

#### Create the thoughts table

```sql
create table thoughts (
  id uuid default gen_random_uuid() primary key,
  content text not null,
  embedding vector(1536),
  metadata jsonb default '{}'::jsonb,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create index on thoughts
  using hnsw (embedding vector_cosine_ops);

create index on thoughts using gin (metadata);

create index on thoughts (created_at desc);

create or replace function update_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger thoughts_updated_at
  before update on thoughts
  for each row
  execute function update_updated_at();
```

#### Create the search function

```sql
create or replace function match_thoughts(
  query_embedding vector(1536),
  match_threshold float default 0.7,
  match_count int default 10,
  filter jsonb default '{}'::jsonb
)
returns table (
  id uuid,
  content text,
  metadata jsonb,
  similarity float,
  created_at timestamptz
)
language plpgsql
as $$
begin
  return query
  select
    t.id,
    t.content,
    t.metadata,
    1 - (t.embedding <=> query_embedding) as similarity,
    t.created_at
  from thoughts t
  where 1 - (t.embedding <=> query_embedding) > match_threshold
  and (filter = '{}'::jsonb or t.metadata @> filter)
  order by t.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

#### Enable row-level security

```sql
alter table thoughts enable row level security;

create policy "Service role full access"
  on thoughts
  for all
  using (auth.role() = 'service_role');
```

---

### Step 3: Save Your Connection Credentials

1. In the Supabase dashboard, go to **Settings > API**.
2. Copy:
   - **Project URL** (looks like `https://xxxxx.supabase.co`)
   - **Service role key** (under `service_role` -- keep this secret)
3. Save both in your credential tracker.

> **Note:** `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are automatically available inside Supabase Edge Functions. You do not need to set them as secrets.

---

### Step 4: Get an OpenRouter API Key

1. Go to [openrouter.ai](https://openrouter.ai) and create an account.
2. Add a few dollars in credits (pay-as-you-go). The models used here cost fractions of a cent per call -- see Appendix A for a full breakdown.
3. Go to **Keys > Create Key**.
4. Name: `second-brain`.
5. Copy the key and save it in your credential tracker.

---

### Step 5: Create a Slack Capture Channel

1. Open Slack.
2. Create a new **private** channel. Name it `capture` (or `brain`, `inbox` -- whatever you prefer).
3. Click the channel name at the top to open channel details.
4. Copy the **Channel ID** (starts with `C`). On desktop, this is at the bottom of the details panel.
5. Save it in your credential tracker.

---

### Step 6: Create a Slack App

#### Create the app

1. Go to [api.slack.com/apps](https://api.slack.com/apps).
2. Click **Create New App > From scratch**.
3. App Name: `Second Brain`.
4. Select your workspace.
5. Click **Create App**.

#### Add bot token scopes

1. In the left sidebar, go to **OAuth & Permissions**.
2. Scroll to **Bot Token Scopes** and add these three scopes:
   - `channels:history`
   - `groups:history`
   - `chat:write`
3. Scroll back up and click **Install to Workspace**.
4. Authorize the app.
5. Copy the **Bot User OAuth Token** (starts with `xoxb-`).
6. Save it in your credential tracker.

#### Invite the bot to your channel

In Slack, go to your capture channel and type:

```text
/invite @Second Brain
```

---

### Step 7: Deploy the Ingest Edge Function

#### Initialize Supabase locally

If you have not already, install the [Supabase CLI](https://supabase.com/docs/guides/cli/getting-started) and link your project:

```bash
supabase init
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

#### Create the function

```bash
supabase functions new ingest-thought
```

#### Replace the function code

Replace the contents of `supabase/functions/ingest-thought/index.ts` with:

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;
const SLACK_BOT_TOKEN = Deno.env.get("SLACK_BOT_TOKEN")!;
const SLACK_CAPTURE_CHANNEL = Deno.env.get("SLACK_CAPTURE_CHANNEL")!;

const OPENROUTER_BASE = "https://openrouter.ai/api/v1";
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: text,
    }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

async function extractMetadata(
  text: string,
): Promise<Record<string, unknown>> {
  const r = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata from the user's captured thought. Return JSON with:
- "people": array of people mentioned (empty if none)
- "action_items": array of implied to-dos (empty if none)
- "dates_mentioned": array of dates YYYY-MM-DD (empty if none)
- "topics": array of 1-3 short topic tags (always at least one)
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what's explicitly there.`,
        },
        { role: "user", content: text },
      ],
    }),
  });
  const d = await r.json();
  try {
    return JSON.parse(d.choices[0].message.content);
  } catch {
    return { topics: ["uncategorized"], type: "observation" };
  }
}

async function replyInSlack(
  channel: string,
  threadTs: string,
  text: string,
): Promise<void> {
  await fetch("https://slack.com/api/chat.postMessage", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${SLACK_BOT_TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ channel, thread_ts: threadTs, text }),
  });
}

Deno.serve(async (req: Request): Promise<Response> => {
  try {
    const body = await req.json();

    // Handle Slack URL verification challenge
    if (body.type === "url_verification") {
      return new Response(JSON.stringify({ challenge: body.challenge }), {
        headers: { "Content-Type": "application/json" },
      });
    }

    const event = body.event;

    // Ignore non-message events, bot messages, and messages outside the capture channel
    if (
      !event ||
      event.type !== "message" ||
      event.subtype ||
      event.bot_id ||
      event.channel !== SLACK_CAPTURE_CHANNEL
    ) {
      return new Response("ok", { status: 200 });
    }

    const messageText: string = event.text;
    const channel: string = event.channel;
    const messageTs: string = event.ts;

    if (!messageText || messageText.trim() === "") {
      return new Response("ok", { status: 200 });
    }

    // Generate embedding and extract metadata in parallel
    const [embedding, metadata] = await Promise.all([
      getEmbedding(messageText),
      extractMetadata(messageText),
    ]);

    // Store the thought
    const { error } = await supabase.from("thoughts").insert({
      content: messageText,
      embedding,
      metadata: { ...metadata, source: "slack", slack_ts: messageTs },
    });

    if (error) {
      console.error("Supabase insert error:", error);
      await replyInSlack(channel, messageTs, `Failed to capture: ${error.message}`);
      return new Response("error", { status: 500 });
    }

    // Build confirmation message
    const meta = metadata as Record<string, unknown>;
    let confirmation = `Captured as *${meta.type || "thought"}*`;
    if (Array.isArray(meta.topics) && meta.topics.length > 0) {
      confirmation += ` - ${meta.topics.join(", ")}`;
    }
    if (Array.isArray(meta.people) && meta.people.length > 0) {
      confirmation += `\nPeople: ${meta.people.join(", ")}`;
    }
    if (Array.isArray(meta.action_items) && meta.action_items.length > 0) {
      confirmation += `\nAction items: ${meta.action_items.join("; ")}`;
    }

    await replyInSlack(channel, messageTs, confirmation);
    return new Response("ok", { status: 200 });
  } catch (err) {
    console.error("Function error:", err);
    return new Response("error", { status: 500 });
  }
});
```

#### Set secrets

```bash
supabase secrets set OPENROUTER_API_KEY=your-openrouter-key-here
supabase secrets set SLACK_BOT_TOKEN=xoxb-your-slack-bot-token-here
supabase secrets set SLACK_CAPTURE_CHANNEL=C0your-channel-id-here
```

#### Deploy

```bash
supabase functions deploy ingest-thought --no-verify-jwt
```

Copy the **Edge Function URL** from the output and save it in your credential tracker.

---

### Step 8: Connect Slack to the Edge Function

1. Go back to your Slack app settings at [api.slack.com/apps](https://api.slack.com/apps).
2. In the left sidebar, click **Event Subscriptions**.
3. Toggle **Enable Events** to on.
4. In the **Request URL** field, paste your Edge Function URL from Step 7.
5. Wait for Slack to verify the URL (the function handles the `url_verification` challenge automatically).
6. Under **Subscribe to bot events**, add both:
   - `message.channels`
   - `message.groups`
7. Click **Save Changes**.

---

### Step 9: Test the Capture System

1. Go to your Slack capture channel.
2. Type a message:
   ```text
   Testing my second brain. This is my first captured thought.
   ```
3. Wait a few seconds.
4. The bot should reply in a thread with a confirmation showing the thought type and topics.
5. Open the Supabase dashboard and check the **Table Editor > thoughts** table to see the stored record.

**If something goes wrong:**

- Check Edge Function logs: `supabase functions logs ingest-thought`
- Check Slack event delivery: go to your Slack app > **Event Subscriptions** and look at the request log
- Verify secrets are set: `supabase secrets list`

---

## Part 2: Retrieval System

### Step 10: Generate an MCP Access Key

Generate a random key for authenticating MCP requests:

```bash
openssl rand -hex 32
```

Save this as your **MCP Access Key** in the credential tracker.

Set it as a Supabase secret:

```bash
supabase secrets set MCP_ACCESS_KEY=your-generated-key-here
```

---

### Step 11: Deploy the MCP Server

#### Create the function

```bash
supabase functions new second-brain-mcp
```

#### Add dependencies

Create `supabase/functions/second-brain-mcp/deno.json`:

```json
{
  "imports": {
    "@hono/mcp": "npm:@hono/mcp@0.1.1",
    "@modelcontextprotocol/sdk": "npm:@modelcontextprotocol/sdk@1.24.3",
    "hono": "npm:hono@4.9.2",
    "zod": "npm:zod@4.1.13",
    "@supabase/supabase-js": "npm:@supabase/supabase-js@2.47.10"
  }
}
```

#### Replace the function code

Replace `supabase/functions/second-brain-mcp/index.ts` with the following. This implements four MCP tools: `search_thoughts`, `list_thoughts`, `thought_stats`, and `capture_thought`.

```typescript
import "jsr:@supabase/functions-js/edge-runtime.d.ts";

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPTransport } from "@hono/mcp";
import { Hono } from "hono";
import { z } from "zod";
import { createClient } from "@supabase/supabase-js";

const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;
const MCP_ACCESS_KEY = Deno.env.get("MCP_ACCESS_KEY")!;

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch("https://openrouter.ai/api/v1/embeddings", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: text,
    }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

// --- MCP Server Setup ---

const server = new McpServer({
  name: "second-brain",
  version: "1.0.0",
});

// Tool 1: Semantic Search
server.registerTool(
  "search_thoughts",
  {
    title: "Search Thoughts",
    description:
      "Search captured thoughts by meaning. Use this when the user asks about a topic, person, or idea they've previously captured.",
    inputSchema: {
      query: z.string().describe("What to search for"),
      limit: z.number().optional().default(10),
      threshold: z.number().optional().default(0.5),
    },
  },
  async ({ query, limit, threshold }) => {
    try {
      const qEmb = await getEmbedding(query);
      const { data, error } = await supabase.rpc("match_thoughts", {
        query_embedding: qEmb,
        match_threshold: threshold,
        match_count: limit,
        filter: {},
      });

      if (error) {
        return {
          content: [{ type: "text" as const, text: `Search error: ${error.message}` }],
          isError: true,
        };
      }

      if (!data || data.length === 0) {
        return {
          content: [{ type: "text" as const, text: `No thoughts found matching "${query}".` }],
        };
      }

      const results = data.map(
        (
          t: {
            content: string;
            metadata: Record<string, unknown>;
            similarity: number;
            created_at: string;
          },
          i: number
        ) => {
          const m = t.metadata || {};
          const parts = [
            `--- Result ${i + 1} (${(t.similarity * 100).toFixed(1)}% match) ---`,
            `Captured: ${new Date(t.created_at).toLocaleDateString()}`,
            `Type: ${m.type || "unknown"}`,
          ];
          if (Array.isArray(m.topics) && m.topics.length)
            parts.push(`Topics: ${(m.topics as string[]).join(", ")}`);
          if (Array.isArray(m.people) && m.people.length)
            parts.push(`People: ${(m.people as string[]).join(", ")}`);
          if (Array.isArray(m.action_items) && m.action_items.length)
            parts.push(`Actions: ${(m.action_items as string[]).join("; ")}`);
          parts.push(`\n${t.content}`);
          return parts.join("\n");
        }
      );

      return {
        content: [
          {
            type: "text" as const,
            text: `Found ${data.length} thought(s):\n\n${results.join("\n\n")}`,
          },
        ],
      };
    } catch (err: unknown) {
      return {
        content: [{ type: "text" as const, text: `Error: ${(err as Error).message}` }],
        isError: true,
      };
    }
  }
);

// Tool 2: List Recent
server.registerTool(
  "list_thoughts",
  {
    title: "List Recent Thoughts",
    description:
      "List recently captured thoughts with optional filters by type, topic, person, or time range.",
    inputSchema: {
      limit: z.number().optional().default(10),
      type: z.string().optional().describe("Filter by type: observation, task, idea, reference, person_note"),
      topic: z.string().optional().describe("Filter by topic tag"),
      person: z.string().optional().describe("Filter by person mentioned"),
      days: z.number().optional().describe("Only thoughts from the last N days"),
    },
  },
  async ({ limit, type, topic, person, days }) => {
    try {
      let q = supabase
        .from("thoughts")
        .select("content, metadata, created_at")
        .order("created_at", { ascending: false })
        .limit(limit);

      if (type) q = q.contains("metadata", { type });
      if (topic) q = q.contains("metadata", { topics: [topic] });
      if (person) q = q.contains("metadata", { people: [person] });
      if (days) {
        const since = new Date();
        since.setDate(since.getDate() - days);
        q = q.gte("created_at", since.toISOString());
      }

      const { data, error } = await q;

      if (error) {
        return {
          content: [{ type: "text" as const, text: `Error: ${error.message}` }],
          isError: true,
        };
      }

      if (!data || !data.length) {
        return { content: [{ type: "text" as const, text: "No thoughts found." }] };
      }

      const results = data.map(
        (
          t: { content: string; metadata: Record<string, unknown>; created_at: string },
          i: number
        ) => {
          const m = t.metadata || {};
          const tags = Array.isArray(m.topics) ? (m.topics as string[]).join(", ") : "";
          return `${i + 1}. [${new Date(t.created_at).toLocaleDateString()}] (${m.type || "??"}${tags ? " - " + tags : ""})\n   ${t.content}`;
        }
      );

      return {
        content: [
          {
            type: "text" as const,
            text: `${data.length} recent thought(s):\n\n${results.join("\n\n")}`,
          },
        ],
      };
    } catch (err: unknown) {
      return {
        content: [{ type: "text" as const, text: `Error: ${(err as Error).message}` }],
        isError: true,
      };
    }
  }
);

// Tool 3: Stats
server.registerTool(
  "thought_stats",
  {
    title: "Thought Statistics",
    description: "Get a summary of all captured thoughts: totals, types, top topics, and people.",
    inputSchema: {},
  },
  async () => {
    try {
      const { count } = await supabase
        .from("thoughts")
        .select("*", { count: "exact", head: true });

      const { data } = await supabase
        .from("thoughts")
        .select("metadata, created_at")
        .order("created_at", { ascending: false });

      const types: Record<string, number> = {};
      const topics: Record<string, number> = {};
      const people: Record<string, number> = {};

      for (const r of data || []) {
        const m = (r.metadata || {}) as Record<string, unknown>;
        if (m.type) types[m.type as string] = (types[m.type as string] || 0) + 1;
        if (Array.isArray(m.topics))
          for (const t of m.topics) topics[t as string] = (topics[t as string] || 0) + 1;
        if (Array.isArray(m.people))
          for (const p of m.people) people[p as string] = (people[p as string] || 0) + 1;
      }

      const sort = (o: Record<string, number>): [string, number][] =>
        Object.entries(o)
          .sort((a, b) => b[1] - a[1])
          .slice(0, 10);

      const lines: string[] = [
        `Total thoughts: ${count}`,
        `Date range: ${
          data?.length
            ? new Date(data[data.length - 1].created_at).toLocaleDateString() +
              " → " +
              new Date(data[0].created_at).toLocaleDateString()
            : "N/A"
        }`,
        "",
        "Types:",
        ...sort(types).map(([k, v]) => `  ${k}: ${v}`),
      ];

      if (Object.keys(topics).length) {
        lines.push("", "Top topics:");
        for (const [k, v] of sort(topics)) lines.push(`  ${k}: ${v}`);
      }

      if (Object.keys(people).length) {
        lines.push("", "People mentioned:");
        for (const [k, v] of sort(people)) lines.push(`  ${k}: ${v}`);
      }

      return { content: [{ type: "text" as const, text: lines.join("\n") }] };
    } catch (err: unknown) {
      return {
        content: [{ type: "text" as const, text: `Error: ${(err as Error).message}` }],
        isError: true,
      };
    }
  }
);

// Tool 4: Capture Thought
server.registerTool(
  "capture_thought",
  {
    title: "Capture Thought",
    description:
      "Save a new thought to the second brain. Use this to capture observations, tasks, ideas, references, or notes about people.",
    inputSchema: {
      content: z.string().describe("The thought content to save"),
      type: z
        .enum(["observation", "task", "idea", "reference", "person_note"])
        .describe("Type of thought"),
      topics: z
        .array(z.string())
        .optional()
        .default([])
        .describe("Topic tags for categorization"),
      people: z
        .array(z.string())
        .optional()
        .default([])
        .describe("People mentioned in this thought"),
      action_items: z
        .array(z.string())
        .optional()
        .default([])
        .describe("Action items extracted from this thought"),
    },
  },
  async ({ content, type, topics, people, action_items }) => {
    try {
      const embedding = await getEmbedding(content);

      const metadata = {
        type,
        topics,
        people,
        action_items,
        source: "mcp",
      };

      const { error } = await supabase.from("thoughts").insert({
        content,
        embedding,
        metadata,
      });

      if (error) {
        return {
          content: [
            { type: "text" as const, text: `Failed to save thought: ${error.message}` },
          ],
          isError: true,
        };
      }

      const preview = content.length > 100 ? content.slice(0, 100) + "..." : content;
      return {
        content: [
          {
            type: "text" as const,
            text: `Thought saved successfully.\nType: ${type}\nTopics: ${topics.length ? topics.join(", ") : "none"}\nContent: ${preview}`,
          },
        ],
      };
    } catch (err: unknown) {
      return {
        content: [{ type: "text" as const, text: `Error: ${(err as Error).message}` }],
        isError: true,
      };
    }
  }
);

// --- Hono App with Auth Check ---

const app = new Hono();

app.all("*", async (c) => {
  // MCP uses POST exclusively
  if (c.req.method !== "POST") {
    return c.json({ error: "Method not allowed" }, 405);
  }

  // Check access key (header preferred, query param as fallback for clients that embed it in the URL)
  const provided = c.req.header("x-brain-key") || new URL(c.req.url).searchParams.get("key");
  if (!provided || provided !== MCP_ACCESS_KEY) {
    return c.json({ error: "Invalid or missing access key" }, 401);
  }

  const transport = new StreamableHTTPTransport();
  await server.connect(transport);
  return transport.handleRequest(c);
});

Deno.serve(app.fetch);
```

#### Deploy

```bash
supabase functions deploy second-brain-mcp --no-verify-jwt
```

Copy the **MCP Server URL** from the output and save it.

Your **MCP Connection URL** is the server URL with your access key appended:

```text
https://YOUR_PROJECT_REF.supabase.co/functions/v1/second-brain-mcp?key=YOUR_MCP_ACCESS_KEY
```

Save this in your credential tracker.

---

### Step 12: Connect AI Clients

Use the MCP Connection URL or the server URL with the `x-brain-key` header, depending on the client.

#### Claude Desktop

1. Open **Settings > Connectors**.
2. Click **Add custom connector**.
3. Name: `Second Brain`.
4. URL: paste your MCP Connection URL (with `?key=`).
5. Save.

#### Claude Code

```bash
claude mcp add second-brain \
  --transport http \
  --url https://YOUR_PROJECT_REF.supabase.co/functions/v1/second-brain-mcp \
  --header "x-brain-key: YOUR_MCP_ACCESS_KEY"
```

#### ChatGPT

1. Go to **Settings > Connected Apps** (or **Settings > Tools & Integrations**).
2. Click **Add > Custom MCP Server**.
3. Name: `Second Brain`.
4. URL: paste your MCP Connection URL (with `?key=`).
5. Save.

#### Other MCP clients (Cursor, VS Code Copilot, Windsurf)

These clients do not natively support remote MCP servers. Use the `mcp-remote` bridge.

Add this to your MCP client configuration (typically `~/.cursor/mcp.json`, `settings.json`, or equivalent):

```json
{
  "mcpServers": {
    "second-brain": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://YOUR_PROJECT_REF.supabase.co/functions/v1/second-brain-mcp",
        "--header",
        "x-brain-key:${BRAIN_KEY}"
      ],
      "env": {
        "BRAIN_KEY": "your-access-key"
      }
    }
  }
}
```

#### Securing your access key

The examples above embed the key directly in URLs and config files for simplicity. For ongoing use, store it in an environment variable instead:

- **direnv:** Add `export BRAIN_KEY=your-access-key` to `~/.envrc` and reference `${BRAIN_KEY}` in your configs.
- **Shell profile:** Add the export to `~/.zshrc` or `~/.bashrc`.
- **macOS Keychain / 1Password CLI:** Retrieve the key at runtime rather than storing it in plaintext files.

---

### Step 13: Usage Examples

Once connected, you can use natural language with any AI client. The client will call the appropriate MCP tool automatically.

| What you want to do | What to say | Tool used |
|---|---|---|
| Find a past thought | "Search my second brain for what I said about project timelines" | `search_thoughts` |
| Browse recent captures | "What have I captured this week?" | `list_thoughts` |
| Filter by person | "What do I know about Sarah?" | `list_thoughts` |
| Filter by topic | "Show my thoughts tagged with hiring" | `list_thoughts` |
| Check your stats | "How many thoughts are in my second brain?" | `thought_stats` |
| Save from a conversation | "Save this to my second brain: we decided to use Postgres for the new service" | `capture_thought` |

---

## Appendix A: Cost Breakdown

All pricing is based on OpenRouter's pass-through rates for the models used.

| Component | Model | Rate | Typical usage (20 thoughts/day) |
|---|---|---|---|
| Embeddings | text-embedding-3-small | $0.02 / 1M tokens | ~600 tokens/day = ~$0.01/month |
| Metadata extraction | gpt-4o-mini | $0.15 / 1M input tokens | ~2,000 tokens/day = ~$0.01/month |
| Search queries | text-embedding-3-small | $0.02 / 1M tokens | ~200 tokens/query |
| MCP capture | Both models | Combined | Same as Slack capture |
| **Total** | | | **~$0.10--0.30/month** |

Supabase free tier includes 500 MB database, 500K Edge Function invocations, and 2 GB bandwidth per month. This is more than sufficient for personal use.

---

## Appendix B: Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Slack message not captured | Bot not in channel, or channel ID mismatch | Run `/invite @Second Brain` in the channel. Verify `SLACK_CAPTURE_CHANNEL` matches the Channel ID. |
| Bot does not reply in thread | Missing `chat:write` scope, or bot token wrong | Check OAuth & Permissions for the `chat:write` scope. Verify `SLACK_BOT_TOKEN` starts with `xoxb-`. |
| "url_verification" fails in Slack | Edge Function not deployed, or URL wrong | Run `supabase functions deploy ingest-thought --no-verify-jwt` again. Check the URL in Slack matches exactly. |
| "Unauthorized" from MCP server | Wrong access key | Verify `MCP_ACCESS_KEY` matches between `supabase secrets` and your client config. |
| Search returns no results | Threshold too high, or no matching thoughts | Lower the similarity threshold (default is 0.5). Check that thoughts exist in the table. |
| Edge Function timeout | OpenRouter slow or unreachable | Check OpenRouter status. Edge Functions have a 60-second timeout by default. |
| "Invalid API key" from OpenRouter | Key expired or wrong | Generate a new key at openrouter.ai and update with `supabase secrets set`. |
| MCP tools not showing in AI client | Client not connected, or server URL wrong | Reconnect the MCP server. For Claude Code, run `claude mcp list` to verify. |
