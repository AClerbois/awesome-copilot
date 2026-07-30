# NuGet packages and target frameworks

## The official packages

All packages live under the [`ModelContextProtocol` NuGet profile](https://www.nuget.org/profiles/ModelContextProtocol). The official C# SDK repo is [`modelcontextprotocol/csharp-sdk`](https://github.com/modelcontextprotocol/csharp-sdk), maintained jointly by the MCP project and Microsoft.

| Package | When to use it | Brings in |
|---|---|---|
| **`ModelContextProtocol`** | Default for STDIO servers and most projects | `Core` + `Microsoft.Extensions.Hosting` integration, attribute discovery (`AddMcpServer`, `WithToolsFromAssembly`, etc.) |
| **`ModelContextProtocol.AspNetCore`** | HTTP (Streamable) servers hosted in ASP.NET Core | The above + `WithHttpTransport` and `MapMcp` |
| **`ModelContextProtocol.Core`** | Pure clients, custom hosts, low-level scenarios where you don't want the `Microsoft.Extensions.*` dependencies | Just the protocol + transports + low-level `McpServer.Create` / `McpClient.CreateAsync` |
| **`ModelContextProtocol.Extensions.Apps`** | Tools that render an interactive UI in the host (MCP Apps) | `[McpAppUi]` attribute, `WithMcpApps()`, typed `_meta.ui` types |
| **`ModelContextProtocol.Extensions.Tasks`** | Long-running tool invocations with status polling (MCP Tasks, SEP-2663) | `WithTasks(...)`, `IMcpTaskStore` / `InMemoryMcpTaskStore`, `TasksProtocol` |

**Rule of thumb:**
- New STDIO server → `ModelContextProtocol` + `Microsoft.Extensions.Hosting`.
- New HTTP server → `ModelContextProtocol.AspNetCore` only (it transitively pulls in everything you need).
- Pure client app → `ModelContextProtocol.Core` (or `ModelContextProtocol` if you also want hosting/DI for the client).
- Add the `Extensions.*` packages only when you actually use MCP Apps or Tasks — they version in lockstep with the core packages.

## Versions

The stable line is **2.x** — `2.0.0` shipped 2026-07-28 alongside the MCP **2026-07-28 spec** it implements. The last 1.x release was `1.4.1`. The `0.x` line was preview and has breaking differences — if you find docs or blog posts referencing `0.4`/`0.6`, treat them as ancient; treat 1.x-era material as pre-2.0 (several defaults flipped, see below).

The SDK negotiates down automatically: a 2.0 server/client interoperates with peers on 2025-11-25 and earlier protocol versions.

To check the latest:

```bash
dotnet search ModelContextProtocol --prerelease
```

## What changed in 2.0 (migration from 1.x)

The compiler tells you most of it — 2.0 stages deprecations behind warning codes:

| Warning | Meaning | What to do |
|---|---|---|
| `MCP9005` | Roots, sampling, or MCP-channel logging API — deprecated by spec 2026-07-28 | Still works against down-level peers; suppress while planning migration, avoid in new designs |
| `MCP9006` | `Stateless = false` (stateful HTTP) on new protocol versions | Keep only if you need server-to-client features; otherwise delete the line — stateless is the default now |
| `MCP9007` | `AuthorizationRedirectDelegate` in OAuth options | Migrate to `ClientOAuthOptions.AuthorizationCallbackHandler` (returns code, state and issuer for RFC 9207 validation) |

Behavioral breaks to check when upgrading:
- **HTTP is stateless by default.** `HttpServerTransportOptions.Stateless` flipped from `false` to `true`. Stateless servers no longer create sessions or expose the standalone SSE GET/DELETE endpoints. See `transport-http.md`.
- **Discovery-first negotiation.** Spec 2026-07-28 removes the `initialize` handshake (SEP-2575); clients probe `server/discover` first and fall back to legacy `initialize` automatically. No code change needed — but don't hand-write `initialize` assumptions into tests or proxies.
- **Structured tool results emit raw values.** With an output schema and a non-object return type, the wire now carries `structuredContent: 72` instead of `{ "result": 72 }`. See `tool-primitive.md`.
- **`Tool.InputSchema` is required.** Deserializing a `Tool` without `inputSchema` throws `JsonException` — hand-built tool payloads and test fixtures need at least an empty `{}` object schema.
- **Tasks moved out of core.** The 1.4.x experimental tasks implementation was replaced (no API or wire compat) by `ModelContextProtocol.Extensions.Tasks`; `RequestMethods.Tasks*` constants became `TasksProtocol` members. See `tasks.md`.
- **OAuth hardening.** Issuer mismatches rejected per RFC 9207/8414, PKCE `S256` must be advertised by the authorization server, dynamic client registration now sends `application_type`, repeated `insufficient_scope` challenges that add no scopes throw `McpException`.

## Target frameworks

The SDK targets **`.NET 8.0`** and **`netstandard2.0`**. That means it runs on:
- .NET 8 (LTS)
- .NET 9
- .NET 10 (current LTS — recommended for new projects)
- .NET Framework 4.6.2+ via netstandard2.0 (rare; only for legacy hosts)

For HTTP servers you specifically need a TFM that supports ASP.NET Core (so .NET 8/9/10).

## Project setup commands

### STDIO server

```bash
dotnet new console -n MyMcpServer -f net10.0
cd MyMcpServer
dotnet add package ModelContextProtocol
dotnet add package Microsoft.Extensions.Hosting
```

### HTTP (Streamable) server

```bash
dotnet new web -n MyMcpServer -f net10.0
cd MyMcpServer
dotnet add package ModelContextProtocol.AspNetCore
```

(`dotnet new web` gives you a minimal ASP.NET Core project — exactly what `MapMcp` needs.)

### Client

```bash
dotnet new console -n MyMcpClient -f net10.0
cd MyMcpClient
dotnet add package ModelContextProtocol.Core
```

## Optional but commonly useful

| Package | Why |
|---|---|
| `Microsoft.Extensions.AI` | Provides `IChatClient`, `ChatMessage`, `ChatRole`, `ChatOptions` — the abstractions used by `AsSamplingChatClient()` and by prompt return types. |
| `Microsoft.Extensions.AI.Abstractions` | Pulled in transitively but worth knowing about for types like `DataContent`, `TextContent`. |
| `OpenTelemetry.Extensions.Hosting` | The SDK emits OTel traces and metrics for tool calls — wire them up if the user has an observability story. |

## What about `dnx`?

Newer Microsoft examples sometimes show launching servers via `dnx PackageName --version 2.0.0`. That's a valid distribution model: publish your server as a NuGet package and let users run it without cloning. It's orthogonal to how the server itself is built — keep your code identical and just change the launch command.
