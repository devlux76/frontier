# SKIBIDI: what it was and what is worth keeping (2026-11-16)

Sources (shallow clones, deleted after; paths repo-relative): syntropiq/skibidi-js, devlux76/skibidi-demo,
devlux76/skibidi-dash, syntropiq/js-libp2p `packages/protocol-http` (sparse). Nothing was run or installed.

## Vision
A Supabase-shaped backend living in browser tabs and peers instead of a hosted Postgres. README goal: apps
keep working and keep their data after the origin server disappears; signed data; live streams on the same
store. Censorship-resistant reading: an app's data and HTTP surface are served by whichever peers still hold
them, and browsers serve HTTP to each other over libp2p. What was built is much smaller (see Known gaps).

## 1. Supabase API mapping (designed vs built)
| Supabase | SKIBIDI docs promised | Code actually has |
|---|---|---|
| `from(t).select().eq()` | JS workalike client | nothing; `GET /api/tables/:t?col=eq.val&select=&order=&limit=&offset=` |
| insert/update/delete | client, auto-propagates | `POST/PATCH/DELETE /api/tables/:t[/:id]`, local only |
| `auth.*` | public-key identity | none |
| `channel().on()` | GossipSub push | topics subscribed, nothing published |
| storage / edge / RLS | OPFS, WASM, encrypted shares | none (`rpc/:fn` runs SQL stored in a table) |
Sources: skibidi-js/src/http/router.ts, src/index.ts (SkibidiClient: createTable, dropTable, tableExists, registerFunction, query, createServer), src/db/query-builder.ts.
**For us:** the reusable part is the PostgREST filter grammar `col=op.value` with a fixed operator map
(eq, neq, gt, gte, lt, lte, like, ilike, in, is). Moon's `feed` on `/vortex/1` could take read-only filters
the same way. Do not copy the JS client shape; the docs and demo call APIs that do not exist.

## 2. Data model and signing (designed, not built)
Tables are plain PGLite tables from `SchemaManager.createTable(TableDefinition)` (src/db/schema-manager.ts).
No per-row author, signature or version. The only "signing" hook is the gossip message id
(skibidi-js/src/p2p/libp2p.ts):
```ts
export async function msgIdFnStrictNoSign(msg: Message): Promise<Uint8Array> {
  const encodedSeqNum = enc.encode(signedMessage.sequenceNumber.toString());
  return await sha256.encode(encodedSeqNum);
}
```
Dedup identity is the sender's seqno, so equal seqnos from different peers collide. Nothing in src signs or verifies.
**For us:** VORTEX records already carry signatures and stamps (v6/webhub). Dedup identity should be a hash of
the signed record body, never a sender counter.

## 3. Replication over gossip (designed, never wired)
`gossipsub({ allowPublishToZeroTopicPeers, msgIdFn, ignoreDuplicatePublishError })` is configured and two topics
are subscribed (src/p2p/libp2p.ts), but no code calls `publish`. Writes hit local PGLite only (tables.ts calls
`dbManager.query`). Replication did not exist.
**For us:** push-only gossip loses records across reconnects. Use Moon's pull model (`Puller` + cursor on `feed`;
moon.md phase 4 "sees every record exactly once"). Gossip, if used, is only a hint to advance the cursor.

## 4. Persistence: PGLite (OPFS on paper only)
`PGLiteManager` (src/db/pglite-manager.ts, singleton) calls `createPGlite({ name, debug, inMemory })` and exposes
`query(sql, params)` and `transaction(cb)`. No OPFS code exists.
**For us:** Moon's IndexedDB `RecordStore` is the equivalent seam; keep a single `transaction(cb)` so multi-row
writes are atomic. SKIBIDI never tested the bar Moon must: keys, cursors and wallet recover together after reload.

## 5. HTTP-over-libp2p wire format
Protocol `/https/1.1`; one request and one response per stream, framed with `it-protobuf-stream`
(`pbStream(stream).write(req)` then `.read(Res)`), then close. Verbatim from
`packages/protocol-http/src/protobuf/split/10-http-message.proto` (ControlData elided, see file):
```proto
message Field { string name = 1; string value = 2; }
message HttpMessage {
  ControlData control_data = 1;
  repeated Field headers = 2;
  bytes content = 3;
  repeated Field trailers = 4;
}
message HttpRequest  { HttpMessage base_message = 1; string method = 2; string target_uri = 3; string protocol_version = 4; }
message HttpResponse { HttpMessage base_message = 1; int32 status_code = 2; string reason_phrase = 3; string protocol_version = 4; }
```
`08-uri.proto`: `Uri{scheme,authority,path,query,fragment}`, `Origin{scheme,host,port}`.
Defect: `ControlData` duplicates method, target and status that also sit at top level.
**For us:** Moon's `/vortex/1` is already length-prefixed request/response per stream, but carries signed records,
not HTTP headers. If Moon serves HTTP-style reads to browsers, reuse this framing and skip ControlData.

## 6. Routing and handlers
`PostgRestServer` (src/http/server.ts) registers one handler on `apiPrefix` (default `/api`), then linear-matches
method + `parsePathParams`. Route table (src/http/router.ts):
```
GET    /tables/:tableName[/:id]     list / get (filters, select, order, limit, offset)
POST   /tables/:tableName           create (array body runs in a transaction)
PATCH  /tables/:tableName[/:id]     update with filter
DELETE /tables/:tableName[/:id]     delete with filter
POST   /rpc/:functionName           registered SQL function
GET    /schema[/tables[/:tableName]]
```
Handler type (src/types/http.ts): `HttpHandler = (request: HttpRequest) => Promise<HttpResponse>`.
**For us:** linear matching is fine at this size; the per-route handler type lets a Moon tool be a route
without a second dispatcher.

## 7. How a browser tab serves requests
`createNode({ enableHttpServer })` sets `services.http = httpServer()` (npm `@libp2p/http`) and listens on `/webrtc`
plus relay circuit addresses (`getRelayListenAddrs`). The tab answers while open; reachability comes from a
circuit-relay reservation plus WebRTC/DCUtR. Serving stops when the tab closes.
**For us:** Moon's phase-1 story; same limit, availability = tab uptime. State it in any claim.

## 8. Discovery and direct messages
- Relay addresses: delegated routing `https://delegated-ipfs.dev`, looked up for a hard-coded bootstrap peer id
  (src/p2p/constants.ts). Peers found via `pubsubPeerDiscovery` on `universal-connectivity-browser-peer-discovery`.
- WebRTC dials one at a time (`dialWebRTCMaddrs`); DM streams (`/universal-connectivity/dm/1.0.0`) time out at 5 s.
  Constants are inherited from libp2p's universal-connectivity example.
**For us:** same stack as Moon's rendezvous + relay plan. Pubsub discovery is a cheap browser-to-browser bootstrap
fallback; the one-dial-at-a-time rule is worth copying.

## What worked vs stubs
In code (unit tests exist; NOT run here, per rule): PGLite wrapper, PostgREST-style query builder, table CRUD
handlers, transaction helper, libp2p factory (gossipsub, identify, ping, dcutr, webRTC, circuit relay v2,
pubsub discovery), route table over `@libp2p/http`.
Stubs or absent: signing, auth, OPFS, realtime publish, RLS, edge functions, the `.from()` client, `.on()` events.

## Known gaps / what never worked
- Demo (`skibidi-demo/src/js/app.js`) calls `createClient('http://localhost:3000','demo-key')`, `auth.login`,
  `query('posts').insert`; dash (`skibidi-dash/src/lib/client.ts`) calls `client.on('peer:connected' | ...)`.
  None of these exist in the library.
- Identifiers are interpolated unescaped (`"${tableName}"`, `"${col}"` in query-builder.ts); a double quote breaks
  the statement. Raw SQL `query()` is public API. Treat as unsafe. `rpc` runs stored SQL with no sandbox.
- No replication, conflict rule or ACL; PLAN.md P2P-sync items unchecked. protocol-http: caching, range and
  conditional requests partial; auth is transport-only.

## Not worth taking
The Supabase-clone framing and JS `.from()` workalike; the duplicated `ControlData` fields; GossipSub as the
replication primitive with seqno message ids; the hard-coded WebTransport bootstrap peer id; the dash D3 view
as a reference (it listens for events the library never emits).
