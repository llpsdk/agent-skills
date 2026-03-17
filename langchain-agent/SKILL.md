---
name: LangChain Agent
description: Scaffold a complete LangChain agent in TypeScript or Python. Covers structured responses, tools, multi-turn conversation, memory, and optional LLP observability.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.1.0
---

# LangChain Agent

Scaffold a complete, runnable agent using [LangChain](https://js.langchain.com) in TypeScript or Python.

## When to use

- When building an agent with LangChain in TypeScript or Python.
- When you need tool calling, structured responses, and optional conversation history.

---

## Instructions

### Step 1 — Gather requirements

Ask the user for the following. Do not proceed until all required items are answered.

**Required:**
1. **Language** — `typescript` or `python`
2. **Agent name** — kebab-case (e.g. `financial-advisor`)
3. **Domain** — one sentence describing what the agent specialises in
4. **LLM** — Ollama model (default: `llama3.1`) or OpenAI/Anthropic model if preferred
5. **Tools** — list the tools this agent needs. For each: name, description, inputs, return value. If the agent needs no tools, confirm this explicitly.

**Optional:**
6. **Response schema** — what structured data should the agent return? Default: derive from domain.
7. **Multi-turn** — maintain conversation history within a session? Default: no.
8. **Memory** — persist memory across sessions? Default: no.

---

### Step 2 — Core pattern

Every LangChain agent follows this pattern:

```
User message → buildPrompt() → agent.invoke() → [tool calls, automatic] → parseResponse() → format → reply
```

#### A. Structured responses

Design a schema that captures what matters for the domain. It must cover at minimum:
- A primary response type with domain-specific fields
- `"capabilities"` — for "what can you do?" questions
- `"decline"` — for out-of-scope questions, with a `reason` field

- **TypeScript:** use `zod` — add `z.string()` as a union fallback on `type` for graceful handling of unexpected output
- **Python:** use `pydantic` — mark non-universal fields as `Optional`

**TypeScript example:**
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

**Python example:**
```python
from pydantic import BaseModel
from typing import Optional, List

class AgentResponse(BaseModel):
    type: str
    category: Optional[str] = None
    recommendation: Optional[str] = None
    considerations: Optional[List[str]] = None
    reason: Optional[str] = None
```

#### B. System prompt

The system prompt must:
- Instruct the LLM to return ONLY valid JSON matching the schema
- Show the exact JSON format for every response type
- Define the agent's expertise areas and categories
- State behavioural rules (what the agent will and won't do)

#### C. Tools

LangChain handles the tool calling loop automatically. Define tools and the agent decides when to invoke them.

Design principles:
- One tool, one responsibility
- Return a plain string — the LLM reads tool output as text
- **Catch all errors inside the tool** and return a graceful error string — never raise to the agent

**TypeScript:**
```ts
import { tool } from 'langchain';

const myTool = tool(
    async ({ param }: { param: string }) => {
        try {
            return await doWork(param);
        } catch (err) {
            return `Tool error: ${err instanceof Error ? err.message : String(err)}`;
        }
    },
    {
        name: 'tool_name',
        description: 'What this tool does and when to use it.',
        schema: z.object({ param: z.string().describe('Description of param') }),
    }
);
```

**Python:**
```python
from langchain_core.tools import tool

@tool
def tool_name(param: str) -> str:
    """What this tool does and when to use it."""
    try:
        return do_work(param)
    except Exception as e:
        return f"Tool error: {e}"
```

#### D. Agent factory

**TypeScript:**
```ts
import { createAgent } from 'langchain';
import { ChatOllama } from '@langchain/ollama';

export function createAdvisorAgent(llm: ChatOllama) {
    return createAgent({
        model: llm,
        tools: [myTool],        // empty array if no tools
        systemPrompt: SYSTEM_PROMPT,
    });
}
export type AdvisorAgent = ReturnType<typeof createAdvisorAgent>;
```

**Python:**
```python
from langchain_ollama import ChatOllama
from langchain.agents import create_agent

def create_advisor_agent(llm: ChatOllama):
    return create_agent(
        model=llm,
        tools=[my_tool],        # empty list if no tools
        system_prompt=SYSTEM_PROMPT,
    )
```

#### E. LLM call

**TypeScript:**
```ts
import { HumanMessage } from '@langchain/core/messages';

const result = await agent.invoke({
    messages: [...conversationHistory, new HumanMessage(buildPrompt(message))],
});
const lastMessage = result.messages[result.messages.length - 1];
const rawText = extractTextContent(lastMessage?.content);
```

`extractTextContent` helper — required because LangChain message content can be a string or array:
```ts
function extractTextContent(content: unknown): string {
    if (typeof content === 'string') return content;
    if (Array.isArray(content)) {
        return content.map(item =>
            typeof item === 'string' ? item :
            (item && typeof item === 'object' && 'text' in item)
                ? String((item as { text?: unknown }).text ?? '') : ''
        ).filter(Boolean).join('\n');
    }
    return String(content ?? '');
}
```

**Python:**
```python
result = await agent.ainvoke({
    "messages": [{"role": "user", "content": build_prompt(message)}],
})
raw_text = result["messages"][-1].content
```

Note: The Python LangChain agent is async — `main()` must be an `async def` run via `asyncio.run(main())`.

#### F. Response parsing

**TypeScript:**
```ts
function extractJson(text: string): string {
    const block = text.match(/```(?:json)?\s*([\s\S]*?)\s*```/);
    return block ? block[1].trim() : text.trim();
}

function parseResponse(text: string): AgentResponse {
    return responseSchema.parse(JSON.parse(extractJson(text)));
}
```

**Python:**
```python
import re, json

def extract_json(text: str) -> str:
    match = re.search(r"```(?:json)?\s*([\s\S]*?)\s*```", text)
    return match.group(1).strip() if match else text.strip()

def parse_response(text: str) -> AgentResponse:
    return AgentResponse.model_validate(json.loads(extract_json(text)))
```

Wrap `parseResponse` / `parse_response` in error handling — a parse failure returns a safe fallback string.

#### G. Response formatters

Define a formatter for each response type. Keep them pure — no LLM calls, no I/O.

- `formatCapabilities()` — human-readable bullet list of what the agent can help with
- `formatResult(response)` — domain-specific fields as plain text
- `formatDecline(response)` — returns the decline reason or a default message

Append `"Note: This is educational information, not personalised advice."` where appropriate.

#### H. Prompt builder

**TypeScript:** `return \`${message}\n\nReturn ONLY valid JSON matching the required schema.\`;`

**Python:** `return f"{message}\n\nReturn ONLY valid JSON matching the required schema."`

#### I. Entry point

**TypeScript:**
```ts
async function main(): Promise<void> {
    const llm = new ChatOllama({
        baseUrl: process.env.OLLAMA_HOST ?? 'http://localhost:11434',
        model: process.env.OLLAMA_MODEL ?? 'llama3.1',
    });
    const agent = createAdvisorAgent(llm);
    // wire up your I/O here
}
```

**Python:**
```python
async def main() -> None:
    llm = ChatOllama(
        base_url=os.getenv("OLLAMA_HOST", "http://localhost:11434"),
        model=os.getenv("OLLAMA_MODEL", "llama3.1"),
    )
    agent = create_advisor_agent(llm)
    # wire up your I/O here

if __name__ == "__main__":
    asyncio.run(main())
```

Env defaults:

| Variable | Default |
|----------|---------|
| `OLLAMA_MODEL` | `llama3.1` |
| `OLLAMA_HOST` | `http://localhost:11434` |

---

### Step 3 — Optional building blocks

#### Multi-turn conversation

Maintain a conversation history list scoped to the session. Key it by a stable session or sender identifier. Append after each successful LLM call. Cap to avoid token bloat (drop oldest pair at ~20 turns).

**TypeScript:** history is a `(HumanMessage | AIMessage)[]` array. Append `new HumanMessage(prompt)` and `new AIMessage(rawText)` after each exchange.

**Python:** history is a list of `HumanMessage` / `AIMessage` instances from `langchain_core.messages`. Append after each exchange.

#### Memory across sessions

For in-session memory only, conversation history above is sufficient.

For persistent memory across sessions, store and retrieve keyed by a stable user or session identifier:
- **Simple:** serialise history to disk (JSON file)
- **Production:** Redis or a database, keyed by sender ID
- **Semantic:** embed past exchanges and retrieve relevant ones by similarity before each call

Always wrap memory retrieval in error handling — a failure must never block the agent from responding.

---

### Step 4 — Observability (optional)

Add LLP platform connectivity for managed routing, tool-call tracing, and observability after the core agent is working.

**Add dependencies:**
- TypeScript: `llpsdk`
- Python: `llpsdk`

**Implement `handleMessage(agent, message, annotater)`:**

```
1. Derive corrId from message (e.g. message.id[:8])
2. Log: [corrId] >>> RECV from=<sender> prompt="<first 80 chars>"
3. Call agent with prompt (and runtimeContext/annotater where applicable)
4. Parse response
5. Log: [corrId] === type=<type>
6. Switch on type → format → return string
7. On error: log [corrId] !!! → return safe fallback string
```

**TypeScript — wire tool tracing via middleware:**

```ts
import { createLLPToolMiddleware } from 'llpsdk/langchain';

export function createAdvisorAgent(llm: ChatOllama) {
    return createAgent({
        model: llm,
        tools: [myTool],
        middleware: [createLLPToolMiddleware()],
        systemPrompt: SYSTEM_PROMPT,
    });
}
```

Pass LLP context when invoking the agent:
```ts
const result = await agent.invoke(
    { messages: [...history, new HumanMessage(buildPrompt(message.prompt))] },
    { context: { llpMessage: message, llpClient: annotater } },
);
```

**TypeScript — wire LLPClient in `main()`:**
```ts
import { LLPClient } from 'llpsdk';

const client = new LLPClient(
    process.env.AGENT_NAME ?? '<agent-name>',
    process.env.AGENT_KEY ?? '',
    { url: process.env.PLATFORM_ADDRESS ?? 'ws://localhost:4000/agent/websocket' }
)
    .onStart(() => createAdvisorAgent(llm))
    .onMessage(async (agent, msg, annotater) => msg.reply(await handleMessage(agent, msg, annotater)))
    .onStop(() => console.log('session ended'));

process.on('SIGINT', async () => { await client.close(); process.exit(0); });
process.on('SIGTERM', async () => { await client.close(); process.exit(0); });

await client.connect();
await new Promise(() => {});
```

**Python — wire tool tracing via middleware and LLP client in `main()`:**

Add `LLPAnnotationMiddleware` to the agent factory:
```python
from llpsdk.langchain import LLPAnnotationMiddleware

def create_advisor_agent(llm):
    return create_agent(
        model=llm,
        tools=[my_tool],
        middleware=[LLPAnnotationMiddleware()],
        system_prompt=SYSTEM_PROMPT,
    )
```

Pass `message` and `annotater` when invoking the agent so the middleware can annotate tool calls:
```python
result = await agent.ainvoke({
    "messages": [{"role": "user", "content": build_prompt(msg.prompt)}],
    "message": msg,
    "annotater": annotater,
})
raw_text = result["messages"][-1].content
```

Define a `format_response` helper to dispatch to the right formatter:
```python
def format_response(response: AgentResponse) -> str:
    if response.type == "capabilities":
        return format_capabilities()
    if response.type == "decline":
        return format_decline(response)
    return format_result(response)
```

Wire the LLP client in `main()`. Note the `on_message` signature is `(agent, annotater, msg)`:
```python
import asyncio
import llpsdk as llp

async def main() -> None:
    load_dotenv()
    cfg = llp.Config(platform_url=os.getenv("LLP_URL"))
    client = llp.Client("<agent-name>", os.getenv("LLP_API_KEY", ""), cfg)

    def on_start():
        llm = ChatOllama(
            base_url=os.getenv("OLLAMA_HOST", "http://localhost:11434"),
            model=os.getenv("OLLAMA_MODEL", "llama3.1"),
        )
        return create_advisor_agent(llm)

    async def on_message(agent, annotater: llp.Annotater, msg: llp.TextMessage) -> llp.TextMessage:
        try:
            result = await agent.ainvoke({
                "messages": [{"role": "user", "content": build_prompt(msg.prompt)}],
                "message": msg,
                "annotater": annotater,
            })
            raw_text = result["messages"][-1].content
            response = parse_response(raw_text)
            # switch on response.type → format → return reply
            return msg.reply(format_response(response))
        except Exception as e:
            print(f"Error: {e}")
            return msg.reply("I'm sorry, I encountered an error processing your request.")

    client.on_start(on_start)
    client.on_message(on_message)

    try:
        await client.connect()
        print("Agent running. Press Ctrl+C to exit...")
        await asyncio.Event().wait()
    except KeyboardInterrupt:
        print("\nShutting down...")
    finally:
        await client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

Additional env vars when using LLP:

| Variable | Description |
|----------|-------------|
| `LLP_URL` | Platform WebSocket URL |
| `LLP_API_KEY` | API key for the LLP platform |

---

### Step 5 — Generate the output

Generate a dedicated directory named `<agent-name>/`.

---

#### TypeScript output

**`<agent-name>.ts`** — sections in this order:
1. File header comment
2. Dotenv config load (before other imports)
3. Imports
4. `SYSTEM_PROMPT`
5. Zod schema + type
6. Tool definitions (if any)
7. Agent factory + exported type
8. `extractTextContent()` helper
9. Formatters
10. `buildPrompt()`, `extractJson()`, `parseResponse()`
11. `handleMessage()` (if LLP observability requested)
12. `main()`
13. `isMainModule` guard + `void main()`

Separate sections with:
```ts
// =============================================================================
// Section Name
// =============================================================================
```

Supporting files:
- `package.json` — dependencies: `@langchain/core`, `@langchain/ollama`, `langchain`, `zod`, `dotenv` (add `llpsdk` if observability requested); devDependencies: `tsx`, `typescript`; script: `"start": "tsx <agent-name>.ts"`
- `tsconfig.json` — target ES2022, module NodeNext, strict true
- `.env.example` — all env vars with inline comments
- `.gitignore` — `node_modules/` and `.env`
- `README.md` — what it does, prerequisites, setup, config table, `npm start`

---

#### Python output

**`<agent-name>.py`** — sections in this order:
1. File header comment
2. Standard library imports
3. Third-party imports
4. `load_dotenv()`
5. `SYSTEM_PROMPT`
6. Pydantic response model
7. Tool definitions (if any)
8. Agent factory
9. Formatters + `format_response()` dispatcher
10. `build_prompt()`, `extract_json()`, `parse_response()`
11. `handle_message()` / inline LLP `on_message` handler (if LLP observability requested)
12. `async def main()`
13. `if __name__ == "__main__": asyncio.run(main())`

Separate sections with:
```python
# =============================================================================
# Section Name
# =============================================================================
```

Supporting files:
- `requirements.txt` — `langchain`, `langchain-ollama`, `langchain-core`, `pydantic`, `python-dotenv` (add `llpsdk` if observability requested)
- `.env.example` — all env vars with inline comments
- `.gitignore` — `__pycache__/`, `*.pyc`, `.env`, `venv/`
- `README.md` — what it does, prerequisites (`python 3.11+`), `pip install -r requirements.txt`, config table, `python <agent-name>.py`
