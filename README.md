# mcp

Model Context Protocol, both directions. Call an MCP server's tools, or be one.

MCP is JSON-RPC 2.0 with a fixed vocabulary: a handshake, then `tools/*`,
`resources/*` and `prompts/*`.

## Install

```bash
ecko get github.com/ecko-lang/mcp
```

## Calling a server

```ecko
import mcp

session = mcp.connect(mcp.stdio_transport("npx", ["-y", "@some/mcp-server"]))

for t in mcp.tools(session) {
    print("{t.name}: {t.description}")
}

print(mcp.text_of(mcp.call_tool(session, "search", { query: "ecko" })))
mcp.close(session)
```

Hand the tools straight to a model:

```ecko
answer = ai "what changed in the changelog?" using mcp.as_tools(session)
```

## Being a server

```ecko
import mcp

mcp.serve_stdio([
    mcp.tool(
        "word_count",
        "Count the words in some text",
        { type: "object", properties: { text: { type: "string" } }, required: ["text"] },
        fn(args) len(split(trim(args.text), " ")),
    ),
], { name: "words", version: "1.0" })
```

Register it with a client by pointing it at `ecko your_server.ecko`.

## Transports

**stdio** is how almost every MCP server ships: the client starts the process
and uses the pipes it already has, so there is no port and nothing to
authenticate. Needs **`exec`**.

```ecko
mcp.stdio_transport("npx", ["-y", "@some/mcp-server"], { timeout_ms: 30000 })
```

**HTTP** is the shape a hosted server takes. Needs **`net`**.

```ecko
token = secret(os.env("MCP_TOKEN"))
mcp.http_transport("https://mcp.example.com/", {
    headers: { Authorization: "Bearer {reveal(token)}" },
})
```

**`mcp.fake(handler)`** is a transport backed by a function, for tests. Needs no
capability at all.

## API

### Client

| call | result |
|---|---|
| `mcp.connect(transport, info?)` | Handshake and return a session. |
| `mcp.close(session)` | Close the transport. |
| `mcp.server_version(session)` / `mcp.server_capabilities(session)` | What the server answered with. |
| `mcp.tools(session)` | Every tool, following pagination. |
| `mcp.call_tool(session, name, args?)` | The raw result: `content`, maybe `isError`. |
| `mcp.text_of(result)` / `mcp.is_error(result)` | The text parts; whether the tool failed. |
| `mcp.as_tools(session)` / `mcp.as_tool(session, t)` | Tool specs for `ai ... using`. |
| `mcp.resources(session)` / `mcp.read_resource(session, uri)` | Resources. |
| `mcp.prompts(session)` / `mcp.get_prompt(session, name, args?)` | Prompts. |
| `mcp.cancel(session, id, reason?)` / `mcp.next_request_id(session)` | Cancel a request. |
| `mcp.call(session, method, params?)` / `mcp.notify(...)` | Anything not wrapped above. |

### Server

| call | result |
|---|---|
| `mcp.serve_stdio(offer, info?)` / `mcp.serve_http(port, offer, info?)` | Serve. Blocks. |
| `mcp.tool(name, description, schema, handler)` | Declare a tool. |
| `mcp.resource(uri, name, mime, reader)` | Declare a resource. Read lazily. |
| `mcp.prompt(name, description, arguments, renderer)` | Declare a prompt. |
| `mcp.handle(request, offer, info?)` | Answer one decoded request. `null` for a notification. |
| `mcp.handle_text(text, offer, info?)` | Decode, answer, re-encode. |
| `mcp.text(value)` / `mcp.tool_error(message)` | Build content; report a tool failure. |
| `mcp.progress(token, done, total?)` / `mcp.progress_token(request)` | Progress notifications. |
| `mcp.tools_changed()` / `mcp.resources_changed()` / `mcp.prompts_changed()` | Announce a change. |

`offer` is `{ tools, resources, prompts }`, each optional. A bare list is taken
as the tools.

## Two kinds of failure, and they are different

- **A JSON-RPC error** means the call was rejected: no such tool, bad argument,
  server fault. It raises an Ecko error with `kind: "mcp"`, carrying the
  server's own code and message.
- **`isError` on a successful result** means the call worked and the *tool*
  failed. MCP models this as data on purpose so a model can read the failure and
  try something else. Check it with `mcp.is_error(result)`.

Conflating them tells a model its arguments were wrong when the tool simply does
not exist.

## Notes

**Lists are paginated.** `tools`, `resources` and `prompts` follow `nextCursor`
to the end. Reading the first page and stopping is the worst kind of bug: a
short list, and nothing saying it was truncated. A server that repeats a cursor
is refused rather than looped on.

**Request ids never repeat**, and every reply is checked against the id that
asked for it. A transport that interleaves messages would otherwise attribute
someone else's result to your call.

**A notification is written but not read back.** Reading after one would block
until the timeout and then take the next reply, misaligning every later call.

**Capabilities are computed, not asserted.** A server advertises only the
sections it actually declared, because a client branches on that.

**A missing list reads as empty.** A server with no tools may answer `{}`.

## Not implemented

Server-to-client requests (sampling, roots, elicitation), resource
subscriptions, and the OAuth flow for hosted servers. Everything absent fails
loudly rather than silently.

## Testing

```bash
ecko test
```

56 cases, all offline against `mcp.fake` - no socket, no subprocess, no API key.

## License

MIT
