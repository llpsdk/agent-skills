---
name: Go Agent
description: Scaffold a complete Go agent using the OpenAI SDK. Covers structured responses, tools, required in-session conversation history for multi-turn behavior, optional persistent memory, and optional LLP observability.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.1.0
---

# Go Agent

Scaffold a complete, runnable Go agent using the OpenAI SDK.

## When to use

- When building an agent in Go using the OpenAI SDK.
- When you want structured responses, tool calling, and multi-turn behavior with in-session conversation history and minimal dependencies.

---

## Instructions

### Step 1 — Gather requirements

Ask only for the minimum needed to scaffold a useful first version. Do not block on optional details.

**Required:**
1. **Agent name** — snake_case (e.g. `loan_advisor`, `code_reviewer`)
2. **Domain** — one sentence describing what the agent specialises in
3. **Tools** — list the tools this agent needs. For each: name, description, inputs, return value. If the agent needs no tools, require the user to confirm that explicitly.

**Optional:**
4. **Model** — default: `gpt-4o-mini`
5. **Response schema** — what structured data should the agent return? Default: derive from domain.
6. **Persistent memory** — persist memory across sessions? Default: no.
7. **LLP observability** — include LLP connectivity? Default: no.

If the user does not specify an optional item, proceed with the default and note the assumption briefly. Do not proceed until the required tools input is provided. An explicit "no tools" answer satisfies that requirement and should produce an agent with no tool definitions or tool loop configuration.

---

### Step 2 — Core pattern

Every Go agent follows this pattern:

```
User message → buildPrompt() → LLM call → [tool loop, manual] → parseResponse() → format → reply
```

Go requires a **manual tool loop** — check the stop reason after each LLM call, execute any requested tools, feed results back, and repeat until the model produces a final text response.

#### A. Structured responses

Define a Go struct for the response. Use `json` tags with `omitempty` for optional fields. Validate required fields manually after unmarshalling.

```go
// Example only — design your own struct to fit the domain
type AgentResponse struct {
    Type            string   `json:"type"`
    Category        string   `json:"category,omitempty"`
    Recommendation  string   `json:"recommendation,omitempty"`
    Considerations  []string `json:"considerations,omitempty"`
    Reason          string   `json:"reason,omitempty"`
}

func validateResponse(r *AgentResponse) error {
    if r.Type == "" {
        return fmt.Errorf("missing required field: type")
    }
    return nil
}
```

Valid `type` values must include at minimum:
- A primary response type (e.g. `"analysis"`) with domain-specific fields
- `"capabilities"` — for "what can you do?" questions
- `"decline"` — for out-of-scope questions

#### B. System prompt

The system prompt must:
- Instruct the LLM to return ONLY valid JSON matching the struct
- Show the exact JSON format for every response type
- Define the agent's expertise areas and categories
- State behavioural rules (what the agent will and won't do)

#### C. Tools

Tools are functions the LLM requests by name. In Go, you execute them manually in a loop.

Design principles:
- One tool, one responsibility
- Return a plain string — the LLM reads tool output as text
- **Catch all errors inside `dispatchTool`** and return a graceful error string — never return an error to the LLM call loop

```go
func dispatchTool(name string, input json.RawMessage) string {
    switch name {
    case "tool_name":
        var args struct {
            Param string `json:"param"`
        }
        if err := json.Unmarshal(input, &args); err != nil {
            return fmt.Sprintf("Tool error: %v", err)
        }
        result, err := myTool(args.Param)
        if err != nil {
            return fmt.Sprintf("Tool error: %v", err)
        }
        return result
    default:
        return fmt.Sprintf("Unknown tool: %s", name)
    }
}
```

#### D. Agent struct

The `Agent` struct and `NewAgent()` constructor are SDK-specific — see Step 3 for the full definition.

#### E. Response parsing

```go
func extractJSON(text string) string {
    re := regexp.MustCompile("(?s)```(?:json)?\\s*(.*?)\\s*```")
    if m := re.FindStringSubmatch(text); m != nil {
        return strings.TrimSpace(m[1])
    }
    return strings.TrimSpace(text)
}

func parseResponse(text string) (*AgentResponse, error) {
    var resp AgentResponse
    if err := json.Unmarshal([]byte(extractJSON(text)), &resp); err != nil {
        return nil, err
    }
    return &resp, validateResponse(&resp)
}
```

#### F. Response formatters

```go
func formatCapabilities() string { /* bullet list of what the agent can help with */ }
func formatResult(r *AgentResponse) string { /* domain-specific fields as plain text */ }
func formatDecline(r *AgentResponse) string {
    if r.Reason != "" { return r.Reason }
    return "I can only help with <domain> questions."
}
```

#### G. Prompt builder

```go
func buildPrompt(message string) string {
    return message
}
```

#### H. Conversation history

Maintain conversation history per session keyed by a stable sender or session identifier.

**Key rule:** the system prompt is never stored in history — it is prepended fresh on every call. This keeps it authoritative and prevents it from being diluted as context grows.

See `processMessage` in Step 3 for the full implementation.

#### I. Entry point

Requires `github.com/joho/godotenv` in your import block.

```go
func main() {
    if err := godotenv.Load(); err != nil {
        log.Println("No .env file found, using environment variables")
    }
    agent := NewAgent()
    // wire up your I/O here (HTTP, stdin, etc.)
}
```

Env defaults:

| Variable | Default |
|----------|---------|
| `MODEL_NAME` | `gpt-4o-mini` |
| `OPENAI_API_KEY` | `""` |

---

### Step 3 — OpenAI SDK implementation

**Import:**
```go
import (
    "sync"
    "github.com/openai/openai-go"
    "github.com/openai/openai-go/option"
)
```

**Client init:**
```go
client := openai.NewClient(option.WithAPIKey(os.Getenv("OPENAI_API_KEY")))
```

**Add client to Agent struct:**
```go
type Agent struct {
    client *openai.Client
    model  string
}
```

**Tool definitions** — declare as a package-level variable so it's accessible from both `main()` and `handleMessage()`:
```go
var toolDefs = []openai.ChatCompletionToolParam{{
    Type: "function",
    Function: openai.FunctionDefinitionParam{
        Name:        "tool_name",
        Description: openai.String("What this tool does and when to use it."),
        Parameters: openai.FunctionParameters{
            "type": "object",
            "properties": map[string]any{
                "param": map[string]any{"type": "string", "description": "..."},
            },
            "required": []string{"param"},
        },
    },
}}
```

**LLM call with manual tool loop:**
```go
func (a *Agent) call(ctx context.Context, messages []openai.ChatCompletionMessageParamUnion, toolDefs []openai.ChatCompletionToolParam) (string, error) {
    for {
        params := openai.ChatCompletionNewParams{
            Model:    openai.F(a.model),
            Messages: openai.F(messages),
        }
        if len(toolDefs) > 0 {
            params.Tools = openai.F(toolDefs)
            params.ToolChoice = openai.F[openai.ChatCompletionToolChoiceOptionUnionParam](openai.ChatCompletionToolChoiceOptionAutoParam("auto"))
        }

        resp, err := a.client.Chat.Completions.New(ctx, params)
        if err != nil {
            return "", err
        }

        choice := resp.Choices[0]
        if choice.FinishReason != openai.ChatCompletionChoicesFinishReasonToolCalls {
            return choice.Message.Content, nil
        }

        // execute tool calls and continue
        messages = append(messages, choice.Message.ToParam())
        for _, tc := range choice.Message.ToolCalls {
            result := dispatchTool(tc.Function.Name, json.RawMessage(tc.Function.Arguments))
            messages = append(messages, openai.ToolMessage(tc.ID, result))
        }
    }
}
```

**Multi-turn:** Maintain a `[]openai.ChatCompletionMessageParamUnion` slice per session. Append raw user and assistant turns after each exchange.

**Build messages slice — system prompt prepended fresh, history in the middle:**
```go
func buildMessages(prompt string, history []openai.ChatCompletionMessageParamUnion) []openai.ChatCompletionMessageParamUnion {
    msgs := []openai.ChatCompletionMessageParamUnion{openai.SystemMessage(SYSTEM_PROMPT)}
    msgs = append(msgs, history...)
    msgs = append(msgs, openai.UserMessage(buildPrompt(prompt)))
    return msgs
}
```

**Process a message:**
```go
type sessionStore struct {
    mu   sync.RWMutex
    data map[string][]openai.ChatCompletionMessageParamUnion
}

func newSessionStore() *sessionStore {
    return &sessionStore{
        data: make(map[string][]openai.ChatCompletionMessageParamUnion),
    }
}

func (s *sessionStore) Get(sessionID string) []openai.ChatCompletionMessageParamUnion {
    s.mu.RLock()
    defer s.mu.RUnlock()

    history := s.data[sessionID]
    out := make([]openai.ChatCompletionMessageParamUnion, len(history))
    copy(out, history)
    return out
}

func (s *sessionStore) Set(sessionID string, history []openai.ChatCompletionMessageParamUnion) {
    s.mu.Lock()
    defer s.mu.Unlock()

    next := make([]openai.ChatCompletionMessageParamUnion, len(history))
    copy(next, history)
    s.data[sessionID] = next
}

var sessions = newSessionStore()

func processMessage(agent *Agent, sessionID, prompt string) string {
    history := sessions.Get(sessionID)

    rawText, err := agent.call(context.Background(), buildMessages(prompt, history), toolDefs)
    if err != nil {
        return "I'm sorry, I encountered an error processing your request."
    }

    history = append(history,
        openai.UserMessage(prompt),
        openai.AssistantMessage(rawText),
    )
    if len(history) > 40 {
        history = history[2:]
    }
    sessions.Set(sessionID, history)

    resp, err := parseResponse(rawText)
    if err != nil {
        return "I'm sorry, I encountered an error processing your request."
    }

    switch resp.Type {
    case "capabilities": return formatCapabilities()
    case "decline":      return formatDecline(resp)
    default:             return formatResult(resp)
    }
}
```

---

### Step 4 — Optional building blocks

#### Persistent memory across sessions

In-session conversation history above is what enables multi-turn behavior.

For persistent memory across sessions, store and retrieve keyed by a stable user or session identifier:
- **Simple:** serialise history to a JSON file on disk
- **Production:** Redis or a database, keyed by sender ID
- **Semantic:** embed past exchanges and retrieve relevant ones by similarity before each call

Always wrap memory retrieval in error handling — a failure must never block the agent from responding.

---

### Step 5 — Observability (optional)

Add LLP platform connectivity for managed routing, tool-call tracing, and observability after the core agent is working.

**Add dependency:** `github.com/llpsdk/llp-go`

**Implement `handleMessage(agent, msg)`:**

```
1. Build messages slice and call agent
2. Parse response
3. Switch on type → format → return string
4. On error: return safe fallback string
```

**Utility helper** — add this alongside the other helpers:
```go
func getEnv(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

`handleMessage` is the LLP equivalent of `processMessage` — it replaces it when using LLP, reusing the same session store keyed by `msg.Sender`:
```go
var sessions = newSessionStore()
```

```go
func handleMessage(agent *Agent, msg llp.TextMessage) string {
    history := sessions.Get(msg.Sender)

    rawText, err := agent.call(context.Background(), buildMessages(msg.Prompt, history), toolDefs)
    if err != nil {
        return "I'm sorry, I encountered an error processing your request."
    }

    history = append(history,
        openai.UserMessage(msg.Prompt),
        openai.AssistantMessage(rawText),
    )
    if len(history) > 40 {
        history = history[2:]
    }
    sessions.Set(msg.Sender, history)

    resp, err := parseResponse(rawText)
    if err != nil {
        return "I'm sorry, I encountered an error processing your request."
    }

    switch resp.Type {
    case "capabilities": return formatCapabilities()
    case "decline":      return formatDecline(resp)
    default:             return formatResult(resp)
    }
}
```

**Wire up LLP client — replaces the standalone `main()` from Step 2H.**

Add to your import block:
```go
"context"
"os/signal"
"sync"
"syscall"
llp "github.com/llpsdk/llp-go"
```

```go
func main() {
    if err := godotenv.Load(); err != nil {
        log.Println("No .env file found, using environment variables")
    }

    agent := NewAgent()

    ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    client, err := llp.NewClient(getEnv("LLP_AGENT_NAME", "<agent-name>"), getEnv("LLP_API_KEY", "")).
        OnMessage(func(ctx context.Context, msg llp.TextMessage) (llp.TextMessage, error) {
            return msg.Reply(handleMessage(agent, msg)), nil
        }).
        Connect(ctx)

    if err != nil {
        log.Fatalf("Fatal: %v", err)
    }

    log.Println("Agent connected")
    <-ctx.Done()
    client.Close()
}
```

Additional env vars when using LLP:

| Variable | Default |
|----------|---------|
| `LLP_AGENT_NAME` | `<agent-name>` |
| `LLP_API_KEY` | `""` |

---

### Step 6 — Generate the output

Generate a dedicated directory named `<agent-name>/` containing:

**`main.go`** — package `main`, sections in this order:
1. File header comment
2. `package main`
3. `import` block
4. Constants (`SYSTEM_PROMPT`)
5. `AgentResponse` struct + `validateResponse()`
6. Tool definitions + `dispatchTool()` (if any)
7. `Agent` struct + `NewAgent()` + `call()`
8. Formatters
9. `buildPrompt()`, `buildMessages()`, `extractJSON()`, `parseResponse()`
10. `sessionStore` + `processMessage()` (standalone) — or `getEnv()` + `handleMessage()` (if LLP observability requested)
11. `main()`

Separate sections with:
```go
// =============================================================================
// Section Name
// =============================================================================
```

Supporting files:
- `go.mod` — `module <agent-name>`, `go 1.21`, `github.com/openai/openai-go`, `github.com/joho/godotenv` (add `github.com/llpsdk/llp-go` if observability requested)
- `.env.example` — all env vars with inline comments
- `.gitignore` — binary output (e.g. `<agent-name>`) and `.env`
- `README.md` — what it does, prerequisites (`go 1.21+`), `go mod tidy`, config table, `go run main.go`

### Step 7 — Validate before calling it runnable

Do not describe the scaffold as complete or runnable until it passes basic local validation.

Minimum validation:
- Run `go mod tidy`
- Run `go build ./...`
- If either command fails, fix the generated code and rerun validation

In the final response:
- State that validation passed
- Mention any assumptions that were applied for omitted optional inputs
