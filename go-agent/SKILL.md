---
name: Go Agent
description: Scaffold a complete Go agent using the OpenAI SDK. Covers structured responses, tools, multi-turn conversation, memory, and optional LLP observability.
license: MIT
metadata:
    author: Large Language Platform, Inc.
    version: 0.1.0
---

# Go Agent

Scaffold a complete, runnable Go agent using the OpenAI SDK.

## When to use

- When building an agent in Go using the OpenAI SDK.
- When you want structured responses, tool calling, and optional conversation history with minimal dependencies.

---

## Instructions

### Step 1 — Gather requirements

Ask the user for the following. Do not proceed until all required items are answered.

**Required:**
1. **Agent name** — snake_case (e.g. `loan_advisor`, `code_reviewer`)
2. **Domain** — one sentence describing what the agent specialises in
3. **Model** — default: `gpt-4o-mini`
4. **Tools** — list the tools this agent needs. For each: name, description, inputs, return value. If the agent needs no tools, confirm this explicitly.

**Optional:**
5. **Response schema** — what structured data should the agent return? Default: derive from domain.
6. **Multi-turn** — maintain conversation history within a session? Default: no.
7. **Memory** — persist memory across sessions? Default: no.

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

```go
type Agent struct {
    model  string
    // SDK client — see Step 3
}

func NewAgent() *Agent {
    model := os.Getenv("MODEL_NAME")
    if model == "" {
        model = "gpt-4o-mini"
    }
    return &Agent{model: model}
}
```

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
    return message + "\n\nReturn ONLY valid JSON matching the required schema."
}
```

#### H. Entry point

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
    "github.com/openai/openai-go"
    "github.com/openai/openai-go/option"
    "github.com/joho/godotenv"
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

**Tool definitions:**
```go
toolDefs := []openai.ChatCompletionToolParam{{
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

**Multi-turn:** Maintain a `[]openai.ChatCompletionMessageParamUnion` slice per session. Append user and assistant turns after each exchange.

**Build initial messages slice from a prompt string:**
```go
func buildMessages(prompt string) []openai.ChatCompletionMessageParamUnion {
    return []openai.ChatCompletionMessageParamUnion{
        openai.UserMessage(buildPrompt(prompt)),
    }
}
```

---

### Step 4 — Optional building blocks

#### Multi-turn conversation

Maintain conversation history scoped to the session, keyed by a stable session or sender identifier. Append user prompt and assistant response after each successful LLM call. Cap history to avoid token bloat (e.g. drop oldest pair at 20 turns).

#### Memory across sessions

For in-session memory only, conversation history above is sufficient.

For persistent memory across sessions, store and retrieve keyed by a stable user or session identifier:
- **Simple:** serialise history to a JSON file on disk
- **Production:** Redis or a database, keyed by sender ID
- **Semantic:** embed past exchanges and retrieve relevant ones by similarity before each call

Always wrap memory retrieval in error handling — a failure must never block the agent from responding.

---

### Step 5 — Observability (optional)

Add LLP platform connectivity for managed routing, tool-call tracing, and observability after the core agent is working.

**Add dependency:** `github.com/llpsdk/llp-go`

**Implement `handleMessage(agent, message, annotater)`:**

```
1. Derive corrId from message (e.g. message.ID[:8])
2. Log: [corrId] >>> RECV from=<sender> prompt="<first 80 chars>"
3. Build messages slice and call agent
4. Parse response
5. Log: [corrId] === type=<type>
6. Switch on type → format → return string
7. On error: log [corrId] !!! → return safe fallback string
```

**Helper functions required by the observability layer:**
```go
func getEnv(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}

func truncate(s string, n int) string {
    if len(s) <= n {
        return s
    }
    return s[:n]
}
```

```go
func handleMessage(agent *Agent, msg llp.TextMessage, annotater llp.Annotater) string {
    corrId := msg.ID
    if len(corrId) > 8 { corrId = corrId[:8] }
    log.Printf("[%s] >>> RECV from=%s prompt=%q", corrId, msg.Sender, truncate(msg.Prompt, 80))

    rawText, err := agent.call(context.Background(), buildMessages(msg.Prompt), toolDefs)
    if err != nil {
        log.Printf("[%s] !!! error=%v", corrId, err)
        return "I'm sorry, I encountered an error processing your request."
    }

    resp, err := parseResponse(rawText)
    if err != nil {
        log.Printf("[%s] !!! parse error=%v", corrId, err)
        return "I'm sorry, I encountered an error processing your request."
    }

    log.Printf("[%s] === type=%s", corrId, resp.Type)
    switch resp.Type {
    case "capabilities": return formatCapabilities()
    case "decline":      return formatDecline(resp)
    default:             return formatResult(resp)
    }
}
```

**Wire up LLP client in `main()`:**

```go
import llp "github.com/llpsdk/llp-go"

client := llp.NewClient(
    getEnv("AGENT_NAME", "<agent-name>"),
    getEnv("AGENT_KEY", ""),
    llp.Config{URL: getEnv("PLATFORM_ADDRESS", "ws://localhost:4000/agent/websocket")},
)

client.OnStart(func() *Agent { return NewAgent() })
client.OnMessage(func(agent *Agent, msg llp.TextMessage, annotater llp.Annotater) llp.Reply {
    return msg.Reply(handleMessage(agent, msg, annotater))
})
client.OnStop(func() { log.Println("session ended") })

stop := make(chan os.Signal, 1)
signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
go func() { <-stop; client.Close(); os.Exit(0) }()

if err := client.Connect(); err != nil {
    log.Fatalf("Fatal error: %v", err)
}
select {}
```

Additional env vars when using LLP:

| Variable | Default |
|----------|---------|
| `AGENT_NAME` | `<agent-name>` |
| `AGENT_KEY` | `""` |
| `PLATFORM_ADDRESS` | `ws://localhost:4000/agent/websocket` |

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
10. `getEnv()`, `truncate()` (if LLP observability requested)
11. `handleMessage()` (if LLP observability requested)
12. `main()`

Separate sections with:
```go
// =============================================================================
// Section Name
// =============================================================================
```

Supporting files:
- `go.mod` — `module <agent-name>`, `go 1.21`, `github.com/openai/openai-go`, `github.com/joho/godotenv`
- `.env.example` — all env vars with inline comments
- `.gitignore` — binary output (e.g. `<agent-name>`) and `.env`
- `README.md` — what it does, prerequisites (`go 1.21+`), `go mod tidy`, config table, `go run main.go`
