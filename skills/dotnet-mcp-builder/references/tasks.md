# Tasks (long-running tool invocations)

The MCP Tasks extension (SEP-2663, stabilised with spec 2026-07-28) lets a tool call run **asynchronously**: instead of holding the request open for minutes, the server returns a task handle and the client polls its status (and can answer input requests) until the result is ready. Typical use cases: exports, batch jobs, builds, anything that outlives a sane HTTP timeout.

> **New in SDK 2.0.** Tasks ship as the dedicated package **`ModelContextProtocol.Extensions.Tasks`** (2.0.0). The experimental tasks support in SDK 1.4.x was replaced wholesale — **no API or wire compatibility**. If you see `RequestMethods.Tasks*` constants in old code, those are gone; the protocol constants now live on `TasksProtocol`.

## Setup

```bash
dotnet add package ModelContextProtocol.Extensions.Tasks
```

```csharp
using ModelContextProtocol.Server;

builder.Services
    .AddMcpServer()
    .WithHttpTransport()
    .WithToolsFromAssembly()
    .WithTasks(new InMemoryMcpTaskStore());
```

`WithTasks(IMcpTaskStore store)` enables the extension and advertises it. With no further configuration, all tools are treated as task-capable: when a client that supports tasks (and negotiates spec 2026-07-28 or later) opts in on a call, the server runs the tool in the background and the client polls; older or non-opting clients get normal synchronous execution.

## Choosing execution mode per tool

Use the `WithTasks` overload with `McpTasksOptions` to control which tools run as tasks. The selector receives the incoming call and returns an `McpTaskExecutionMode`:

```csharp
.WithTasks(new InMemoryMcpTaskStore(), options =>
{
    options.ExecutionModeSelector = ctx => ctx.Params?.Name switch
    {
        "export_report" => McpTaskExecutionMode.Required,   // must run as a task; throws if the client can't
        "quick_lookup"  => McpTaskExecutionMode.Synchronous, // never a task
        _               => McpTaskExecutionMode.Optional,    // task if the client opted in, sync otherwise
    };
})
```

| Mode | Behaviour |
|---|---|
| `Synchronous` | Executes immediately, no task created. |
| `Optional` | Creates a task only when the client advertises task support and opts in on the request. Safe default. |
| `Required` | Demands task capability from the client; the call fails otherwise. Use for work that genuinely can't run inline. |

The selector is a `Func<RequestContext<CallToolRequestParams>, McpTaskExecutionMode>` — you can also inspect `ctx.MatchedPrimitive` to decide from tool metadata instead of hardcoding names.

## Task storage

`IMcpTaskStore` abstracts where task state (status, results, pending input requests) lives:

- **`InMemoryMcpTaskStore`** — the built-in default. Fine for STDIO and single-instance servers. Task state dies with the process.
- **Custom `IMcpTaskStore`** — implement it over Redis/a database for horizontally-scaled HTTP deployments, where the poll request may land on a different instance than the one running the task.

That second point is the whole reason tasks pair well with **stateless HTTP** (the 2.0 default): the client polls with plain requests, no session affinity needed — provided the store is shared.

## Interaction with other features

- **Progress notifications** still work for tools that stay synchronous — see `server-features.md`. Tasks are the better fit once durations get long enough that clients would time out or want to disconnect and come back.
- **Elicitation from a task:** the extension supports input requests during task execution (the client sees them while polling). Check the [API reference](https://csharp.sdk.modelcontextprotocol.io/) for the current shape (`InputResponseReceivedEventArgs` et al.) rather than guessing.
- **Down-level clients** that don't know the extension simply call tools synchronously — design `Optional`-mode tools so an inline run, while slow, still completes.

## Pitfalls

- **Reusing 1.4.x tasks code.** It won't compile and wouldn't be wire-compatible anyway. Port to the extension package.
- **In-memory store on a scaled deployment.** Polls that land on another instance return "unknown task". Share the store.
- **`Required` mode everywhere.** It breaks every client that hasn't adopted the extension yet. Prefer `Optional` unless inline execution is genuinely impossible.
