# MoQ local dev setup

End-to-end instructions for running the MoQ pipeline locally — a Rust
moq-relay, a pipecat MoQ bot, and the prebuilt React client talking to
both through the locally-built `voice-ui-kit` + `@pipecat-ai/moq-transport`.

The MoQ wire bits are unpublished, so a few `npm link` chains are needed.
Most of the friction is handled by `./scripts/moq-dev-setup.sh` in this
repo.

## 0. Layout

Clone all five repos as siblings under one parent directory. The setup
script assumes this layout:

```
<parent>/
├── pipecat/                          # github.com/pipecat-ai/pipecat
├── pipecat-client-web-transports/    # github.com/pipecat-ai/pipecat-client-web-transports
├── pipecat-prebuilt/                 # internal
├── voice-ui-kit/                     # github.com/pipecat-ai/voice-ui-kit
└── moq-relay/                        # github.com/kixelated/moq-rs
```

The four pipecat repos all live on the **`vp-add-moq-transport`**
branch for this work. `moq-relay` is whichever revision builds —
`main` has worked.

```bash
for repo in pipecat pipecat-client-web-transports pipecat-prebuilt voice-ui-kit; do
  (cd "$repo" && git checkout vp-add-moq-transport)
done
```

## 1. Tooling prerequisites

- Node 18+ and `npm`
- `pnpm` 10+ (the corepack shim on some Node versions has signing-key
  issues — if `pnpm install` fails with a corepack error, install it
  globally instead: `npm install -g --force pnpm@10.14.0`)
- `uv` for the Python side
- `cargo` / Rust toolchain for the relay
- `openssl` (macOS ships with LibreSSL — that works)

## 2. Build the JS packages

These three need to be installed and (where applicable) built once,
before the link chain can succeed. Paths are relative to `<parent>/`.

```bash
# @pipecat-ai/moq-transport
cd pipecat-client-web-transports
npm install
cd transports/moq-transport
npm run build

# @pipecat-ai/voice-ui-kit (must produce dist/)
cd ../../../voice-ui-kit
pnpm install
pnpm -F @pipecat-ai/voice-ui-kit build

# pipecat-prebuilt/client (npm install is enough — vite builds on demand)
cd ../pipecat-prebuilt/client
npm install
```

A couple of things to know:

- `voice-ui-kit/package/package.json` declares `@pipecat-ai/moq-transport`
  as a `link:` reference to its sibling repo. `pnpm install` resolves
  that automatically — no extra step. (This will change once
  moq-transport ships to npm.)
- Python side: from this `pipecat/` repo, `uv sync --extra moq`.

## 3. Run the dev setup script

From the `pipecat/` repo root:

```bash
./scripts/moq-dev-setup.sh ../moq-relay
```

The script does four things:

1. Generates a self-signed cert (`localhost`, 14 days — the WebTransport
   limit) into `pipecat/.moq-certs/`.
2. Symlinks `moq-cert.pem` and `moq-key.pem` into both `pipecat/` and
   the relay dir so each side can find them by relative path.
3. Prints the cert SHA-256 (the bot threads it back through `/start`
   so the browser can pin WebTransport against it).
4. Sets up the `npm link` chain so the prebuilt client picks up the
   local moq-transport + the local voice-ui-kit build, and so
   `@pipecat-ai/client-js` is a single instance across all three
   packages (prebuilt, voice-ui-kit, moq-transport). Without that
   dedupe, the React `PipecatClientProvider` context breaks across
   two copies.

The link chain is temporary — delete that block from the script once
both packages ship to npm.

## 4. Run the stack

The setup script prints these at the end; they're listed here for
reference. Three terminals.

### Terminal 1 — moq-relay

```bash
cd ../moq-relay
cargo run --bin moq-relay -- \
  --server-bind '[::]:4080' \
  --tls-cert moq-cert.pem \
  --tls-key moq-key.pem \
  --auth-public ''
```

### Terminal 2 — pipecat bot

From this `pipecat/` repo:

```bash
uv run python examples/transports/transports-moq.py \
  -t moq \
  --moq-cert moq-cert.pem \
  --moq-insecure \
  --moq-path /
```

`--moq-insecure` tells the bot to accept the self-signed cert; the
fingerprint is still pinned on the browser side.

### Terminal 3 — prebuilt client dev server

```bash
cd ../pipecat-prebuilt/client
npm run dev
```

Open <http://localhost:7860>, pick **MoQ** in the transport dropdown.
The `ConsoleTemplate` should drive the full UI — connect/disconnect,
transcript, mic device picker, SessionInfo showing "MoQ".

## 5. Iterating

- **Edited `moq-transport/src/*`?** Rebuild it: `cd
  pipecat-client-web-transports/transports/moq-transport && npm run build`.
  The npm link picks up the new `dist/` immediately; restart the
  prebuilt dev server (Vite caches imports).
- **Edited `voice-ui-kit/package/src/*`?** Rebuild: `pnpm -F
  @pipecat-ai/voice-ui-kit build` (or `build:watch`). Same restart
  caveat for the consumer.
- **Edited `pipecat-prebuilt/client/src/*`?** Vite hot-reloads
  automatically.
- **Edited the pipecat bot?** Restart terminal 2.
- **Cert expired?** Re-run `./scripts/moq-dev-setup.sh ../moq-relay`.
  The link chain is idempotent.

## 6. Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Browser console: `Failed to load transport "moq"` | moq-transport isn't linked into prebuilt. Re-run the setup script and check the "Linking …" output. |
| `Property '_options' is protected` during typecheck | Two copies of `@pipecat-ai/client-js` are visible. The setup script's `npm link @pipecat-ai/client-js` chain is the fix — confirm it ran for both `pipecat-client-web-transports/` and `voice-ui-kit/package/`. |
| Browser: "MoqTransport requires `relayUrl`" | `/start` didn't return the `moq` block. Bot probably isn't running with `-t moq`. |
| WebTransport handshake fails with `net::ERR_QUIC_PROTOCOL_ERROR` | Cert expired (14-day limit) or the fingerprint pinned in the browser doesn't match the cert the relay loaded. Re-run the setup script. |
| `pnpm install` errors with corepack signing-key complaint | Install pnpm globally: `npm install -g --force pnpm@10.14.0`. |
| Stale voice-ui-kit code after editing | `pnpm -F @pipecat-ai/voice-ui-kit build` was skipped or failed. |

## 7. What the link chain actually does

For when something breaks and the script's output isn't enough.

```
pipecat-prebuilt/client/node_modules/@pipecat-ai/
├── moq-transport     -> ../../../../pipecat-client-web-transports/transports/moq-transport
├── voice-ui-kit      -> ../../../../voice-ui-kit/package
├── client-js         (real copy, npm-installed at 1.8.x — the canonical instance)
└── ...

pipecat-client-web-transports/node_modules/@pipecat-ai/
├── client-js         -> <prebuilt's client-js>  (linked, so moq-transport shares it)
└── ...

voice-ui-kit/package/node_modules/@pipecat-ai/
├── moq-transport     -> ../../../../pipecat-client-web-transports/transports/moq-transport  (pnpm link:)
├── client-js         -> <prebuilt's client-js>  (linked, so voice-ui-kit shares it)
└── ...
```

The whole point is that every layer that imports `@pipecat-ai/client-js`
resolves to the same `dist/` directory. Vite resolves symlinks to real
paths by default (`resolve.preserveSymlinks: false`), so without the
client-js link chain you'd end up with three independent copies and a
broken React context.
