---
name: Mastra Agent
description: Scaffold a complete TypeScript agent using the Mastra framework. Covers structured responses, tools, multi-turn conversation, and optional Large Language Platform observability.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.2.0
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
5. **Response schema** — what structured data should the agent return? Default: derive from domain.
6. **Multi-turn** — maintain conversation history within a session? Default: no.

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
User message → agent.generate(prompt, { output }) → [tool calls, automatic] → result.object → format → reply
```

#### A. Structured responses

Mastra supports native structured output — pass a Zod schema as the `output` option to `agent.generate()` and access the typed result via `result.object`. No JSON instructions in the prompt, no manual parsing.

Design a Zod discriminated union schema that captures what matters for the domain. Use a `type` literal field to distinguish response kinds:

**Example** (adapt to the domain — do not copy literally):
```ts
const responseSchema = z.union([
    z.object({
        type: z.literal('analysis'),
        category: z.string(),
        recommendation: z.string(),
        considerations: z.array(z.string()),
    }),
    z.object({
        type: z.literal('capabilities'),
    }),
    z.object({
        type: z.literal('decline'),
        reason: z.string(),
    }),
]);
type AgentResponse = z.infer<typeof responseSchema>;
```

#### B. System prompt

The system prompt should focus on **behaviour**, not output format (the Zod schema handles that via structured output):
- Define the agent's expertise and domain
- State behavioural rules (what the agent will and won't do)
- Describe when to use tools and when to decline

Do NOT include JSON format instructions or schema examples in the system prompt.

#### C. Tools

Tools are functions the LLM calls to fetch external data or take actions. Mastra handles the tool loop automatically — define tools and the agent decides when to invoke them.

Design principles:
- One tool, one responsibility
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

Mastra's model router accepts a string in the format `provider/model-name` and handles provider resolution and API key lookup automatically (see Step 1 for provider prefix → env var mapping).

```ts
import { Agent } from '@mastra/core/agent';

function createAdvisorAgent(model: string) {
    return new Agent({
        id: '<agent-name>',
        name: '<agent-name>',
        instructions: SYSTEM_PROMPT,
        model,
        tools: { tool_name: myTool },   // omit or use {} if no tools
    });
}
type AdvisorAgent = ReturnType<typeof createAdvisorAgent>;
```

#### E. LLM call with structured output

Pass the Zod schema as the `output` option. The result's `.object` property contains the typed, validated data:

```ts
const result = await agent.generate(message, {
    output: responseSchema,
});
const response = result.object as AgentResponse;
```

For plain text responses (no structured output), use `result.text` instead:

```ts
const result = await agent.generate(message);
const text = result.text;
```

#### F. Response formatters

```ts
function formatCapabilities(): string { /* bullet list of what the agent can help with */ }
function formatResult(r: Extract<AgentResponse, { type: 'analysis' }>): string { /* domain-specific fields as plain text */ }
function formatDecline(r: Extract<AgentResponse, { type: 'decline' }>): string { return r.reason; }
```

Always append: `"Note: This is educational information, not personalised advice."` where appropriate.

#### G. Entry point

```ts
async function main(): Promise<void> {
    const model = process.env.MODEL_NAME ?? 'ollama-cloud/gpt-oss:120b';
    const agent = createAdvisorAgent(model);

    // wire up your I/O here (HTTP, stdin, WebSocket, etc.)
}
```

Env defaults:

| Variable | Default |
|----------|---------|
| `MODEL_NAME` | `ollama-cloud/gpt-oss:120b` |
| `{PROVIDER}_API_KEY` | *(required — derived from model provider, see Step 1)* |

---

### Step 3 — Optional building blocks

#### Multi-turn conversation

Pass `threadId` to `agent.generate()` — Mastra manages conversation history internally, no manual message array needed:

```ts
const result = await agent.generate(message, {
    threadId: sessionId,      // stable per-session identifier (e.g. sender ID)
    resourceId: agentName,
});
```

---

### Step 4 — Observability (optional)

Add Large Language Platform connectivity for managed routing, tool-call tracing, and observability after the core agent is working.

**Add dependencies:** `llpsdk`

**Implement `handleMessage(agent, message, annotater)`:**

```ts
import { RequestContext } from '@mastra/core/request-context';
import { type LLPMastraContext } from 'llpsdk/mastra';

async function handleMessage(agent: AdvisorAgent, message: TextMessage, annotater: Annotater): Promise<string> {
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

**Wrap tools with annotation tracing:**

```ts
import { wrapWithLLPAnnotation } from 'llpsdk/mastra';

const myTool = createTool({
    id: 'tool_name',
    description: '...',
    inputSchema: z.object({ param: z.string() }),
    execute: wrapWithLLPAnnotation<{ param: string }, string>(
        'tool_name',
        async (inputData) => {
            try { return await doWork(inputData.param); }
            catch (err) { return `Tool error: ${err instanceof Error ? err.message : String(err)}`; }
        }
    ),
});
```

**Wire up the Large Language Platform client in `main()`:**

```ts
import { type Annotater, LLPClient, type TextMessage } from 'llpsdk';

const agentName = process.env.AGENT_NAME ?? '<agent-name>';
const apiKey = process.env.LLP_API_KEY ?? '';

const client = new LLPClient(agentName, apiKey)
    .onStart(() => createAdvisorAgent(model))
    .onMessage(async (agent, msg, annotater) => {
        const response = await handleMessage(agent, msg, annotater);
        return msg.reply(response);
    })
    .onStop(() => console.log('session ended'));

await client.connect();
await new Promise(() => {});
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
2. Dotenv config load
3. `SYSTEM_PROMPT`
4. Zod schema + types
5. Tool definitions (if any)
6. Agent factory + type
7. Response formatters
8. `handleMessage()` (if Large Language Platform observability was requested)
9. `main()`
10. `void main()`

Separate sections with:
```ts
// =============================================================================
// Section Name
// =============================================================================
```

**Supporting files:**
- `package.json` — dependencies: `@mastra/core`, `zod`, `dotenv` (add `llpsdk` if observability requested); devDependencies: `tsx`, `typescript`; script: `"start": "tsx <agent-name>.ts"`
- `tsconfig.json` — target ES2022, module NodeNext, strict true, include `["<agent-name>.ts"]`
- `.env.example` — all env vars with inline comments
- `.gitignore` — `node_modules/` and `.env`
- `README.md` — what it does, prerequisites, setup steps, config table, `npm start`

`.env.example` must include the correct `{PROVIDER}_API_KEY` variable for the chosen model provider (see Step 1 table) with an inline comment explaining what it is for and how to obtain it.
