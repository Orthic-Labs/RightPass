# RightPass

Local-first passwords and passkeys. Native OS integration. Your own storage. Agents only with your consent.

RightPass is a password and passkey manager that keeps the vault on your devices, syncs through storage you own (a private Cloudflare R2 bucket, any S3-compatible store with conditional writes, or nothing at all), unlocks with Windows Hello, Touch ID and Face ID, and acts as a native passkey provider on Windows 11 and Apple platforms. It has a command-line interface and an MCP server so coding agents and scripts can use credentials, but never read the vault.

**Status:** design stage. This repository holds the architecture; there is no application code yet. Nothing here has been audited.

- [ARCHITECTURE.md](ARCHITECTURE.md): the full design, from the key hierarchy to the agent surface.
- Maintained by Orthic Labs. Built on the published `rightkit-*` crates that the rest of the Right Suite shares.
- Licence: to be decided before the first code lands. The working proposal is MIT OR Apache-2.0, matching the shared crates.

## What makes it different

| | RightPass |
|---|---|
| Where the vault lives | On your devices, encrypted with keys only you hold |
| Where it syncs | Your own R2 bucket or S3-compatible store; or local only |
| Account required | None. No RightPass server is in the data path |
| Unlock | Windows Hello, Touch ID, Face ID bound to an OS-protected key; master passphrase; recovery kit |
| Passkeys | Native provider on Windows 11 and Apple platforms, so any browser and app that asks the OS can use them |
| Agents and scripts | `rp` CLI with secret references and `rp run`; MCP tools; every use approved, scoped, one-time and receipted |
| Sharing | Expiring encrypted share links with no server, and shared vaults for people you trust |
| Recovery | A recovery kit you print, and optional social recovery by splitting the recovery secret among people you choose |
| Portability | FIDO Credential Exchange Format import and export, CSV, optional KDBX |

## Contributing

Not yet. Read the architecture, open an issue if you find a flaw, and watch for the first milestone.
