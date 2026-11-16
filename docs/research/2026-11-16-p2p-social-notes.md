# Smoke and universal-connectivity: what they were and what is worth keeping (2026-11-16)

Sources (read via `gh api`, nothing cloned into ~/projects, nothing run or installed):
- devlux76/smoke, branch `feature/universal-connectivity` (head 6814312), against upstream sinclairzx81/smoke `master` (bdb75c3). Our fork `master` (22febf07) is one commit ahead (.gitignore only).
- syntropiq/universal-connectivity, branches `seperate-ui` (head d2fb3ca) and `feature/headless` (head d98afdc), against upstream libp2p/universal-connectivity (merge base 18f0504e, upstream main has moved 25 commits past it).

## What they were
- **smoke** (upstream): "Run Web Servers in Web Browsers over WebRTC". A `Network` object gives each browser tab a virtual HTTP/TCP/WebSocket/media layer over WebRTC data channels. `Http.listen({port})` in one tab, `Http.fetch(url)` from another. Signalling (ICE/SDP) goes through a pluggable `Hub`. Goal: user services run peer-to-peer in the browser, away from the cloud. Readme: `readme.md`.
- **universal-connectivity** (upstream): a realtime P2P chat showcasing libp2p transports (WebRTC, WebTransport, WebSocket, circuit relay) across Go, Rust and TS. Chat is GossipSub pubsub.
- Ours: an attempt to (a) put a libp2p hub under smoke's Hub seam, and (b) run UC's TS peer as a headless node with no browser. Neither reached a verified working state in the sources I could read.

## Mechanisms (numbered)

1. **Browser-hosted HTTP over WebRTC** (`src/http/http.mts`, `src/http/listener.mts`, `src/http/fetch.mts`, `src/webrtc/`). A tab is a server on a virtual port; requests and responses are framed over a data channel. Signal frames `---RESPONSE---` and `---SOCKET---` (`src/http/signal.mts`) mark response streams and WebSocket upgrades.
   **For us:** Moon can serve a site or API from whichever browser tab still holds it, addressed by a URL-like name rather than an IP. The cost is that the host must stay online, so this covers availability only while a holder is up. Keep the framing (a few signal bytes ahead of a stream) as the wire contract to copy, not the code.

2. **Signalling Hub as the seam** (`src/hubs/hub.mts`, `src/network.mts:47`). The `Hub` interface is five methods: `configuration`, `address`, `send`, `receive`, `dispose`. `Network({hub})` injects any implementation. Three exist: `Private` (BroadcastChannel, same browser only, `src/hubs/private.mts`), `Public` (WebSocket client to an endpoint, `src/hubs/public.mts`; readme says "pending"), and ours (`Libp2pHub`).
   **For us:** VORTEX should keep exactly this split: transport-agnostic WebRTC on top, a swappable signalling Hub below. Hub messages carry only ICE/SDP, so a relay that sees them learns little beyond who talks to whom.

3. **Addressing** (`src/hubs/private.mts`, `Crypto.randomUUID()` in both smoke hubs). A peer's address is a random UUID per page load. The `from` field is set by the sender and never checked, so anyone can claim any address. Our `DESIGN.md` ("Replacing Peer Discovery", considerations list) says "consider authentication" and nothing implements it.
   **For us:** Moon addresses must be self-certifying (a public key or its hash) and every signalling message signed. Without that, a censor or a hostile peer can impersonate a hub-level identity. Do not reuse the UUID scheme.

4. **libp2p as the signalling transport, our `Libp2pHub`** (`src/libp2p/Libp2pHub.ts`, `P2PManager.ts`, `PeerDiscovery.ts`). All signals go to one GossipSub topic `smoke.signaling`. Each hub filters by `message.to === this.address`. Discovery uses `pubsubPeerDiscovery` plus delegated routing over HTTP (`@helia/delegated-routing-v1-http-api-client`) and bootstrap peers.
   **For us:** the pattern is right (one shared topic, filter by recipient, discovery via pubsub). Two problems. Every signal reaches every subscriber, which leaks the social graph to any observer of the topic. And the HTTP routing and bootstrap endpoints are a hard dependency on third-party infrastructure, which is exactly the chokepoint a censor targets. For Moon: use a per-recipient topic or encrypted envelope, and make bootstrap a configurable list with at least two independent kinds of path.

5. **Headless libp2p node** (UC `seperate-ui`: `js-peer/src/lib/libp2p-core.ts` `createHeadlessLibp2p`, `js-peer/src/demo-headless.ts`, `js-peer/src/index-headless.ts`). Node built with no React. Listen addresses are only `'/webrtc'` plus circuit-relay addresses from the delegated router (`libp2p-core.ts` ~line 41-46). Transports: WebRTC, WebSockets, WebTransport, circuit relay; gossipsub; pubsub peer discovery. The `feature/headless` version of the same code (`js-peer/js-peer-lib/src/lib/libp2p.ts`) is the same shape.
   **For us:** a headless peer is the shape Moon needs for relay and seed nodes. The likely cause of "builds and runs but does not bind" (commit 4792f21 on `feature/headless`) is inference, not verified: the node has no direct TCP or WebSocket listen address, so it is reachable only through relay reservations, which depend on the router. Lesson for Moon: always declare at least one direct listen address next to relay listens, and log the resulting `getMultiaddrs()` on startup as a bind check.

6. **Chat and pubsub model** (UC `js-peer/js-peer-lib/src/lib/constants.ts`). Public chat is GossipSub topic `universal-connectivity`; file bytes use a separate topic `universal-connectivity-file`; direct messages use a stream protocol `/universal-connectivity/dm/1.0.0` (`src/lib/direct-message.ts`, protobuf in `src/lib/protobuf/`); file exchange uses `/universal-connectivity-file/1`.
   **For us:** Moon should split the same way: pubsub for open channels, one protocol stream per peer pair for direct messages, and a separate path for bulk payloads so chat stays small. The protobuf DM envelope is worth reading before we design ours.

## Our commits: what they added and whether they worked

**devlux76/smoke, `feature/universal-connectivity`** (4 commits, 2025-03-01, on top of upstream master bdb75c3):
- `fa6fdf0` add Channel and P2PManager for pubsub messages.
- `82ad7f8` .gitignore: bun.lock.
- `2c5d760` fix "Dialing" spelling; add `DESIGN.md` (252 lines, "Smoke Network Architecture: HTTP over WebRTC").
- `6814312` PeerDiscovery, example, `test.ts`, and an `index.ts`.
- Files: `src/libp2p/{Channel,Libp2pHub,P2PManager,PeerDiscovery,example,index,test}.ts`, `DESIGN.md`, `.gitignore`. Upstream `network.mts`, the Hub interface and CI were not changed, so `Libp2pHub` is opt-in via `Network({hub})` and is not wired in.
- Status: **not verified.** The commits make no claim of a passing run, and I found no CI change. `test.ts` documents running with `node -r ts-node/register`, which breaks our Bun-only rule. Its imports omit the `.mjs` extension that upstream uses, so it may not match the build.

**syntropiq/universal-connectivity, `seperate-ui`** (4 commits, 2025-02-21):
- `9a9b030`, `2b5e8d0`, `801fd49`, `d2fb3ca`: headless libp2p support, a headless demo, a headless entry point, and webpack and next config changes.
- Status: **not verified.** The last commit's message only says it was added to webpack config. No run is recorded.

**syntropiq/universal-connectivity, `feature/headless`** (16 commits ahead of base, head d98afdc):
- Adds `js-peer-bun` and `js-peer-lib`, and refactors `js-peer-react` to use `P2PNode`.
- Status: **does not build.** Head commit message is "Doesn't build but lots of changes"; the earlier commit 4792f21 says "builds and runs but does not bind". This branch was superseded in intent by `seperate-ui`.

## Not worth taking
- Smoke's `Private` hub as a social transport. It only reaches tabs in one browser process; useful for tests, nothing else.
- Our single shared signalling topic that broadcasts every signal (item 4 above) as-is.
- Delegated routing or bootstrap as hard, single-provider dependencies.
- Our `test.ts` runner (Node with ts-node).
- Smoke's Media, Net, fs and async modules: large, off-goal for now.
- UC's React chat context and component refactor (`chat-ctx.tsx`, `chat.tsx`): UI churn with no protocol content.
- Random per-session UUIDs as addresses.
