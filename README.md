# @plannotator/agent-bridge

The host-agnostic bridge client for the Plannotator agent dispatch protocol.
It is the shared core under every adapter that connects a coding agent to an
[Artifact Server](https://github.com/plannotator/artifact-server) (or any
server implementing the protocol): review comments dispatched from the web UI
arrive in the agent's session as follow-up work, and the agent's replies
close the loop.

The protocol itself is documented in [PROTOCOL.md](./PROTOCOL.md).

## What it does

- **Connect** — resolves credentials (environment or the local server's
  discovery record), registers a self-named agent with a stable identity
  that survives restarts, declares capabilities, and reads the protocol
  version handshake.
- **Claim** — long-polls the dispatch mailbox; the poll is the heartbeat
  that makes presence honest.
- **Render** — turns a claimed bundle of comment threads into one readable
  message, stripping bidirectional-override and invisible Unicode from all
  untrusted text before it enters an agent's context.
- **Deliver** — hands the message to the host through a three-method port,
  follow-up only (never interrupting), held while the host compacts, and
  reports `delivered` only after the host accepted it.
- **Close the loop** — comment reply/resolve operations for the host's
  `artifact_comments` tool.
- **Presence** — a fire-and-forget activity beacon (`thinking` /
  `replying` / `idle`) at natural work boundaries.
- **Fail open** — no configuration means one notice then dormancy; a dead
  server means bounded backoff; no failure ever reaches the host.

## Writing an adapter

An adapter supplies a `HostPort` — three functions — and gets everything
above:

```ts
import {startBridge, resolveBridgeCredentials} from "@plannotator/agent-bridge";

const credentials = await resolveBridgeCredentials(environment, homedir());
const bridge = startBridge({
  agentSessionId: null,
  credentials,
  displayName: "my-project",
  fetchImplementation: fetch,
  host: {
    isCompacting: () => false,
    notify: (message, kind) => { /* surface a notice in your host */ },
    sendUserMessage: async (text) => { /* await host admission */ },
  },
  hostname: hostname(),
  kind: "my-harness",
  workingDirectory: process.cwd(),
});
```

Reference adapters live in the Artifact Server repository under
`integrations/`: the Pi extension, the OpenCode plugin, and the Claude Code
channel — each ~250 lines over this core.

`sendUserMessage` may return `void` for a synchronous host or a promise for an
asynchronous host. Resolve that promise only after the host admits the message.
Rejecting it fails that dispatch and leaves the bridge available for later
work. Reserve a synchronous throw for an invalid host handle that requires the
claim loop to stop.

`renderBundleMessage(bundle)` defaults to the native/channel instruction that
names `artifact_comments`. Pass `"mailbox"` as its second argument when the
receiving agent instead has the MCP `comment_reply` and `comment_resolve` tools.
Both profiles use the same sanitization and message structure.

## Ships TypeScript source

The package exports `index.ts` directly. Every current host loads TypeScript
natively (Pi via jiti, OpenCode via Bun, channel processes via tsx); a
compiled distribution can be added when a consumer needs one.

## Conformance

The protocol's conformance suite (BRP/PRS requirement IDs) runs in the
Artifact Server repository against a real server, with this package as the
client under test.

## License

MIT
