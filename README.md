# Burp MCP Core

Reusable local MCP transport for Burp extensions. The module owns HTTP framing,
loopback CORS policy, JSON-RPC validation, protocol negotiation, tool discovery,
and tool invocation. A plugin only supplies a `McpToolProvider`.

```java
McpToolProvider provider = new McpToolProvider() {
    public List<McpTool> listTools() {
        return List.of(new McpTool("plugin.status", "Read status",
                Map.of("type", "object")));
    }

    public Object callTool(String name, Map<String, Object> arguments) {
        return Map.of("status", "ok");
    }
};

McpServer server = new McpServer(provider, "my-plugin-mcp", "1.0.0");
server.start();
// server.getEndpoint() -> http://127.0.0.1:<port>/mcp
```

The server is intentionally bound to loopback and has no authentication layer;
extensions should not expose the endpoint outside the local machine.

An `initialize` response includes an `Mcp-Session-Id` and the negotiated
`MCP-Protocol-Version`. Session-aware clients should send both headers on later
requests, first acknowledge `notifications/initialized`, and use `DELETE /mcp`
to close the session. Sessions expire after 30 minutes of inactivity and the
server keeps at most 128 sessions. POST requests without a session header remain
available for backwards-compatible stateless clients.

Before a provider is invoked, `tools/call` checks the tool's `inputSchema` for
required properties, basic JSON types, enum values, and array item types. Provider
code can still perform stricter, operation-specific validation.

## Standalone package

Build the executable package from the repository root:

```powershell
.\mvnw.cmd package
```

The output is `target/mcp-core-1.0.0-standalone.jar`. Run it with Java 21:

```powershell
java -jar target/mcp-core-1.0.0-standalone.jar
```

The launcher binds to `127.0.0.1`, tries ports `8765` through `8785`, and exposes
two diagnostic tools (`standalone.status` and `standalone.echo`). Use
`--port`, `--name`, `--version`, or `--provider-class` to customize it. A custom
provider class must implement `McpToolProvider` and have a public no-argument
constructor. To load one, put its jar beside the standalone package and launch
with an explicit class path, for example on Windows:

```powershell
java -cp "custom-provider.jar;target/mcp-core-1.0.0-standalone.jar" `
  burp.vaycore.mcp.McpStandaloneMain --provider-class com.example.MyProvider
```

On Linux/macOS, use `:` instead of `;` in the class path.

OpenAPI 3.1 documentation is provided in `openapi.yaml`. It can be opened in
Swagger Editor or Swagger UI, with the server URL changed to the actual port
shown by the launcher.
