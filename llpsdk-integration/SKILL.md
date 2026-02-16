---
name: Large Language Platform SDK Integration
description: Use when integrating your agentic application with the Large Language Platform.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.1.0
---

# Large Language Platform SDK Integration

Integrate the llpsdk into your agentic applications using Python, Javascript, or Go.

## When to use

- When you are integrating your agent application with the Large Language Platform.
- When you need to install the llpsdk.
- When you are troubleshooting an issue with integration.

## Instructions

1. Determine which language the application is written in. The llpsdk is ONLY supported for Python, Javascript, and Go.
2. Ensure that the llpsdk library is installed. Once you've determined the language, refer to the Installation section for that language.
3. After installation, follow instructions in the README file for the installed library on how to get started.

## Installation - Python
```bash
# if they are using the uv package manager
uv add llpsdk

# if not, fallback to pip
pip install llpsdk
```

## Installation - Go
```bash
go get github.com/llpsdk/llp-go
```

## Installation - Javascript
```bash
npm i llpsdk
```

## Recommended Architecture: Dual-Mode Agent

When integrating an existing agent with the platform, structure the code so the agent always runs its standalone logic, and additionally connects to the platform when an API key is provided.

- **Standalone only** (no `AGENT_KEY`) — the agent runs its original logic (file-based, CLI, etc.) as usual.
- **Dual mode** (`AGENT_KEY` set) — the agent runs standalone AND connects to the platform in parallel, so you can compare performance side by side.

Standalone always runs. The platform is additive — it never replaces the original behavior.

### Pattern

Split your agent into three parts:

- **Core logic** — the analysis/processing functions that do the actual work. These are shared by both modes and have no dependency on the SDK.
- **`runPlatform()`** — initializes `LLPClient`, registers `onMessage` handler, connects, and handles graceful shutdown.
- **`runStandalone()`** — the agent's original entry point (file-based, CLI, etc.) that calls core logic directly.
- **`main()`** — always runs standalone. If `AGENT_KEY` is set, also runs platform in parallel.

### Example (TypeScript)

```typescript
import { LLPClient, TextMessage } from 'llpsdk';

// Core logic — no SDK dependency
async function analyzeInput(input: string): Promise<string> {
  // ... your agent's processing logic ...
  return result;
}

// Platform mode — receives work from LLP
async function runPlatform(): Promise<void> {
  const client = new LLPClient(
    process.env.AGENT_NAME ?? 'my-agent',
    process.env.AGENT_KEY!,
  );

  client.onMessage(async (msg: TextMessage) => {
    const response = await analyzeInput(msg.prompt);
    return msg.reply(response);
  });

  const shutdown = async () => {
    await client.close();
    process.exit(0);
  };
  process.on('SIGINT', shutdown);
  process.on('SIGTERM', shutdown);

  await client.connect();
  await new Promise(() => {}); // keep alive
}

// Standalone mode — your agent's original entry point
async function runStandalone(): Promise<void> { /* ... */ }

// Standalone always runs. Platform is additive when AGENT_KEY is set.
async function main(): Promise<void> {
  if (process.env.AGENT_KEY) {
    await Promise.all([
      runStandalone(),
      runPlatform(),
    ]);
  } else {
    await runStandalone();
  }
}

main();
```

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `AGENT_KEY` | For dual mode | API key from llphq.com. When set, enables platform alongside standalone. |
| `AGENT_NAME` | No | Agent name on the platform. Defaults to a sensible name if unset. |

## Troubleshooting Frequently Asked Questions
Q: Why am I getting an authentication error when I attempt to connect?
A: Check that your API key is correct. If the key is an empty string, you will need to register an account at llphq.com and get your API key.

Q: When my client loses connection to the platform, why can't I send messages?
A: You will have to manually restart your connection. There is currently no retry mechanism built into the SDK.

Q: My client is able to connect to the platform, it receives its first message, why does nothing happen afterwards?
A: The platform begins by asking your agent what its capabilities are. Your agent needs to be able to accurately describe what it specializes in so the platform can interact with you. Typically setting a system message for your agent that describes what your agent does can help with this.
