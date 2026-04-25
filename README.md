# syncer

WebAssembly bindings that turn a Sia consensus syncer into something you can run inside a browser tab. Provides P2P header sync over WebTransport, block-filter generation, txindex, UTXO scanning, and v2 transaction build / sign / broadcast — no backend required.

Used by [Sialo Browser](https://github.com/Alrighttt/sialo_browser) for its in-browser wallet, blockchain explorer, and on-chain attestation manifest tooling.

## What it does

- **Connects to Sia peers over WebTransport** (HTTP/3 / QUIC) and runs the v2 syncer protocol from the browser.
- **Header sync** — downloads and verifies the chain of block headers, persisted to OPFS.
- **Block filters + txindex** — generates the filter / txindex artifacts the wallet uses to scan address activity in O(log N) per address rather than full-block replay.
- **UTXO scanning** — derives addresses from a BIP-39 seed, scans for matching UTXOs, returns balance + history.
- **V2 transactions** — builds, signs, and broadcasts v2 transactions; computes Merkle proofs for state-element inclusion; watches the mempool.
- **Manifests** — produces and resolves on-chain attestation manifests (the primitive Sialo Browser uses for wallet backup, channel pointers, and group membership).

Everything runs on the WASM event loop via `wasm_bindgen_futures::spawn_local`. No service workers, no shared workers — the parent page calls into the bindings and consumes the results.

## Layout

This repo is the WASM-binding crate. It depends on three Rust crates that aren't published to crates.io yet:

| Dependency | Source | Notes |
|---|---|---|
| `sia_syncer` | currently `path = "../syncer"` | v2 syncer protocol types and RPC encoding |
| `sia_sdk` | currently `path = "../sia_sdk"` | core SDK (cryptography, encoding, transactions) |
| `sia-sdk-derive` | pulled transitively | proc-macros used by `sia_sdk` |

To build standalone, either:

1. Check out matching versions of those crates next to this repo so the path deps resolve:
   ```
   ~/repos/
     ├── sia_sdk/      (lib name "sia"; package `sia_sdk`)
     ├── syncer/       (package `sia_syncer`)
     └── syncer_wasm/  (this repo)
   ```
2. Or rewrite `Cargo.toml` to use git deps:
   ```toml
   sia_syncer = { git = "https://github.com/Alrighttt/sia-sdk-rs", branch = "master" }
   sia_sdk    = { git = "https://github.com/Alrighttt/sia-sdk-rs", branch = "master" }
   ```

## Build

Requires `wasm-pack` (install via `curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh`).

```bash
RUSTFLAGS='--cfg=web_sys_unstable_apis' wasm-pack build --release --target web
```

The `--cfg=web_sys_unstable_apis` flag is required — `web-sys`'s `WebTransport` types are gated behind it. Output lands in `pkg/`:

- `syncer_wasm.js` — JS shim
- `syncer_wasm_bg.wasm` — the WASM module itself
- `syncer_wasm.d.ts`, `syncer_wasm_bg.wasm.d.ts` — TypeScript declarations
- `snippets/` — inline JS shims emitted by `wasm-bindgen` (only when present)

## Use from a browser

```html
<script type="module">
  import init, { /* exported symbols */ } from './pkg/syncer_wasm.js';

  await init();
  // … call into the bindings
</script>
```

Sialo Browser drops these files into its `pkg/` directory and imports them like any other ES module — see [`sialo_browser/.vscode/tasks.json`](https://github.com/Alrighttt/sialo_browser/blob/main/.vscode/tasks.json) for the build-and-sync task it uses during development.

## Browser support

Chrome/Edge and Firefox 114+ on platforms with working HTTP/3 reach. Safari does not implement `WebTransport` yet, so the syncer cannot connect to peers from Safari today — Sialo surfaces a Safari-blocked banner rather than silently failing.

## License

MIT.
