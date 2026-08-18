# Agent Guidelines

This file provides guidance to coding agents when working with code in this
repository.

## Overview

TypeScript library implementing the [
`Ed25519VerificationKey2020`](https://www.w3.org/community/reports/credentials/CG-FINAL-di-eddsa-2020-20220724/)
spec for use with Linked Data Proofs / Verifiable Credentials. Forked from
`digitalbazaar/ed25519-verification-key-2020` to add TypeScript support.
Published as `@interop/ed25519-verification-key`; the main class is
`Ed25519VerificationKey` (the `2020` suffix was dropped as part of the v6.1 rename
since the class now supports multiple serialization formats).

Intended for use with `@digitalcredentials/ed25519-signature-2020` and
`@digitalcredentials/vc`.

## Toolchain & Project Layout

### Package Manager

Use `pnpm` (not `npm` or `yarn`). The lockfile is `pnpm-lock.yaml`. Install deps
with `pnpm install`; run scripts with `pnpm run <script>` or `pnpm <script>`.

### Build

The library is built with `tsc` (not `vite build`). `vite.config.ts` exists only
to configure Vitest and to run `vite dev` as a server for Playwright. Running
`pnpm run build` compiles `src/` to `dist/` via `tsconfig.json`.

### Two tsconfigs

- `tsconfig.json` — library build only; includes `src/**/*`
- `tsconfig.dev.json` — extends the above with `noEmit: true`; adds `test/**/*`,
  `vite.config.ts`, and `playwright.config.ts` so ESLint's type-aware rules
  cover all files

Do not add test files to `tsconfig.json` — they would be emitted into `dist/`.

### Tests

- `test/node/` — Vitest unit tests (`pnpm run test-node`); run in Node
- `test/browser/` — Playwright tests (`pnpm run test-browser`); run in real
  Chromium via a Vite dev server (`pnpm run dev`)

The `dev` script exists solely to give Playwright a server that can serve and
transform TypeScript source files on the fly. There is no browser app.

The Vite dev server is pinned to port **5373** (`server.port` with
`strictPort: true` in `vite.config.ts`), and `playwright.config.ts` points
`baseURL`/`webServer.url` at the same port. This avoids colliding with other
Vite projects on the default 5173 -- with `reuseExistingServer` enabled,
Playwright would otherwise latch onto a foreign dev server and the browser
import of `/src/index.ts` would fail. If you change the port, update both files.

### ESM & import paths

The package is ESM-only (`"type": "module"`). Local imports must use the `.js`
extension even though source files are `.ts` — e.g.
`import { Example } from '../../src/index.js'`. TypeScript's
`moduleResolution: Bundler` resolves these to the `.ts` source at compile time.

## Conventions

Code style, refactoring, JSDoc, comment, and error-handling conventions live in
@CONTRIBUTING.md -- follow them.

## Architecture

### Platform-specific crypto (`src/ed25519*.ts`)

The library has three crypto backends selected via `package.json` field
overrides:

| File                         | Environment  | Crypto backend                                 |
|------------------------------|--------------|------------------------------------------------|
| `src/ed25519.ts`             | Node.js      | Node's built-in `crypto` module                |
| `src/ed25519-browser.ts`     | Browser      | `@noble/ed25519` + `crypto.subtle` (WebCrypto) |
| `src/ed25519-reactnative.ts` | React Native | `@noble/ed25519` + `@noble/hashes` (pure JS)   |

The `browser` and `react-native` fields in `package.json` remap
`./dist/ed25519.js` to the appropriate backend at bundle time. All three expose
the same interface:
`{ generateKeyPair, generateKeyPairFromSeed, sign, verify, sha256digest }`.

`@noble/ed25519` v3 requires explicit sha512 configuration before use. The
browser backend wires it to `crypto.subtle`; the React Native backend wires it
to `@noble/hashes` (pure JS, no WebCrypto needed). Both noble backends use the
v3 async API (`signAsync`, `verifyAsync`, `getPublicKeyAsync`).

### Utility modules

- `src/baseX.ts` — exports `base58btc` and `base64url` codecs (via
  `@scure/base`). `base64url` is `base64urlnopad` (RFC 4648, unpadded) for JWK
  interop; an earlier `base-x` codec was a big-number radix conversion that
  produced non-standard base64.
- `src/validators.ts` — exports `assertKeyBytes` for validating key byte
  length and type

### Key encoding

Keys are stored as multibase base58-btc strings with multicodec varint headers:

- Public key: `[0xed, 0x01]` prefix → `publicKeyMultibase` (starts with `z`)
- Private key: `[0x80, 0x26]` prefix → `privateKeyMultibase`

The private key buffer internally stores `<32-byte seed><32-byte public key>` (
64 bytes total). When exporting to JWK (`toJwk`), only the first 32 bytes are
used as `d`.

### Serialization formats (Multikey is the default)

This class is a **superset that reads and writes the `Multikey` format** in
addition to the legacy 2020/2018/JWK formats. The goal is for downstream
libraries (notably `@digitalcredentials/ed25519-signature-2020`) to depend on
*this* library alone and drop their separate `@digitalcredentials/ed25519-multikey`
dependency.

Import (`from()`) dispatches on `options.type`:

| `type`                        | handler                            |
|-------------------------------|------------------------------------|
| `Multikey`                    | `fromMultikey()`                   |
| `Ed25519VerificationKey2018`  | `fromEd25519VerificationKey2018()` |
| `JsonWebKey2020`              | `fromJsonWebKey2020()`             |
| (default / 2020)              | constructor                        |

Export — there is a `to<Format>()` family plus a Multikey-default `export()`:

| Method                            | Output                                              |
|-----------------------------------|-----------------------------------------------------|
| `export()`                        | **Multikey** (`type: 'Multikey'`, multikey context) |
| `toVerificationKey2020()`         | `Ed25519VerificationKey2020`                         |
| `toJwk()`                         | JWK                                                  |

`export()` returns Multikey and uses Multikey field naming (`secretKey` option →
`secretKeyMultibase`). `toVerificationKey2020()` keeps 2020 naming (`privateKey`
option → `privateKeyMultibase`).

**Secret-key byte-length gotcha (the one hard part).** Both formats share the
same multicodec headers (`0xed01` pub, `0x8026` priv); only the secret payload
length differs:

- 2020 `privateKeyMultibase` is **64 bytes** (`seed‖publicKey`); the sign path
  asserts exactly 64.
- Multikey `secretKeyMultibase` is **64 bytes by default** (`seed‖pub`, the
  legacy form); the canonical 32-byte seed is opt-in. On import, both lengths
  are accepted.

So the conversion is lossless in both directions and is handled at the
boundaries:

- **Multikey → 2020 (`fromMultikey`):** a 32-byte secret is re-concatenated with
  the 32-byte public key to rebuild the 64-byte buffer, then re-encoded under
  `0x8026` into `privateKeyMultibase`. A 64-byte legacy secret passes through.
- **2020 → Multikey (`export`):** by default the full 64-byte `seed‖pub` buffer
  is emitted (matching `@digitalbazaar/ed25519-multikey`); pass
  `export({canonicalize: true})` to emit the canonical 32-byte seed instead.

**Parity with `@digitalbazaar/ed25519-multikey`:** our `export()` defaults to the
64-byte legacy secret form, same as that library; the canonical 32-byte form is
opt-in via `export({canonicalize: true})`. Both lengths are valid Multikey and
share the `0x8026` header; public-key and signature bytes are identical across
the two libraries (verified by the known-answer fixtures in
`test/node/mock-data.ts`).

*Not yet implemented:* that library's `export()` also takes a `raw` option,
which returns the key material as `Uint8Array`s (`{publicKey, secretKey}`)
instead of multibase strings -- intended for binary-capable stores (e.g. Mongo
BSON, `BYTEA`) that can skip base58 encoding overhead. `raw` and `canonicalize`
are independent there. We've deferred adding `raw` until we have a concrete need
(current usage serializes keys as JSON text on disk, which can't hold binary
anyway). If added, it should return copies on both the public and secret paths.

### Refactoring downstream `ed25519-signature-2020`

When that (still plain-JS) repo migrates to depend on this library alone:

1. Replace both `Ed25519Multikey.from(...)` calls in
   `lib/Ed25519Signature2020.js` (`verifySignature`, `getVerificationMethod`)
   with `Ed25519VerificationKey.from(...)` — `from()` now ingests
   `type: 'Multikey'`, which was the entire reason for the second dependency.
2. **Audit every `.export()` call site** — `export()` now returns Multikey, not
   2020. Any flow that expects a 2020 verification method must switch to
   `toVerificationKey2020(...)`.
3. Drop `@digitalcredentials/ed25519-multikey` from its dependencies.

### Class hierarchy

`Ed25519VerificationKey` extends `KeyPair` from
`@digitalcredentials/keypair`. Interfaces (`ISigner`, `IVerifier`,
`IVerificationKeyPair2020`, `IMultikeyPair`, etc.) come from
`@digitalcredentials/ssi` — do not redefine them.
