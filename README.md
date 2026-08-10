# mcp

A Model Context Protocol client. Connect to an MCP server, list and call its
tools, read its resources and prompts.

MCP is JSON-RPC 2.0 with a fixed vocabulary: a handshake, then `tools/*`,
`resources/*` and `prompts/*`. This package speaks it over HTTP.

## Install

```bash
ecko get github.com/ecko-lang/mcp
```

## Usage

```ecko
import mcp

session = mcp.connect(mcp.http_transport("https://mcp.example.com/"))

for t in mcp.tools(session) {
    print("{t.name}: {t.description}")
}

result = mcp.call_tool(session, "search", { query: "ecko" })
print(mcp.text_of(result))

mcp.close(session)
```

For a server behind a token:

```ecko
token = secret(os.env("MCP_TOKEN"))
session = mcp.connect(
    mcp.http_transport("https://mcp.example.com/", {
        headers: { Authorization: "Bearer {reveal(token)}" },
    }),
)
```

## API

| call | result |
|---|---|
| `mcp.connect(transport, info?)` | Handshake and return a session. `info` is `{ name, version }`. |
| `mcp.close(session)` | Close the transport. |
| `mcp.server_version(session)` | The protocol version the server answered with. |
| `mcp.server_capabilities(session)` | What the server advertises: `tools`, `resources`, `prompts`. |
| `mcp.tools(session)` | `[{ name, description, inputSchema }]` |
| `mcp.call_tool(session, name, args?)` | The raw result, carrying `content` and maybe `isError`. |
| `mcp.text_of(result)` | The text parts of a result, joined. |
| `mcp.is_error(result)` | Whether the *tool* reported failure. |
| `mcp.resources(session)` | `[{ uri, name, mimeType }]` |
| `mcp.read_resource(session, uri)` | `[{ uri, mimeType, text }]` |
| `mcp.prompts(session)` | `[{ name, description, arguments }]` |
| `mcp.get_prompt(session, name, args?)` | The rendered `messages` list. |
| `mcp.call(session, method, params?)` | Any method, for parts of the protocol this package does not wrap. |
| `mcp.notify(session, method, params?)` | Send a notification. |
| `mcp.http_transport(url, opts?)` | HTTP transport. `opts` takes `headers`. |
| `mcp.fake(handler)` | A transport backed by a function, for tests. |

## Capability

The HTTP transport needs **`net`**. Nothing else here touches the outside
world, so a program using `mcp.fake` needs no grant at all.

## Two kinds of failure, and they are different

A tool can fail in two ways, and conflating them will cost you an afternoon:

- **A JSON-RPC error** means the call was rejected: no such tool, bad argument,
  server fault. This raises an Ecko error with `kind: "mcp"`, carrying the
  server's own code and message.
- **`isError` on a successful result** means the call worked and the *tool*
  failed. MCP models this as data on purpose, so a model can read the failure
  text and try something else. Check it with `mcp.is_error(result)`.

## The transport is a value

`connect` takes a transport rather than a URL, so the protocol layer never
learns whether the server is across a network. That is what lets the whole
package be tested offline against `mcp.fake`, and it is where a stdio transport
will plug in.

## No stdio transport yet

Most MCP servers today run as a local subprocess speaking JSON-RPC over stdin
and stdout. This package cannot do that, because `std.proc` spawns children
with `stdin` closed - there is no way to write to a running child. Adding one
primitive to `std.proc` is what unblocks it; the transport itself is then about
thirty lines.

Until then this package works with servers reachable over HTTP.

## Notes

**Request ids never repeat within a session**, and every reply is checked
against the id that asked for it. A transport that interleaves messages (an SSE
channel carries notifications and other calls' replies on the same stream) would
otherwise attribute someone else's result to your call.

**A POST may come back as JSON or as an SSE frame.** The Streamable HTTP
transport allows either, so the response body is unwrapped before decoding.

**A missing list reads as empty.** A server with no tools may answer `{}`
rather than `{"tools": []}`; both give you `[]`.

## Testing

```bash
ecko test
```

21 cases, all offline against `mcp.fake` - no socket, no API key, no server.

## License

MIT
