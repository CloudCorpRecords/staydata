# staydata

staydata is a small local bridge to the hosted HasData
Airbnb MCP server, which returns Airbnb data as JSON.
Give it a place — or a map bounding box — plus a
check-in date, and it searches the stays available
there for your dates and party; give it one listing's
URL, and it fetches that property's full page: host,
rooms and beds, photos, amenities, ratings and reviews
counts.

The bridge keeps no listing data of its own. It serves
the upstream server's tool list locally, exactly as
captured from the live endpoint, and forwards each
tool call upstream, returning the result untouched.
The captured surface is read-only: it searches and
reads public listing pages. There is no booking,
messaging, or payment tool in it.

## Tools

- `hasdata_airbnb_listing_getAirbnbListings` — search
  available stays. Required: `checkIn`. Provide either
  `location`, or all four bounding-box coordinates
  (`neLat`, `neLng`, `swLat`, `swLng`) to search inside
  a map box instead. Optional: `checkOut`, `adults`,
  `children`, `infants`, `pets`, and `nextPageToken`
  for paging. Returns listing id/url, title, nightly
  and total price, rating, review count, coordinates,
  and photos.
- `hasdata_airbnb_property_getAirbnbPropertyDetails` —
  fetch one listing's full property page. Required:
  `url` (a valid Airbnb listing URL). Returns title,
  description, location, coordinates, rating and
  review count, host profile, room/bed/bath counts,
  photos, and the amenities list.

## Configuration

- `HASDATA_AIRBNB_MCP_URL` (optional) — override the
  upstream endpoint. Defaults to
  `https://mcp.hasdata.com/mcp?apis=airbnb`. (The
  override hook is this rebuild's own labelled
  reconstruction.)
- `HASDATA_API_KEY` (optional) — when set, every
  upstream request carries an `x-api-key: <key>`
  header, for the caller's own HasData API key. When
  it is not set, no key header is sent. (The header
  name and the env name are pinned by the original's
  registry record; forwarding the key from this env
  var is a labelled reconstruction — the original
  surface was captured keyless.)

### Forwarding mechanics

On the first tool call the client opens a streamable
HTTP session with the upstream: a JSON-RPC
`initialize` request (protocolVersion `2025-03-26`,
clientInfo name `staydata`), then a
`notifications/initialized` notification, then the
`tools/call` request carrying the caller's arguments
unchanged. When the upstream answers `initialize`
with an `Mcp-Session-Id` header, that value rides on
every later request of the session, which is reused
for the life of the process. Replies may arrive as
plain JSON or as server-sent events; both are read.

Successful upstream results pass through verbatim,
including any `isError` flag the upstream sets.
Failures — a JSON-RPC error, a non-2xx status, an
unreadable response, or a request that cannot be
completed — come back as an `isError` result whose
text starts with `staydata error:` (this rendering
is this rebuild's own labelled reconstruction).

## CLI

Each tool is also a subcommand taking one JSON
argument string:

```bash
npx @reneturcios/staydata hasdata_airbnb_listing_getAirbnbListings '{"location":"San Francisco","checkIn":"2026-11-10","checkOut":"2026-11-14","adults":2}'
npx @reneturcios/staydata hasdata_airbnb_property_getAirbnbPropertyDetails '{"url":"https://www.airbnb.com/rooms/12345678"}'
npx @reneturcios/staydata --version
```

Exit codes: `0` on success, `1` on an `isError`
result, `2` on a usage error.

## MCP server

```bash
npx -y @reneturcios/staydata mcp
```

MCP client config:

```json
{
  "mcpServers": {
    "staydata": {
      "command": "npx",
      "args": ["-y", "@reneturcios/staydata", "mcp"]
    }
  }
}
```

## Docker

```bash
docker run -i ghcr.io/cloudcorprecords/staydata
docker run --rm ghcr.io/cloudcorprecords/staydata --version
```

The container's default command is `mcp`, so the first
form is a stdio MCP server; any other arguments reach
the CLI normally.

## Development

```bash
npm install
npm run build
npm test
npm run dev
```

The test suite runs entirely against a stubbed
JSON-RPC upstream — no live network calls.

## License

MIT — see `LICENSE`.
