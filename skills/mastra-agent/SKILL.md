---
name: Mastra Agent
description: Scaffold a complete TypeScript agent using the Mastra framework. Covers tools, multi-turn conversation, and optional Large Language Platform observability.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.3.0
---

# Mastra Agent

Scaffold a complete, runnable TypeScript agent using [Mastra](https://mastra.ai).

## When to use

- When building a TypeScript agent with the Mastra framework.
- When you want tool calling and optional multi-turn conversation with minimal boilerplate.

---

## Instructions

### Step 1 — Gather requirements

Ask the user for the following. Do not proceed until all required items are answered.

**Required:**
1. **Agent name** — kebab-case (e.g. `loan-advisor`, `code-reviewer`)
2. **Domain** — one sentence describing what the agent specialises in
3. **LLM** — model to use via Mastra's model router. The user may give a vague answer like "openai" or "claude" instead of an exact model string. Map their answer to a concrete `provider/model` value using this table:

| User says | Use |
|-----------|-----|
| ollama, gpt-oss *(default)* | `ollama-cloud/gpt-oss:120b` |
| openai, gpt | `openai/gpt-4o` |
| claude, anthropic | `anthropic/claude-sonnet-4-5-20241022` |
| google, gemini | `google/gemini-2.0-flash` |
| groq | `groq/llama-3.3-70b-versatile` |

If the user provides an exact `provider/model` string, use it as-is. If they name a provider without a specific model, use the default from this table. Confirm the resolved model string with the user before proceeding.
4. **Tools** — list the tools this agent needs. For each: name, description, inputs, return value. If the agent needs no tools, confirm this explicitly.

**Optional:**
5. **Multi-turn** — maintain conversation history within a session? Default: no.
6. **Structured output** — return typed structured data instead of plain text? Default: no (use `result.text`).

**Derived from LLM choice:**

The provider prefix in the model string determines the required API key env var. Mastra's model router looks up `{PROVIDER}_API_KEY` automatically:

| Provider prefix | Env var |
|----------------|---------|
| `ollama-cloud/` | `OLLAMA_API_KEY` |
| `openai/` | `OPENAI_API_KEY` |
| `anthropic/` | `ANTHROPIC_API_KEY` |
| `google/` | `GOOGLE_API_KEY` |
| `groq/` | `GROQ_API_KEY` |

Use this convention throughout the generated `.env.example`, env table, and README.

---

### Step 2 — Core pattern

Every Mastra agent follows this pattern:

```
User message → agent.generate(prompt) → [tool calls, automatic] → result.text → reply
```

#### A. System prompt

The system prompt defines the agent's behaviour and domain rules:
- Define the agent's expertise and domain
- State behavioural rules (what the agent will and won't do)
- Describe when to use tools and when to decline
- Keep it concise — the LLM generates plain text responses by default

```ts
const SYSTEM_PROMPT = `
You are a helpful [domain expert] that gives succinct responses.

Important rules:
1. Use the [tool_name] tool for every [domain] question about a supported [entity].
2. Supported [entities] are [list].
3. If the [entity] is unsupported, decline and mention the supported [entities].
4. If the question is not about [domain], decline.
5. Keep responses concise and practical.
`;
```

#### B. Tools

Tools are functions the LLM calls to fetch external data or take actions. Mastra handles the tool loop automatically — define tools and the agent decides when to invoke them.

Design principles:
- One tool, one responsibility
- Define a type for the tool's return value
- **Catch all errors inside the tool** and return a graceful error string — never throw to the agent

```ts
import { createTool } from '@mastra/core/tools';

type MyToolResult = { field1: string; field2: number };

const myTool = createTool({
    id: 'tool_name',
    description: 'What this tool does and when to use it.',
    inputSchema: z.object({ param: z.string().describe('Description of param') }),
    execute: async ({ context }) => {
        try {
            const data = await doWork(context.param);
            return data;
        } catch (err) {
            return `Tool error: ${err instanceof Error ? err.message : String(err)}`;
        }
    },
});
```

> **Note:** When Large Language Platform observability is used (Step 4), the `execute` function is wrapped differently — see that section for the updated tool pattern.

#### C. Agent factory

Mastra's model router accepts a string in the format `provider/model-name` and handles provider resolution and API key lookup automatically (see Step 1 for provider prefix → env var mapping).

```ts
import { Agent } from '@mastra/core/agent';

function createMyAgent(model: string) {
    return new Agent({
        id: '<agent-name>',
        name: '<agent-name>',
        instructions: SYSTEM_PROMPT,
        model,
        tools: { tool_name: myTool },   // omit or use {} if no tools
    });
}
type MyAgent = ReturnType<typeof createMyAgent>;
```

#### D. Handling messages

The agent generates a plain text response. Access it via `result.text`:

```ts
async function handleMessage(agent: MyAgent, prompt: string): Promise<string> {
    try {
        const result = await agent.generate(prompt);
        return result.text;
    } catch (error) {
        console.error('Agent execution failed:', error);
        return "I'm sorry, I encountered an error processing your request.";
    }
}
```

#### E. Entry point

Use the ESM-compatible dotenv pattern and a keep-alive promise:

```ts
import { dirname, join } from 'node:path';
import { fileURLToPath } from 'node:url';
import { config } from 'dotenv';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
config({ path: join(__dirname, '.env') });

async function main(): Promise<void> {
    const model = process.env.MODEL_NAME ?? 'ollama-cloud/gpt-oss:120b';
    const agent = createMyAgent(model);

    // wire up your I/O here (HTTP, stdin, WebSocket, etc.)

    try {
        console.log(`Agent initialized model=${model}`);
        // start your server/client here
        await new Promise(() => {}); // keep-alive
    } catch (error) {
        console.error('Fatal error:', error);
        process.exit(1);
    }
}

void main();
```

Env defaults:

| Variable | Default |
|----------|---------|
| `MODEL_NAME` | `ollama-cloud/gpt-oss:120b` |
| `{PROVIDER}_API_KEY` | *(required — derived from model provider, see Step 1)* |

---

### Step 3 — Optional enhancements

#### Multi-turn conversation

Pass `threadId` to `agent.generate()` — Mastra manages conversation history internally, no manual message array needed:

```ts
const result = await agent.generate(prompt, {
    threadId: sessionId,      // stable per-session identifier (e.g. sender ID)
    resourceId: agentName,
});
```

#### Structured output

If the user requested structured output, pass a Zod schema as the `output` option to `agent.generate()` and access the typed result via `result.object` instead of `result.text`:

```ts
const responseSchema = z.union([
    z.object({
        type: z.literal('analysis'),
        category: z.string(),
        recommendation: z.string(),
    }),
    z.object({
        type: z.literal('decline'),
        reason: z.string(),
    }),
]);
type AgentResponse = z.infer<typeof responseSchema>;

const result = await agent.generate(prompt, {
    output: responseSchema,
});
const response = result.object as AgentResponse;
```

When using structured output:
- Do NOT include JSON format instructions in the system prompt — the Zod schema handles format enforcement at the provider level
- Add response formatters to convert the typed object to a human-readable string for display

---

### Step 4 — Large Language Platform observability (optional)

Add Large Language Platform connectivity for managed routing, tool-call tracing, and observability after the core agent is working.

**Add dependency:** `llpsdk`

#### Wrap tools with annotation tracing

When using Large Language Platform, replace the standard `execute` function with `wrapWithLLPAnnotation`. This changes the signature — the wrapped function receives `inputData` directly (not `{ context }`):

```ts
import { wrapWithLLPAnnotation } from 'llpsdk/mastra';

type MyToolResult = { field1: string; field2: number };

const myTool = createTool({
    id: 'tool_name',
    description: 'What this tool does and when to use it.',
    inputSchema: z.object({ param: z.string() }),
    execute: wrapWithLLPAnnotation<{ param: string }, MyToolResult>(
        'tool_name',
        async (inputData) => {
            return await doWork(inputData.param);
        },
    ),
});
```

#### Implement `handleMessage`

Add `RequestContext` to pass LLP tracing context through the agent call:

```ts
import { RequestContext } from '@mastra/core/request-context';
import { type LLPMastraContext } from 'llpsdk/mastra';

async function handleMessage(
    agent: MyAgent,
    message: TextMessage,
    annotater: Annotater,
): Promise<string> {
    const requestContext = new RequestContext<LLPMastraContext>();
    requestContext.set('llpMessage', message);
    requestContext.set('llpAnnotater', annotater);

    try {
        const result = await agent.generate(message.prompt, { requestContext });
        return result.text;
    } catch (error) {
        console.error('Agent execution failed:', error);
        return "I'm sorry, I encountered an error processing your request.";
    }
}
```

#### Wire up the LLP client in `main()`

The LLP client manages the agent lifecycle — `onStart` creates the agent, `onMessage` handles each incoming message, `onStop` runs cleanup:

```ts
import { type Annotater, LLPClient, type TextMessage } from 'llpsdk';

async function main(): Promise<void> {
    const agentName = process.env.AGENT_NAME ?? '<agent-name>';
    const apiKey = process.env.LLP_API_KEY ?? '';
    const model = process.env.MODEL_NAME ?? 'ollama-cloud/gpt-oss:120b';

    if (!apiKey) {
        throw new Error('LLP_API_KEY env var is not defined');
    }

    const client = new LLPClient(agentName, apiKey)
        .onStart(() => createMyAgent(model))
        .onMessage(async (agent, msg, annotater) => {
            const response = await handleMessage(agent, msg, annotater);
            return msg.reply(response);
        })
        .onStop(() => {
            console.log('session ended');
        });

    try {
        console.log(`Agent initialized model=${model}`);
        await client.connect();
        console.log('Connected to platform');
        await new Promise(() => {});
    } catch (error) {
        console.error('Fatal error:', error);
        process.exit(1);
    }
}

void main();
```

Additional env vars when using Large Language Platform:

| Variable | Default |
|----------|---------|
| `AGENT_NAME` | `<agent-name>` |
| `LLP_API_KEY` | *(required)* |
| `MODEL_NAME` | *(from Step 1 LLM choice)* |
| `{PROVIDER}_API_KEY` | *(required — derived from model provider, see Step 1)* |

---

### Step 5 — Generate the output

Generate a dedicated directory named `<agent-name>/` containing:

**`<agent-name>.ts`** — single source file, sections in this order:
1. Imports
2. Dotenv config load (`dirname`/`fileURLToPath` pattern for ESM)
3. `SYSTEM_PROMPT`
4. Domain data + types
5. Tool definitions (if any)
6. Agent factory + type
7. `handleMessage()`
8. `main()`
9. `void main()`

Separate sections with:
```ts
// =============================================================================
// Section Name
// =============================================================================
```

**Supporting files:**
- `package.json` — dependencies: `@mastra/core`, `zod`, `dotenv` (add `llpsdk` if observability requested); devDependencies: `tsx`, `typescript`, `@types/node`; script: `"start": "tsx <agent-name>.ts"`; set `"type": "module"`
- `tsconfig.json` — target ES2022, module NodeNext, strict true, include `["<agent-name>.ts"]`
- `.env.example` — all env vars with inline comments
- `README.md` — what it does, prerequisites, setup steps, config table, and exact run instructions

`.env.example` must include the correct `{PROVIDER}_API_KEY` variable for the chosen model provider (see Step 1 table) with an inline comment explaining what it is for and how to obtain it.

`README.md` must include:
- a short description of the agent and its tools
- prerequisites, including Node.js and any required provider/API setup
- setup steps: `cd <agent-name>`, `npm install`, copy `.env.example` to `.env`, fill in env vars
- a configuration table documenting every env var used by the example
- a run section showing `npm start`
- a brief note explaining what the agent will do when it receives a message
