# MCP Apps (interactive UI)

[MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview) is the official extension that lets a tool return an **interactive UI** rendered in a sandboxed iframe inside the host (Claude, Claude Desktop, VS Code Copilot, Goose, Postman, MCPJam). Typical use cases: charts, dashboards, multi-step forms, 3D viewers, real-time monitors, PDF/video viewers.

> **New in SDK 2.0:** the typed convenience layer finally shipped as the **`ModelContextProtocol.Extensions.Apps`** package (2.0.0). Prefer it: `[McpAppUi]` on the tool + `.WithMcpApps()` on the builder replaces the manual `_meta` plumbing. The manual pattern (further down) remains valid and is still useful for dynamic tools or hosts needing legacy `_meta` keys.

## The typed way (SDK 2.0+, recommended)

```bash
dotnet add package ModelContextProtocol.Extensions.Apps
```

1. Serve the `ui://` resource exactly as before (Step 1 below).
2. Annotate the tool and enable the extension:

```csharp
using ModelContextProtocol.Server;

[McpServerToolType]
public class ChartTools
{
    [McpServerTool(Name = "visualize_data")]
    [McpAppUi(ResourceUri = "ui://charts/interactive")]
    [Description("Visualize the user's data as an interactive chart.")]
    public static async Task<ChartData> VisualizeData(string datasetId, CancellationToken ct)
        => await LoadDataset(datasetId, ct);
}
```

```csharp
builder.Services
    .AddMcpServer()
    .WithHttpTransport()
    .WithToolsFromAssembly()
    .WithMcpApps();          // AFTER tool registration — it post-processes registered tools
```

`WithMcpApps()` advertises the MCP Apps capability and stamps `_meta.ui` onto every tool carrying `[McpAppUi]`. Notes:
- Order matters: call it after `WithTools*`. It skips tools that already have an explicit `Meta["ui"]` entry, so you can mix attribute-driven and manual tools.
- `[McpAppUi]` also has a `Visibility` property (`McpUiToolVisibility.Model` / `McpUiToolVisibility.App`) to control whether the LLM, the rendered app, or both may invoke the tool. Null/empty means both.
- CSP and permissions for the iframe have typed counterparts (`McpUiResourceCsp`, `McpUiResourcePermissions`, `McpUiResourceMeta`) — check the [API reference](https://csharp.sdk.modelcontextprotocol.io/) for the current shape rather than guessing.

The rest of this page shows the underlying wire pattern — read it to understand what the extension emits, or to target hosts that predate it.

## How it works (short version)

1. You register a **resource** at a `ui://` URI returning an HTML bundle.
2. You register a **tool** whose definition includes `_meta.ui.resourceUri` pointing to that URI.
3. When the LLM calls the tool, the host fetches the UI resource and renders it in a sandboxed iframe in the chat.
4. The HTML talks to the host over `postMessage` JSON-RPC (use `@modelcontextprotocol/ext-apps` from the bundle, or hand-roll it).
5. The app can call back into your MCP server (any tool), update the model context, etc.

The full protocol spec is at [`@modelcontextprotocol/ext-apps`](https://github.com/modelcontextprotocol/ext-apps).

## Step 1: Serve the UI resource

Bundle your HTML/JS/CSS into a single string (or load from `wwwroot`). Serve it at a `ui://` URI.

```csharp
using System.ComponentModel;
using System.IO;
using System.Reflection;
using ModelContextProtocol.Protocol;
using ModelContextProtocol.Server;

[McpServerResourceType]
public static class ChartUiResource
{
    [McpServerResource(
        UriTemplate = "ui://charts/interactive",
        Name = "Interactive chart",
        MimeType = "text/html+skybridge")]   // see "MIME type" note below
    [Description("UI bundle for the interactive chart MCP App.")]
    public static TextResourceContents GetUi()
    {
        // Load a bundled HTML/JS file from embedded resources or wwwroot.
        var html = LoadEmbeddedString("MyMcpServer.AppUi.chart.html");

        return new TextResourceContents
        {
            Uri = "ui://charts/interactive",
            MimeType = "text/html+skybridge",
            Text = html
        };
    }

    private static string LoadEmbeddedString(string resourceName)
    {
        var asm = Assembly.GetExecutingAssembly();
        using var stream = asm.GetManifestResourceStream(resourceName)
            ?? throw new InvalidOperationException($"Missing embedded resource {resourceName}");
        using var reader = new StreamReader(stream);
        return reader.ReadToEnd();
    }
}
```

**MIME type note:** the spec uses `text/html+skybridge` for app HTML so hosts can distinguish UI bundles from regular `text/html` previews. Use that, even though plain `text/html` may work today on lenient hosts.

## Step 2 (manual alternative): Emit `_meta` on the tool yourself

If you don't use the `Extensions.Apps` package — or the tool is built dynamically — set `_meta` via the lower-level `Tool` definition. Do this once at startup:

```csharp
using ModelContextProtocol.Protocol;
using ModelContextProtocol.Server;
using System.Text.Json;
using System.Text.Json.Nodes;

builder.Services.Configure<McpServerOptions>(options =>
{
    options.Capabilities ??= new();
    options.Capabilities.Tools ??= new();

    // Define the tool manually so we can attach _meta.
    var visualizeTool = new Tool
    {
        Name = "visualize_data",
        Description = "Visualize the user's data as an interactive chart.",
        InputSchema = JsonDocument.Parse("""
            {
              "type": "object",
              "properties": {
                "datasetId": { "type": "string", "description": "Dataset to visualize." }
              },
              "required": ["datasetId"]
            }
            """).RootElement,
        Meta = new JsonObject
        {
            ["ui"] = new JsonObject
            {
                ["resourceUri"] = "ui://charts/interactive"
                // Optionally:
                // ["csp"] = new JsonObject { ["default-src"] = "'self' https://cdn.example.com" },
                // ["permissions"] = new JsonArray("clipboard-write")
            }
        }
    };

    // Implement the call handler that returns the data the UI will render.
    options.Capabilities.Tools.ToolCollection ??= new();
    options.Capabilities.Tools.ToolCollection.Add(McpServerTool.Create(
        async (CallToolRequestParams req, CancellationToken ct) =>
        {
            var args = req.Arguments ?? new();
            var datasetId = args["datasetId"]!.GetValue<string>();
            var data = await LoadDataset(datasetId, ct);
            return new CallToolResult
            {
                Content = [new TextContentBlock { Text = JsonSerializer.Serialize(data) }],
                StructuredContent = JsonSerializer.SerializeToNode(data)
            };
        },
        visualizeTool));
});
```

If you don't need full structured content, the tool can return *just* JSON in a text block — the UI fetches it via `app.callServerTool(...)` after rendering.

### Backwards compatibility key

Some older hosts expect `_meta["ui/resourceUri"]` instead of `_meta.ui.resourceUri`. Set both for safety:

```csharp
Meta = new JsonObject
{
    ["ui"] = new JsonObject { ["resourceUri"] = "ui://charts/interactive" },
    ["ui/resourceUri"] = "ui://charts/interactive"   // legacy
}
```

## Step 3: The HTML bundle

A minimum viable bundle: vanilla JS using `@modelcontextprotocol/ext-apps`. The simplest build is a single self-contained HTML file.

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Chart</title>
    <style>body { font-family: system-ui; margin: 0; }</style>
  </head>
  <body>
    <div id="root">Loading…</div>
    <script type="module">
      import { App } from "https://esm.sh/@modelcontextprotocol/ext-apps@1";

      const app = new App();
      await app.connect();

      // Fetch the data we need from the server.
      const resp = await app.callServerTool({
        name: "visualize_data",
        arguments: { datasetId: "default" }
      });

      const data = JSON.parse(resp.content[0].text);
      document.getElementById("root").textContent =
        `Loaded ${data.points.length} data points.`;

      // Tell the model what just happened (becomes part of its context).
      await app.updateModelContext({
        content: [{ type: "text", text: "User opened the chart UI." }]
      });
    </script>
  </body>
</html>
```

**Tip:** for non-trivial UIs, build with Vite (React/Vue/Svelte/Solid — any of the [official starter templates](https://github.com/modelcontextprotocol/ext-apps/tree/main/examples)) and have the build emit a single inlined HTML you embed as a project resource.

## Project layout

A pragmatic layout for an MCP App in .NET:

```
MyMcpServer/
├── Program.cs
├── Tools/
│   └── VisualizeDataTool.cs       # (or registered via Configure as above)
├── Resources/
│   └── ChartUiResource.cs         # serves the ui:// resource
├── AppUi/
│   ├── chart.html                 # bundled UI (Embedded Resource)
│   └── package.json + src/...     # if you build with Vite, output to chart.html
└── MyMcpServer.csproj
```

In the csproj:

```xml
<ItemGroup>
  <EmbeddedResource Include="AppUi\chart.html" />
</ItemGroup>
```

Read it via `Assembly.GetManifestResourceStream("MyMcpServer.AppUi.chart.html")`.

## Testing locally

1. Run your MCP server (STDIO or HTTP).
2. Use a host that supports MCP Apps — Claude Desktop or VS Code Copilot Chat are the easiest.
3. Trigger the tool via the LLM. The UI renders inline.

For pure-UI iteration, [MCP Inspector](https://github.com/modelcontextprotocol/inspector) shows resource contents but does not fully render apps; for that, point Claude Desktop at your dev server.

## Pitfalls

- **Wrong MIME type.** Use `text/html+skybridge`. Plain `text/html` may still work but isn't future-proof.
- **Calling `WithMcpApps()` before tool registration.** It post-processes already-registered tools; called too early, no `_meta.ui` gets stamped and the UI silently never appears.
- **CSP too tight or too loose.** If your UI loads from a CDN, declare it (typed via the CSP types in `Extensions.Apps`, or manually in `Meta["ui"]["csp"]` — serialises to `_meta.ui.csp` on the wire). Otherwise the iframe sandbox blocks it.
- **No `ui` metadata on the tool.** Without `[McpAppUi]` + `WithMcpApps()` (or a manual `Meta["ui"]` entry), the host treats your tool as a regular text-returning tool. The UI never appears.
- **Trying to use browser APIs outside the sandbox.** No cookies, no localStorage from the parent. Use `app.updateModelContext` and tool calls for state.
