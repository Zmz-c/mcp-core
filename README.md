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

McpServer server = new McpServer(provider, "my-plugin-mcp", "1.1.2");
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

The output is `target/mcp-core-1.1.2-standalone.jar`. Run it with Java 21:

```powershell
java -jar target/mcp-core-1.1.2-standalone.jar
```

The launcher uses the shared `127.0.0.1:8765` host port and exposes two
diagnostic tools (`standalone.status` and `standalone.echo`). If another host
already owns the port, it registers this provider and joins that host. Use
`--port`, `--name`, `--version`, or `--provider-class` to customize it. A custom
provider class must implement `McpToolProvider` and have a public no-argument
constructor. To load one, put its jar beside the standalone package and launch
with an explicit class path, for example on Windows:

```powershell
java -cp "custom-provider.jar;target/mcp-core-1.1.2-standalone.jar" `
  burp.zm.mcp.McpStandaloneMain --provider-class com.example.MyProvider
```

On Linux/macOS, use `:` instead of `;` in the class path.

## Offline Swagger UI

The public Java API package is `burp.zm.mcp` (the previous `burp.vaycore.mcp`
package is no longer used). The standalone server bundles Swagger UI, its CSS/JavaScript assets, and the
OpenAPI document. No CDN, network connection, or external browser extension is
needed. After starting the server, open the following local URL in a browser:

```text
http://127.0.0.1:8765/docs/
```

The page loads the specification from `/docs/openapi.yaml` and lets you try
the health and MCP endpoints against the running local server. If `--port` is
used for an isolated instance, replace `8765` with that explicit port. The raw OpenAPI
3.1 document is also available as `openapi.yaml` in the repository.

## Shared single-port host

Every default `McpServer` uses the fixed loopback port `8765`. The first plugin
that binds it becomes the host. If another plugin starts later, it registers
its `McpToolProvider` through the host's loopback registration endpoint and
opens a heartbeat-backed provider channel. The host aggregates all registered
tools in `tools/list` and forwards `tools/call` over that channel.

This makes the MCP client configuration stable:

```text
http://127.0.0.1:8765/mcp
```

Provider names and tool names must be unique; use a namespace such as
`my-plugin.status` to avoid collisions. Providers are removed on clean stop or
after a 45-second lease expires. If the host disappears, a connected provider
automatically attempts to claim `8765` and become the new host, then accepts
new registrations. The internal registration endpoints are
`/__mcp/providers/register` and `/__mcp/providers/unregister`; they are
loopback-only and are documented in the bundled OpenAPI specification.
