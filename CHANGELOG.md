# Changelog

## 0.56.0

### Fixed

- The HTTP client can now connect to a spec-compliant Streamable HTTP server,
  including this package's own `serve_http`. The empty `202` a server sends
  for a notification used to fail every `connect`.
- An HTTP response sent as an SSE stream with a notification ahead of the
  reply, or with a payload split over several `data:` lines, is now read
  correctly instead of failing to decode.
- The HTTP client sends the `MCP-Protocol-Version` header, echoes the
  `Mcp-Session-Id` a server hands out, and ends the session on `close`, so
  servers that keep sessions no longer reject every request after the first.
- On stdio, a notification, a server `ping` or a late reply arriving before
  the reply to a request is no longer taken as that reply. Server pings are
  answered.
- A reply with no id, or a notification, is refused rather than accepted as
  the answer to a request.
- One bad request no longer kills a stdio server. A cursor the server never
  issued is error -32602, a request that is not an object or has no method is
  -32600, parameters of the wrong shape are -32602, and any failure inside the
  server is -32603 with the detail written to standard error.
- A tool handler that throws a plain string (`error("boom")`) is reported as a
  tool error carrying the message, as a thrown map already was. A resource
  reader or prompt renderer that throws a string is error -32603.
- A tool result that cannot be encoded as JSON is error -32603 rather than a
  crash.
- A server reply that is not JSON is an `mcp` error rather than a bare `parse`
  error.

### Added

- Every `mcp` error carries a `reason` (`server_error`, `bad_reply`,
  `id_mismatch`, `http_status`, `no_reply`, `repeated_cursor`,
  `too_many_pages`); `server_error` also carries the server's `code`.
- `serve_http` takes `host:` and `origins:`, and refuses a browser `Origin`
  that is not loopback or listed, as the MCP spec asks.
- A wrong argument type is a `bug` error naming the function and argument.

### Breaking

- Options are named arguments instead of a map.
  Migration: `http_transport(url, { headers: h })` becomes
  `http_transport(url, headers: h)`; `stdio_transport(cmd, args, { timeout_ms: n })`
  becomes `stdio_transport(cmd, args, timeout_ms: n)`.
- The client and server identity is named arguments instead of an `info` map.
  Migration: `connect(t, { name: "app", version: "1.0" })` becomes
  `connect(t, name: "app", version: "1.0")`, and the same for `serve_stdio`,
  `serve_http`, `handle` and `handle_text`.
- `serve_http` binds `127.0.0.1` by default instead of every interface.
  Migration: pass `host: "0.0.0.0"` to listen on every interface.
- A cursor the server never issued is answered with error -32602 instead of
  raising an error out of `handle`. Migration: none needed unless you caught
  that error; check the reply's `error.code` instead.
