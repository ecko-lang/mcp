# mcp

The Model Context Protocol, both directions: call an MCP server's tools,
resources and prompts over stdio or HTTP, or be an MCP server. Needs `exec` to
start a stdio server and `net` for HTTP; grant them when you import the
package. A stdio server and the in-memory `fake` transport need nothing.

## Install

```bash
ecko get github.com/ecko-lang/mcp
```

`ecko get` writes the dependency into your `ecko.json`; add the grant for the
transports you use there:

```json
{
  "dependencies": {
    "mcp": {
      "path": "github.com/ecko-lang/mcp",
      "version": "0.56.0",
      "grant": ["exec", "net"]
    }
  }
}
```

```ecko
import mcp
```

## Usage

A server is a list of tools backed by Ecko functions; a client connects to it
through a transport. Here the transport is `fake`, which calls the server in
memory, so this runs offline:

```ecko
import mcp

tools = [
    mcp.tool(
        "word_count",
        "Count the words in some text",
        { type: "object", properties: { text: { type: "string" } }, required: ["text"] },
        fn(args) len(filter(split(args.text, " "), fn(w) w != "")),
    ),
]

session = mcp.connect(mcp.fake(fn(req) mcp.handle(req, tools, name: "words")))
print(session.server.serverInfo.name)                    # words
print(map(mcp.tools(session), fn(t) t.name))             # [word_count]

result = mcp.call_tool(session, "word_count", { text: "a b  c" })
print(mcp.text_of(result))                               # 3
print(mcp.is_error(result))                              # false

missing = try { mcp.call_tool(session, "nope") } catch (e) { get(e, "code") }
print(missing)                                           # -32602
mcp.close(session)
```

### Calling a real server

Most MCP servers are programs the client starts and talks to over their pipes:

```ecko norun
import mcp

session = mcp.connect(mcp.stdio_transport("npx", ["-y", "@modelcontextprotocol/server-everything"]))
for t in mcp.tools(session) {
    print("{t.name}: {t.description}")
}
print(mcp.text_of(mcp.call_tool(session, "echo", { message: "hi" })))

# Hand every tool to a model; it picks one and Ecko calls it over MCP.
answer = ai "echo the word ecko" using mcp.as_tools(session)
mcp.close(session)
```

A hosted server is a URL. The transport sends the protocol-version header and
keeps the `Mcp-Session-Id` the server hands out:

```ecko norun
import std.os
import mcp

token = secret(os.env("MCP_TOKEN"))
session = mcp.connect(
    mcp.http_transport(
        "https://mcp.example.com/",
        headers: { Authorization: "Bearer {reveal(token)}" },
    ),
)
print(mcp.server_capabilities(session))
mcp.close(session)
```

### Being a server

Save this as `words.ecko` and register `ecko words.ecko` with an MCP client:

```ecko norun
import mcp

mcp.serve_stdio(
    [
        mcp.tool(
            "word_count",
            "Count the words in some text",
            { type: "object", properties: { text: { type: "string" } }, required: ["text"] },
            fn(args) len(filter(split(args.text, " "), fn(w) w != "")),
        ),
    ],
    name: "words",
    version: "1.0",
)
```

`serve_http(port, tools)` serves the same tools over HTTP on `127.0.0.1`.

## API

### Client

| function | returns |
|---|---|
| `stdio_transport(command, args = [], timeout_ms = 30000)` | a transport that starts `command` and speaks over its pipes |
| `http_transport(url, headers = {})` | a Streamable HTTP transport |
| `fake(handler)` | a transport that hands each decoded request to `handler` in memory |
| `connect(transport, name = "ecko", version = "0.1.0")` | a session, after the `initialize` handshake |
| `close(session)` | `null`; stops the server process or ends the HTTP session |
| `server_version(session)` | the protocol version the server answered with, or `null` |
| `server_capabilities(session)` | the server's capabilities map (`{}` when it sent none) |
| `tools(session)` | every tool, `[{ name, description, inputSchema }]`, across all pages |
| `call_tool(session, name, args = {})` | the raw result: `content`, maybe `isError` |
| `text_of(result)` | the text parts of a result, joined |
| `is_error(result)` | `true` when the tool ran and reported failure |
| `as_tools(session)` | every tool as a spec for `ai ... using` |
| `as_tool(session, t)` | one tool as a spec for `ai ... using` |
| `resources(session)` | every resource, `[{ uri, name, mimeType }]`, across all pages |
| `read_resource(session, uri)` | the resource's contents, `[{ uri, mimeType, text }]` |
| `prompts(session)` | every prompt template, across all pages |
| `get_prompt(session, name, args = {})` | the rendered `messages` list |
| `call(session, method, params = {})` | the result of any request |
| `notify(session, method, params = {})` | `null`; sends a notification |
| `cancel(session, request_id, reason = "")` | `null`; sends `notifications/cancelled` |
| `next_request_id(session)` | the Int id the next request will use |

### Server

| function | returns |
|---|---|
| `tool(name, description, schema, handler)` | a tool declaration |
| `resource(uri, name, mime_type, reader)` | a resource declaration, read lazily |
| `prompt(name, description, arguments, renderer)` | a prompt declaration |
| `text(value)` | text content, `[{ type: "text", text }]` |
| `tool_error(message)` | a tool result flagged `isError` |
| `serve_stdio(offer, name = "ecko", version = "0.1.0")` | `null` once standard input closes |
| `serve_http(port, offer, name = "ecko", version = "0.1.0", host = "127.0.0.1", origins = [])` | blocks until the HTTP server stops |
| `handle(request, offer, name = "ecko", version = "0.1.0")` | the reply map for one decoded request, or `null` for a notification |
| `handle_text(text, offer, name = "ecko", version = "0.1.0")` | the reply as JSON text, or `null` for a notification |
| `progress(token, done, total = null)` | a progress notification, or `null` without a token |
| `progress_token(request)` | the request's `progressToken`, or `null` |
| `tools_changed()` | the tool-list-changed notification |
| `resources_changed()` | the resource-list-changed notification |
| `prompts_changed()` | the prompt-list-changed notification |

`offer` is `{ tools, resources, prompts }`, each optional; a bare list is the
tools.

### Messages and constants

| function | returns |
|---|---|
| `request(id, method, params = {})` | a JSON-RPC request map |
| `notification(method, params = {})` | a JSON-RPC notification map |
| `encode(message)` | the message as JSON text |
| `PROTOCOL_VERSION` | `"2025-06-18"`, the MCP revision both halves speak |
| `MAX_PAGES` | `1000`, how many pages of a list the client follows |
| `PAGE_SIZE` | `100`, how many entries a served list page holds |

## Notes

### Errors

| kind | reason | when |
|---|---|---|
| `mcp` | `server_error` | the server answered with a JSON-RPC error; the error carries its `code` and `detail` |
| `mcp` | `bad_reply` | the reply is not JSON, has neither result nor error, has no id, is a notification, or an HTTP body holds no reply to the request |
| `mcp` | `id_mismatch` | the reply answers a different request |
| `mcp` | `http_status` | the HTTP server answered outside 2xx; carries `status` and `body` |
| `mcp` | `no_reply` | a stdio server did not reply within `timeout_ms`, or exited |
| `mcp` | `repeated_cursor` | a server repeated a pagination cursor |
| `mcp` | `too_many_pages` | a list ran past `MAX_PAGES` pages |
| `net` | | the HTTP request could not be made |
| `bug` | `bad_argument` | an argument has the wrong type (the message names the function) |

The server half throws only `bug`/`bad_argument`, for a bad declaration or
offer. Nothing a client sends makes `handle`, `handle_text` or `serve_*`
throw: a malformed message is a JSON-RPC error reply (-32700 for bad JSON,
-32600 for an invalid request, -32602 for bad parameters or an unknown cursor,
tool, resource or prompt, -32601 for an unknown method, -32603 for a fault in
the server), and the server carries on with the next message.

### Two kinds of tool failure

A JSON-RPC error means the call was rejected - no such tool, bad arguments -
and the client raises `mcp`/`server_error`. `isError` on a result means the
call worked and the *tool* failed; MCP models this as data so a model can read
the failure and try something else. Check it with `is_error`. On the server, a
handler that throws (a map or a plain string) becomes an `isError` result
carrying the message.

### How the transports behave

- Lists are paginated; `tools`, `resources` and `prompts` follow `nextCursor`
  to the end, and a server that repeats a cursor is refused rather than looped
  on.
- Every reply is matched to its request id. On stdio, notifications the server
  sends in between (logging, progress) are skipped, a server `ping` is
  answered, any other server request gets error -32601, and a late reply to an
  earlier request is dropped.
- On HTTP, a response may be JSON or an SSE stream; the event carrying the
  reply is picked out of the stream. A notification answered `202` with an
  empty body is accepted.
- `serve_http` binds `127.0.0.1` unless given `host:`, and refuses (403) a
  browser `Origin` that is not loopback or listed in `origins:`, as the spec
  asks to prevent DNS rebinding.

### Not implemented

Server-to-client requests (sampling, roots, elicitation), resource
subscriptions, the SSE `GET` channel, a way for a tool handler to send progress
or log notifications, and the OAuth flow for hosted servers.

## Testing

```bash
ecko test
```

The suite is offline: the HTTP tests run a server on loopback in the same
process, and the stdio tests start `ecko` on the scripts in `testdata/`.

## License

[MIT](LICENSE)
