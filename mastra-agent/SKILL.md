---
name: Mastra Agent
description: Scaffold a complete TypeScript agent using the Mastra framework. Covers structured responses, tools, multi-turn conversation, and optional LLP observability.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.1.0
---

# Mastra Agent

Scaffold a complete, runnable TypeScript agent using [Mastra](https://mastra.ai).

## When to use

- When building a TypeScript agent with the Mastra framework.
- When you want structured responses, tool calling, and optional multi-turn conversation with minimal boilerplate.

---

## Instructions

### Step 1 — Gather requirements

Ask the user for the following. Do not proceed until all required items are answered.

**Required:**
1. **Agent name** — kebab-case (e.g. `loan-advisor`, `code-reviewer`)
2. **Domain** — one sentence describing what the agent specialises in
3. **LLM** — Ollama model to use (default: `llama3.1`)
4. **Tools** — list the tools this agent needs. For each: name, description, inputs, return value. If the agent needs no tools, confirm this explicitly.

**Optional:**
5. **Response schema** — what structured data should the agent return? Default: derive from domain.
6. **Multi-turn** — maintain conversation history within a session? Default: no.

---

### Step 2 — Core pattern

Every Mastra agent follows this pattern:

```
User message → buildPrompt() → agent.generate() → [tool calls, automatic] → parseResponse() → format → reply
```

#### A. Structured responses

Design a Zod schema that captures what matters for the domain. It must include a `type` field covering at minimum:
- A primary response type (e.g. `"analysis"`) with domain-specific fields
- `"capabilities"` — for "what can you do?" questions
- `"decline"` — for out-of-scope questions, with a `reason` field

Use `.optional()` for fields that only apply to some types. Add `z.string()` as a fallback union member on `type` to handle unexpected LLM output gracefully.

**Example** (adapt to the domain — do not copy literally):
```ts
const responseSchema = z.object({
    type: z.union([z.literal('analysis'), z.literal('capabilities'), z.literal('decline'), z.string()]),
    category: z.string().optional(),
    recommendation: z.string().optional(),
    considerations: z.array(z.string()).optional(),
    reason: z.string().optional(),
});
type AgentResponse = z.infer<typeof responseSchema>;
```

#### B. System prompt

The system prompt must:
- Instruct the LLM to return ONLY valid JSON matching the schema
- Show the exact JSON format for every response type
- Define the agent's expertise areas and categories
- State behavioural rules (what the agent will and won't do)

#### C. Tools

Tools are functions the LLM calls to fetch external data or take actions. Mastra handles the tool loop automatically — define tools and the agent decides when to invoke them.

Design principles:
- One tool, one responsibility
- Return a plain string — the LLM reads tool output as text
- **Catch all errors inside the tool** and return a graceful error string — never throw to the agent

```ts
import { createTool } from '@mastra/core/tools';

const myTool = createTool({
    id: 'tool_name',
    description: 'What this tool does and when to use it.',
    inputSchema: z.object({ param: z.string().describe('Description of param') }),
    execute: async ({ context }) => {
        try {
            return await doWork(context.param);
        } catch (err) {
            return `Tool error: ${err instanceof Error ? err.message : String(err)}`;
        }
    },
});
```

#### D. Agent factory

```ts
import { Agent } from '@mastra/core';
import { createOllama } from 'ollama-ai-provider';

type OllamaModel = ReturnType<ReturnType<typeof createOllama>>;

export function createAdvisorAgent(model: OllamaModel) {
    return new Agent({
        name: '<agent-name>',
        instructions: SYSTEM_PROMPT,
        model,
        tools: { tool_name: myTool },   // omit or use {} if no tools
    });
}
export type AdvisorAgent = ReturnType<typeof createAdvisorAgent>;
```

#### E. LLM call

```ts
const result = await agent.generate(buildPrompt(message));
const rawText = result.text;
```

#### F. Response parsing

```ts
function extractJson(text: string): string {
    const block = text.match(/```(?:json)?\s*([\s\S]*?)\s*```/);
    return block ? block[1].trim() : text.trim();
}

function parseResponse(text: string): AgentResponse {
    return responseSchema.parse(JSON.parse(extractJson(text)));
}
```

Wrap `parseResponse` in try/catch in the message handler — a parse failure returns a safe fallback string.

#### G. Response formatters

```ts
function formatCapabilities(): string { /* bullet list of what the agent can help with */ }
function formatResult(r: AgentResponse): string { /* domain-specific fields as plain text */ }
function formatDecline(r: AgentResponse): string { return r.reason ?? 'I can only help with <domain> questions.'; }
```

Always append: `"Note: This is educational information, not personalised advice."` where appropriate.

#### H. Prompt builder

```ts
function buildPrompt(message: string): string {
    return `${message}\n\nReturn ONLY valid JSON matching the required schema.`;
}
```

#### I. Entry point

```ts
async function main(): Promise<void> {
    const ollama = createOllama({ baseURL: `${process.env.OLLAMA_HOST ?? 'http://localhost:11434'}/api` });
    const model = ollama(process.env.MODEL_NAME ?? 'llama3.1');
    const agent = createAdvisorAgent(model);

    // wire up your I/O here (HTTP, stdin, WebSocket, etc.)
}
```

Env defaults:

| Variable | Default |
|----------|---------|
| `MODEL_NAME` | `llama3.1` |
| `OLLAMA_HOST` | `http://localhost:11434` |

---

### Step 3 — Optional building blocks

#### Multi-turn conversation

Pass `threadId` to `agent.generate()` — Mastra manages conversation history internally, no manual message array needed:

```ts
const result = await agent.generate(buildPrompt(message), {
    threadId: sessionId,      // stable per-session identifier (e.g. sender ID)
    resourceId: agentName,
});
```

---

### Step 4 — Observability (optional)

Add LLP platform connectivity for managed routing, tool-call tracing, and observability after the core agent is working.

**Add dependencies:** `llpsdk`

**Implement `handleMessage(agent, message, annotater)`:**

```
1. Derive corrId from message (e.g. message.id.slice(0, 8))
2. Log: [corrId] >>> RECV from=<sender> prompt="<first 80 chars>"
3. Build RuntimeContext, set llpMessage and llpAnnotater
4. Call agent.generate() with runtimeContext
5. Parse response
6. Log: [corrId] === type=<type>
7. Switch on type → format → return string
8. On error: log [corrId] !!! → return safe fallback string
```

```ts
import { RuntimeContext } from '@mastra/core/runtime-context';
import { type LLPMastraContext } from 'llpsdk/mastra';

export async function handleMessage(agent: AdvisorAgent, message: TextMessage, annotater: Annotater): Promise<string> {
    const corrId = message.id?.slice(0, 8) ?? 'no-id';
    console.log(`[${corrId}] >>> RECV from=${message.sender} prompt="${message.prompt.slice(0, 80)}"`);

    const runtimeContext = new RuntimeContext<LLPMastraContext>();
    runtimeContext.set('llpMessage', message);
    runtimeContext.set('llpAnnotater', annotater);

    try {
        const result = await agent.generate(buildPrompt(message.prompt), { runtimeContext });
        const response = parseResponse(result.text);
        console.log(`[${corrId}] === type=${response.type}`);
        if (response.type === 'capabilities') return formatCapabilities();
        if (response.type === 'decline') return formatDecline(response);
        if (response.type === 'analysis') return formatResult(response);
        return JSON.stringify(response);
    } catch (err) {
        console.error(`[${corrId}] !!! error=${err}`);
        return "I'm sorry, I encountered an error processing your request.";
    }
}
```

**Wrap tools with annotation tracing:**

```ts
import { wrapWithLLPAnnotation } from 'llpsdk/mastra';

const myTool = createTool({
    id: 'tool_name',
    description: '...',
    inputSchema: z.object({ param: z.string() }),
    execute: wrapWithLLPAnnotation<{ param: string }, string>(
        'tool_name',
        async ({ context }) => {
            try { return await doWork(context.param); }
            catch (err) { return `Tool error: ${err instanceof Error ? err.message : String(err)}`; }
        }
    ),
});
```

**Wire up LLPClient in `main()`:**

```ts
import { LLPClient } from 'llpsdk';

const client = new LLPClient(
    process.env.AGENT_NAME ?? '<agent-name>',
    process.env.AGENT_KEY ?? '',
    { url: process.env.PLATFORM_ADDRESS ?? 'ws://localhost:4000/agent/websocket' }
)
    .onStart(() => createAdvisorAgent(model))
    .onMessage(async (agent, msg, annotater) => msg.reply(await handleMessage(agent, msg, annotater)))
    .onStop(() => console.log('session ended'));

process.on('SIGINT', async () => { await client.close(); process.exit(0); });
process.on('SIGTERM', async () => { await client.close(); process.exit(0); });

await client.connect();
await new Promise(() => {});
```

Additional env vars when using LLP:

| Variable | Default |
|----------|---------|
| `AGENT_NAME` | `<agent-name>` |
| `AGENT_KEY` | `''` |
| `PLATFORM_ADDRESS` | `ws://localhost:4000/agent/websocket` |

---

### Step 5 — Generate the output

Generate a dedicated directory named `<agent-name>/` containing:

**`<agent-name>.ts`** — single source file, sections in this order:
1. File header comment
2. Dotenv config load (before other imports)
3. Imports
4. `SYSTEM_PROMPT`
5. Zod schema + type
6. Tool definitions (if any)
7. Agent factory + exported type
8. Formatters
9. `buildPrompt()`, `extractJson()`, `parseResponse()`
10. `handleMessage()` (if LLP observability was requested)
11. `main()`
12. `isMainModule` guard + `void main()`

Separate sections with:
```ts
// =============================================================================
// Section Name
// =============================================================================
```

**Supporting files:**
- `package.json` — dependencies: `@mastra/core`, `ollama-ai-provider`, `zod`, `dotenv` (add `llpsdk` if observability requested); devDependencies: `tsx`, `typescript`; script: `"start": "tsx <agent-name>.ts"`
- `tsconfig.json` — target ES2022, module NodeNext, strict true, include `["<agent-name>.ts"]`
- `.env.example` — all env vars with inline comments
- `.gitignore` — `node_modules/` and `.env`
- `README.md` — what it does, prerequisites, setup steps, config table, `npm start`
