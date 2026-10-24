# RightPass architecture

**Revision:** 0.1, 24 October 2026. **Status:** design, pre-implementation. No code, audit, deployment or benchmark has been produced for this document. Every number marked "target" is a goal to measure against, not a result.

## Contents

1. [Product boundary and principles](#1-product-boundary-and-principles)
2. [Feature set](#2-feature-set)
3. [System overview](#3-system-overview)
4. [Trust boundaries and threat model](#4-trust-boundaries-and-threat-model)
5. [Local vault and data model](#5-local-vault-and-data-model)
6. [Keys, unlock and lock](#6-keys-unlock-and-lock)
7. [Sync on storage you own](#7-sync-on-storage-you-own)
8. [Devices, enrollment and recovery](#8-devices-enrollment-and-recovery)
9. [Native passkeys and autofill](#9-native-passkeys-and-autofill)
10. [Sharing](#10-sharing)
11. [Command line, agents and MCP](#11-command-line-agents-and-mcp)
12. [Health, privacy and diagnostics](#12-health-privacy-and-diagnostics)
13. [Platforms, packaging and updates](#13-platforms-packaging-and-updates)
14. [Shared foundation](#14-shared-foundation)
15. [Security testing](#15-security-testing)
16. [Milestones](#16-milestones)
17. [Non-goals and open questions](#17-non-goals-and-open-questions)
18. [References](#18-references)

## 1. Product boundary and principles

RightPass is a password and passkey manager for people who want the convenience of a modern manager without a hosted vault, a mandatory account or a subscription that can lock them out. Each person owns their vault, their recovery material and, if they sync, their storage account.

Principles that every later section obeys:

1. **No developer service in the credential path.** No account, no hosted vault, no relay, no escrow. A RightPass server may exist one day for optional features; the vault, unlock, sync, export and recovery never depend on it.
2. **Offline always works.** Reading, filling, signing with a passkey, editing and exporting work with no network. Sync and health checks are the only network features and both are optional.
3. **The OS is the trust anchor for unlock, not the UI.** Biometric unlock releases a key through an OS-protected key operation. A successful prompt alone never unlocks anything.
4. **The window is a client, not the boundary.** The desktop window, browser extension and CLI talk to a resident broker that owns the vault handles. Any of them can crash or be tampered with without exposing a key.
5. **Agents use credentials; they never read the vault.** Every agent operation is scoped to an item and a destination, approved by the owner on a trusted surface, one-time, time-limited and receipted.
6. **Secrets are never silently merged.** Two edits to the same password keep both versions and ask. Deletions are explicit, authenticated facts.
7. **Portability is a release requirement.** Standard export formats, documented envelopes, and a restore path that needs nothing from us.
8. **Low footprint by construction.** The resident part is a small native process with no browser engine. The interface is opened on demand.

## 2. Feature set

The list below is the target for version 1 unless marked later. Ideas are drawn from what the best managers do well; the implementations are our own.

| Area | Version 1 | Later |
|---|---|---|
| Item types | Logins, passkeys, TOTP, secure notes, API keys and secret references, SSH keys, attachments | Identities, cards, software licences, document scans |
| Unlock | Windows Hello, Touch ID, Face ID bound key release; master passphrase; lock on sleep, session lock and idle | Apple Watch unlock through the OS policy; hardware security keys as an unlock factor |
| Passkeys | Native provider on Windows 11 and on macOS and iOS; create, sign, list, update, delete; cross-device sync of passkeys | FIDO credential exchange with other providers over CXP |
| Browser | Chrome and Edge extension for passwords, save prompts and generation; Safari through the native AutoFill extension | Firefox extension |
| Desktop apps | Opt-in, focus-verified fill into native app fields by hotkey | — |
| Sync | Your own R2 bucket or S3-compatible store with conditional writes; local-only mode | Self-hosted sync service for shared vaults and push |
| Devices | Enrol a new device from an unlocked one with a short code; revoke devices | — |
| Recovery | Printable recovery kit; clean-device restore with no server | Social recovery: split the recovery secret among people you choose |
| Sharing | Expiring encrypted share links with no server | Shared vaults between RightPass users |
| Agents and scripts | `rp` CLI, secret references, `rp run` and `rp inject`, scoped grants with approval, MCP server, receipts | Service principals for headless machines |
| SSH | Built-in SSH agent serving keys from the vault with per-use approval | Git commit signing |
| Health | Weak, reused and old password report; optional breach check with k-anonymity; "passkey available here" hints | — |
| Import and export | FIDO Credential Exchange Format, CSV, encrypted owner backup | KDBX adapter |
| Travel | Hide selected vaults until unlocked with a separate factor | — |

## 3. System overview

```text
   Windows: plugin authenticator (COM, MSIX identity)      Apple: AutoFill credential-provider extension
                         |                                                   |
   Hub window ------+    |    Browser extension native host --+   rp CLI / MCP server --+
   (on demand)      v    v                                     v                          v
             +---------------------------------------------------------------------------+
             |  Broker: resident native process, no browser engine                        |
             |  owns vault handles, lock state, policy, approvals, sync, SSH agent         |
             |  local transport: user-only socket or pipe, per-caller allowlist,           |
             |  peer-executable and code-signature checks                                  |
             +------+----------------+----------------------+---------------------------+
                    |                |                      |
         native prompt surface   SQLCipher vault      sync worker: ciphertext only
         (overlay window, then   (one per vault        -> your R2 / S3 bucket
          OS key release)         per device)
```

**Broker.** One per user session, written in Rust, with no WebView. It starts at login or on demand: the OS passkey provider, the extension host, the CLI and the hub all connect to it over an authenticated local transport. Locked, it holds no plaintext and no vault key. Target footprint: under 32 MiB locked, under 96 MiB unlocked with ten thousand items.

**Hub.** The full interface (vault browser, editor, settings, conflicts, devices). It is a window the broker opens on demand and tears down on close, so no browser engine stays resident. It renders decrypted fields the owner asked to see and never holds a vault key.

**Prompt surface.** Approvals and unlock prompts are native overlay windows owned by the broker, followed by the OS authentication prompt. A two-button approval never instantiates a browser engine.

**Providers and extensions.** The Windows plugin authenticator, the Apple credential-provider extension, the browser extension's native host, the CLI and the MCP server are thin adapters: they translate a platform request into a typed broker command and render the platform's own UI.

**Apple devices.** The iOS app and the AutoFill extension share one app group container and the same Rust core. The extension answers from the device's local encrypted vault under extension lifecycle limits; it never waits on a cloud merge or on the main app being alive.

## 4. Trust boundaries and threat model

| Component | Trusted to |
|---|---|
| Broker and the OS-bound key adapter | Handle plaintext, verify requests, bind key use to policy |
| OS authentication and credential-provider frameworks | Verify the user and supply request context within their documented contracts |
| Hub, extension popup, CLI output | Show what the owner explicitly asked for; never hold the root key; never authorise by returning "approved" |
| Browser extension native host | A trusted installed component; every page and every page-supplied request stays untrusted |
| Agent, MCP client, script | An authenticated requester, never an approver |
| Storage provider | Availability and ciphertext storage; never decryption and never truth about the latest safe state |
| Signed update channel | A privileged supply-chain boundary, protected and reviewed as such |

**Defended:** disclosure of stored objects, tampering with stored objects, malicious websites and page scripts, replayed approvals, confused-deputy requests from other local apps, accidental plaintext on disk or in logs, lost locked devices, a stolen storage token (ciphertext only; deletion possible, so backups exist).

**Not defended:** a fully compromised operating system, a malicious approved application update, and an agent or browser that has already received an authenticated session for a site. The owner can always export their own data.

## 5. Local vault and data model

One SQLCipher database per vault per device, keyed with a raw 256-bit key derived from the vault root key (no password-based KDF inside the database; the KDF protects the passphrase slot, section 6). Durability is full synchronous for user mutations.

| Table | Contents |
|---|---|
| `vault_meta` | Format, schema version, vault id, active key epoch, local causal state |
| `items` | Random stable id, type, title, account, origins or relying-party id, fields, timestamps, revision |
| `passkeys` | Credential id, RP id, user handle, algorithm, private key, backup flags, authenticator state; one item each |
| `item_versions` | Prior versions and explicit conflict variants |
| `mutations` | Device id plus monotonic sequence, base revision, operation id, change |
| `tombstones` | Authenticated deletion facts with causal metadata |
| `sync_state` | Last accepted remote head, last agreed snapshot, pending publication, retry state |
| `devices` | Approved device identities, key epoch, revocation |
| `shares` | Outgoing share links and shared-vault memberships |
| `audit_events` | Local security receipts with bounded retention; no secret values |

Rules: a mutation and its outbox entry commit in one transaction; a passkey is reported created to the OS only after key material and metadata are durable; sync failure is reported separately from local save success; a wrong key, missing OS key, newer schema, corruption or full disk never leads to initialising a fresh vault over existing data, only to explicit recovery states; migrations keep a verified encrypted backup until the new state is validated.

Non-secret preferences live in a separate settings file and hold references, never tokens or keys.

## 6. Keys, unlock and lock

### 6.1 Key hierarchy

```text
Vault root key (VRK): 256 random bits, generated on the first device

Slots that wrap VRK (each independent, each an AEAD envelope with authenticated metadata):
  passphrase slot   Argon2id(passphrase, salt, params) -> wrapping key
  recovery slot     HKDF(recovery secret)              -> wrapping key
  device slot       OS-protected key operation         -> wrapping key   (one per device)
  member slot       X25519 to a member's public key     -> wrapping key   (shared vaults only)

Subkeys from VRK by HKDF with distinct labels:
  local database key (device-specific context)
  snapshot key-wrapping key
  manifest and metadata authentication keys
  share-link derivation key
```

Passphrase slot parameters start from the RFC 9106 memory-constrained recommendation (64 MiB, three passes) and are benchmarked and fixed per platform. Parameters and salts are stored with the slot and validated before memory is allocated. The recovery secret is high-entropy random material, printed in the recovery kit; it is never a user-chosen phrase.

Each uploaded snapshot uses a fresh random data-encryption key wrapped under the snapshot key-wrapping key. Revision, format and vault identity are bound into the ciphertext as associated data.

### 6.2 OS-bound unlock

| Platform | Device slot mechanism |
|---|---|
| macOS, iOS | A Secure Enclave P-256 key with an access-control policy of current biometric set (or user presence as a weaker opt-in). The device slot is wrapped with ECIES under that key; decryption happens through the Secure Enclave. A biometric re-enrolment invalidates the key by design and the user falls back to the passphrase. |
| Windows | A Windows Hello key credential. The broker asks the credential to sign a fixed per-vault challenge; the signature is user-verification gated by Windows and is deterministic, so a key derived from it by HKDF wraps the device slot. Behaviour under Remote Desktop and after Hello re-enrolment is qualified explicitly. |
| Fallback | Passphrase slot only. A successful OS prompt followed by an unrestricted secret read is not an unlock mechanism and is not implemented. |

The credential-provider extension on iOS never runs the passphrase KDF: it unlocks through the Secure Enclave slot or tells the user to open the app.

### 6.3 Lock policy

Lock on OS session lock, on sleep, on an idle interval, and on command. Lock cancels privileged work, retires unlocked database connections, releases decrypted views and zeroes keys as far as the platform allows. The sync worker may keep already-encrypted outgoing blobs and the storage credential while locked; it cannot apply new remote revisions until the next authorised unlock. Agent use and sensitive actions can require fresh authentication regardless of lock state. Clipboard copies are explicit, clear only the value RightPass placed, and request exclusion from clipboard history and cloud clipboard where the platform offers it.

## 7. Sync on storage you own

### 7.1 Providers

Version 1 syncs directly to a private Cloudflare R2 bucket using S3-compatible requests signed on the device. Any S3-compatible store that documents atomic conditional writes (`If-Match`, `If-None-Match`) can implement the same provider contract. Stores without conditional writes are not supported without a mediating server, because a vault cannot rely on a race window. Local-only mode needs no storage at all.

Setup takes an endpoint, bucket and a bucket-scoped access key. RightPass refuses plain HTTP, displays the endpoint for confirmation, never follows a redirect with signed credentials, and tests the configuration by writing and deleting a harmless marker under a random prefix. Token values never reach logs, diagnostics, command arguments or agent context.

### 7.2 Object layout

```text
rightpass/v1/<opaque-vault-id>/
  head                        authenticated encrypted current head; small bounded outer header
  keys/<epoch>/<object-id>    encrypted key slots (passphrase, recovery, member); KDF parameters in the outer framing
  snapshots/<revision-id>     immutable encrypted logical snapshot
  attachments/<object-id>     separately encrypted immutable attachments
  shares/<share-id>           expiring encrypted share-link payloads (section 10)
```

The head's outer header carries format, vault id, key epoch and the exact references of the key slots a new device needs, so bootstrap never requires the root key to find its own recovery slot. Everything else, including item titles, origins and counts, is inside ciphertext. The provider still sees object names, sizes, timestamps and access patterns.

### 7.3 Publication

1. Read and authenticate the current head; validate vault id, format and epoch against local history.
2. Build the next logical snapshot inside one local transaction; encrypt before any network call.
3. Upload immutable objects with create-only semantics; verify they exist.
4. Write the new head with `If-Match: <previous ETag>` (first write: `If-None-Match: *`). Never an unconditional replace.
5. On `412 Precondition Failed`, fetch the winning head, reconcile against the known base, publish a new candidate; bounded retries, then a visible conflict with nothing lost.
6. On a timeout with an uncertain write, read back and compare operation ids before retrying. A retry is never allowed to become an unconditional write.
7. Mark the local outbox committed only after publication evidence. An unreferenced uploaded blob is an orphan, never a lost vault.

### 7.4 Conflict rules

No wall-clock last-writer-wins for secrets. Each device has an identity and a monotonic sequence; each edit carries a base revision.

| Concurrent change | Behaviour |
|---|---|
| Different items | Merge both |
| Same item, disjoint non-secret fields | Merge where the schema proves independence |
| Same password, OTP secret or private key | Keep both versions and ask |
| Delete versus edit | Keep an explicit conflict and the authenticated tombstone; never resurrect because an object is absent |
| Same passkey | Credential id, RP id, user handle and private key move as one unit; never mix parts |
| A device returns after history compaction | Full reconciliation with a preserved local branch; never blind upload of an old snapshot |

### 7.5 Rollback and deletion safety

Devices keep the last accepted revision and refuse a remote head that moves them backwards without an explicit restore. A new device with no checkpoint cannot distinguish every replayed old vault; the recovery kit's checkpoint and enrolment from a trusted device close that gap. Missing remote objects never delete local data; failures are typed (`RemoteUnavailable`, `RemoteMissing`, `AuthenticationFailed`, `Conflict`, `RecoveryRequired`). Remote pruning is explicit and conservative; automatic garbage collection arrives only with a tested reachability protocol that handles offline devices.

### 7.6 Budget

At about 5 KB per item, one thousand items are about 5 MB; the live vault plus twenty retained snapshots is roughly 105 MB before attachments. R2's free allowance (10 GB-month of storage and millions of operations) is the user's and is shared with their other workloads; these are checked prices, not promises. Sync runs on local change, on unlock and foreground, and on a polling interval; mobile platforms sync when the OS allows and never promise instant delivery to a suspended phone.

## 8. Devices, enrollment and recovery

**Enrolment.** An unlocked device approves a new one over a mutually authenticated transfer: ephemeral X25519 keys, a commit-and-reveal short authenticated string shown on both screens and typed into the other device, transcript binding of device identities, vault id and protocol version, rate limits and expiry. The user's own bucket, or a direct local connection, carries the opaque messages; no relay of ours exists. Storage credentials are separate from vault keys; prefer a bucket-scoped key per device so one can be revoked alone.

**Recovery kit.** Vault id, storage endpoint and bucket, the recovery secret, the format version and the restore steps, printed and stored apart from the vault and the storage key. Restoring on a clean device from the kit alone is tested before any real credential enters a vault. There is no "forgot password" service.

**Social recovery (later).** The recovery secret can be split with Shamir's secret sharing into shares (for example two of three) handed to people the owner chooses. Shares are useless alone, reveal nothing about the vault, and recombine only on the owner's device. This gives emergency access with no escrow and no server.

**Rotation.** Changing the passphrase rewraps the root key. Compromise of the root key means a new key epoch and re-encryption; old copies held by an attacker are not unlearned. Removing a device revokes its storage credential and enrolment rights but cannot erase what that device already copied. Compromised passkeys are also revoked at the relying party.

## 9. Native passkeys and autofill

### 9.1 Windows 11

RightPass registers as a plugin passkey manager: a packaged app with identity (MSIX), a COM server implementing the plugin authenticator interface, enabled by the user under Settings, Accounts, Passkeys, Advanced options. Windows activates the server for creation and assertion requests from any browser or app that uses the OS WebAuthn API, with the desktop window closed and the browser extension disabled. The server routes requests into the broker; user verification is Windows Hello; the broker signs only after the device slot or passphrase has unlocked the vault. Cancellation, timeouts, discoverable-credential selection and credential updates are supported.

### 9.2 macOS and iOS

An AuthenticationServices credential-provider extension handles passwords and passkeys for Safari, system web views and apps that use the OS APIs. Credential metadata may be offered to the OS for matching, with the privacy implications documented. Newly created passkeys are durable before success is reported, then synced. A vault not yet present on the device is reported as unavailable, never as "no credential".

Embedded web views in other apps only offer passkeys for their own associated domains or when the app is an entitled browser; RightPass documents that limitation rather than working around it.

### 9.3 Browsers

A thin Chrome and Edge extension handles password matching, save prompts, generation and in-page UX over native messaging to the broker, with the extension id pinned and every message validated. The broker, not the extension's service worker, owns vault data and lock state: a hung popup never blocks the desktop interface or the OS passkey provider. Matching is explicit: normalised scheme, host and port, public-suffix boundaries, frame identity and user gesture. No automatic fill into hidden, insecure or cross-origin fields on a lookalike name. Passkeys in browsers go through the OS provider; no `navigator.credentials` patching and no trusting a page's claimed RP id.

### 9.4 Desktop applications

Fill into native app fields is opt-in, bound to a hotkey, and verified against the focused window and process before typing. It is presented as a convenience, never as system-wide autofill.

### 9.5 Authenticator core

The passkey authenticator is implemented over the open `passkey-rs` libraries with a credential store backed by the unlocked vault. Registration binds RP id, credential id, user handle, public-key parameters, private key and properties atomically; assertion binds the platform-validated request context, client-data hash, RP and selected credential. User-presence and user-verification flags are always truthful. Private keys leave the authenticator boundary only inside authenticated ciphertext (sync, backup, export).

## 10. Sharing

### 10.1 Share links: one item, one recipient, no server

`rp share` (or Share in the hub) encrypts a copy of an item under a fresh random key, uploads it as an expiring object under `shares/<id>` in the owner's own bucket, and produces a link made of a presigned, time-limited URL to that object plus the decryption key in the URL fragment. The fragment never reaches the storage provider. Options: expiry (default 24 hours, maximum 7 days), view limit (best effort), and a passcode the sender gives the recipient another way, which derives a second key layer.

The recipient opens the link in RightPass (any platform) or in a static viewer page served from a public static host; the page contains no network calls except the presigned GET and performs decryption locally. Revocation deletes the object; a link already opened cannot un-share what the recipient saw. The hub shows every live share and lets the owner revoke it.

### 10.2 Shared vaults: ongoing sharing with people you trust

A shared vault is a second vault with its own root key, synced with the same protocol. Membership is a set of member slots: the vault's root key wrapped to each member's public key, with the key exchanged by the same short-code ceremony used for device enrolment (in person or over a call). Roles: owner, editor, viewer, enforced by signing keys for edits and verified by every member; a viewer's edit is rejected by everyone else.

Storage options, honestly stated:

- **One bucket per shared vault**, with a scoped access key per member. R2 and most S3 stores scope keys to a bucket, not a prefix, so this is the only zero-server layout that keeps members out of each other's private vaults. It works well for a household or a two-person team.
- **A self-hosted sync service** (later) for larger groups, per-member authorisation, revocation without rotating storage keys, and push. It stores ciphertext only. It is optional and user-deployed; RightPass never requires one.

Removing a member rotates the shared vault's key epoch and re-encrypts; what the member already saw stays seen. Passkeys in shared vaults follow the same credential-unit rules as everywhere else.

## 11. Command line, agents and MCP

### 11.1 Two kinds of caller

**The owner** at a terminal: full control after OS authentication, including reading a secret when they ask for it.

**An agent or script**: a coding agent, a build job, an automation. It can cause credentials to be used; it cannot enumerate or read the vault. Every use is a scoped, one-time, time-limited grant that the owner approves on the broker's native prompt surface, or that falls inside a standing grant the owner created in advance for that exact item and caller.

### 11.2 The `rp` command

```text
rp status                              lock state, devices, sync state, pending approvals
rp unlock | rp lock
rp item list [--vault V] [--tag T]     redacted listing (titles and accounts, never secrets)
rp item get ITEM [--field F] [--reveal] owner only; --reveal requires fresh OS authentication
rp item create|edit|rm
rp generate [--words N | --length N --symbols]
rp totp ITEM                           one-time code; owner, or agent under a grant
rp read rp://VAULT/ITEM/FIELD          secret reference; owner, or agent under a grant
rp run [--env-file F] -- CMD ARGS      resolve rp:// references in the environment and run CMD with them injected
rp inject -i template -o out           resolve rp:// references in a template file
rp ssh agent                           serve vault SSH keys over the agent protocol with per-use approval
rp grant create --item I --fields F --caller C --ttl 1h [--uses N] [--origin O]
rp grant list | rp grant revoke ID
rp approve ID | rp deny ID             act on a pending agent request from the terminal
rp receipt ID                          redacted authenticated outcome of an agent operation
rp share ITEM [--expires 24h] [--passcode] | rp share list | rp share revoke ID
rp sync now | rp sync status | rp sync conflicts
rp device list | rp device enrol | rp device revoke
rp import --format cxf|csv FILE | rp export --format cxf|backup FILE
rp mcp serve                           MCP server over stdio for the current owner session
```

Secret references have the form `rp://vault/item/field` and may appear in environment variable values, `.env`-style files and templates. `rp run` resolves them in the broker and injects the values into the child process environment or a file descriptor; nothing is printed. For an agent caller, resolution needs a grant for each referenced item; the agent process that invoked `rp run` can still observe the child it spawned, which is why grants are per item, per caller and time-limited, and why the receipt names the command that ran.

### 11.3 Grants and approvals

A grant binds: the caller identity (authenticated local process, code-signature verified where the platform allows), the item and fields, the operation (fill, sign, read, totp, run), the destination (origin or relying-party id for web use, process for `run`), a nonce, an expiry and a use count. The native prompt shows exactly those things. Cancellation, lock, restart, destination navigation and session replacement invalidate pending grants. A model's text, a tool argument or a web page can never approve a request. Receipts are kept locally without the secret and without a reusable token.

Standing grants exist for the boring cases (a build script that needs the same API key every day): still per item, per caller, with a TTL the owner chooses, visible and revocable in the hub, and never covering passkey private keys.

### 11.4 MCP server

`rp mcp serve` exposes typed tools to a local MCP client under the same gate as the CLI:

| Tool | Effect |
|---|---|
| `rightpass.find` | Redacted search by site or title; returns item ids and account names, never secrets |
| `rightpass.request_use` | Ask to fill, sign in, or resolve a reference for a named destination; returns a pending approval id |
| `rightpass.status` | Pending, approved, executed, denied, expired |
| `rightpass.cancel` | Cancel a pending request |
| `rightpass.receipt` | Redacted outcome |
| `rightpass.totp` | A one-time code for an item under a grant |

There is no `read_all`, no `dump`, no `sign(hash)`, no clipboard tool, no cookie export. Signing into a site through a browser the agent controls gives that browser a session; the receipt says so.

### 11.5 Headless machines (later)

A service principal is a device identity with no interactive user: its own device slot, an allowlist of items, and grants created on an interactive device. It gets exactly those items and nothing else, and it can be revoked like any device.

## 12. Health, privacy and diagnostics

**Health.** A local report of weak, reused and old passwords, missing 2FA where the site is known to offer it, and sites known to support passkeys. Breach checking uses a k-anonymity range query (only a hash prefix leaves the device) and is opt-in. No vault content leaves the device for any health feature.

**Telemetry.** None. Diagnostics are local, contain allowlisted event names, error classes, durations and opaque ids, and are reviewed by the owner before export. Item titles, site names and usernames are treated as sensitive, not only passwords. Support bundles never include the database, memory dumps, storage credentials or key material.

**Crash handling.** A panic in the broker produces a redacted report with no memory dump of a secret-bearing process. Native dumps are off for the broker.

## 13. Platforms, packaging and updates

| Platform | Baseline | Packaging |
|---|---|---|
| Windows | Windows 11 with the November 2025 update or later (plugin passkey manager support became generally available with it; the exact minimum build is an open question) | MSIX with package identity; Authenticode signed |
| macOS | macOS 14 or later | Signed, notarized app with the credential-provider extension; broker as a login item |
| iOS | iOS 17 or later | App plus AutoFill extension, App Store |
| Linux | Core, broker and CLI build and are tested; no graphical hub in version 1 | tar.gz; distro packaging later |

Updates are signed and verified before activation. A failed update must leave the previous version working; where the platform owns updates (MSIX, App Store) that guarantee comes from the platform, elsewhere the updater keeps the previous build and rolls back when the new one does not report a healthy start.

## 14. Shared foundation

RightPass is built on the published `rightkit-*` crates that Orthic Labs uses across its desktop apps, so that storage safety, OS secret stores, local transport, logging and release discipline are shared rather than reinvented. The main ones and their roles:

| Crate | Role in RightPass |
|---|---|
| `rightkit-db` | SQLCipher bootstrap, durability profiles, migrations with verified backups, atomic restore |
| `rightkit-fs` | Atomic writes, owner-only files, cross-process locks, state directories |
| `rightkit-credentials` | OS secret stores (Keychain, Credential Manager, Secret Service) and the OS presence prompt; the OS-bound key release is added here |
| `rightkit-daemon` | The broker's user-only transport, per-caller allowlists, peer-executable verification, device link |
| `rightkit-identity` | Installation identity, clone detection, the pairing ceremony and replay protection |
| `rightkit-sync` | AEAD and key-derivation primitives and the wire conventions; the conditional-object adapter and conflict-preserving policy are added for the vault |
| `rightkit-http` | Signed HTTPS with timeouts, bounded bodies, redirect policy and secret redaction |
| `rightkit-logs`, `rightkit-crash` | Redacted local diagnostics and crash capture |
| `rightkit-hotkeys` | Global shortcuts for summon, lock and desktop fill |
| `rightkit-agent-tools`, `rightkit-mcp` | The typed, gated tool surface and MCP host |
| `rightkit-overlay`, `rightkit-overlay-win` | The native prompt surface |

Capabilities that do not exist yet in the shared crates and are built there first: OS-bound key release, power and session events for lock-on-sleep, the conditional-object sync adapter, the passkey authenticator adapter, and the Apple credential-provider product.

## 15. Security testing

Synthetic credentials, isolated OS-store namespaces and test buckets only; the real vault is never a fixture; no QA control surface exists in release builds.

| Area | Must be proven |
|---|---|
| Durability | Create, close every process, reopen offline, recover exactly; crash and power-loss injection at every save, migration and rotation step |
| Unlock | Wrong passphrase, cancelled prompt, unavailable verification and changed biometric enrolment each produce the intended state; a direct broker call or a tampered hub cannot obtain the key |
| Lock | Session lock, sleep and idle invalidate handles and grants, including a prompt that succeeds late |
| Providers | Windows passkey create and sign with the hub closed and the extension disabled; Apple extension create and sign with the containing app suspended; durability before success |
| Browser | Wrong origin, sibling domain, untrusted frame, stale navigation and forged caller context are refused |
| Sync | Real conditional-write races have one winner and no loss; delete-versus-edit and long-offline reconnection do not resurrect; tampered or rolled-back objects are rejected; deleting remote objects or revoking the storage key never wipes local data; clean-device restore from kit and backup alone |
| Agents | Model-supplied approval, replayed ids and cross-session reuse fail; navigation after approval invalidates; transcripts and logs contain no credentials |
| Sharing | Expired and revoked links fail closed; a viewer cannot edit a shared vault; member removal rotates the epoch |
| Hygiene | No plaintext secret in objects, ledger, temp files, journals or logs (grep for synthetic values); no secret in argv or environment; no public bucket or wildcard token; bounded parsing of hostile imports and KDF parameters |
| Release | No debug port, test provider, memory backend or approval bypass in the signed artefact; public clean-room build resolves published packages only |

Independent review of the key envelope, the OS-bound key adapters, the provider boundaries, the agent authorisation and the sync protocol precedes general availability. Tests are evidence, not a claim of invulnerability.

## 16. Milestones

| Gate | Deliverable | Exit |
|---|---|---|
| 0 | Shared-crate reconciliation, version pins, this document's open questions closed | Every owner has a published version or a named task |
| 1 | Footprint and activation spike: broker locked and unlocked footprint on both OSes, native prompt from a cold broker, Windows provider activation with the hub closed | Numbers recorded; activation works without the hub |
| 2 | Native provider and key-release prototypes against synthetic vaults: Windows plugin authenticator, Apple extension, Secure Enclave and Windows Hello slots | Provider and unlock tests pass on real hardware |
| 3 | Vault, recovery and lock: SQLCipher vault, slots, recovery kit, clean-device restore, lock on sleep and session lock | Durability and recovery tests pass; format and vectors documented |
| 4 | Durable credential use: providers on the real vault, browser extension, Apple container and extension, MSIX packaging | Provider suite passes on installed builds |
| 5 | Sync: conditional-object adapter, conflict preservation, rollback protection, enrolment, backups | Sync suite passes against a real bucket |
| 6 | CLI, grants, MCP, SSH agent, share links | Agent suite passes with adversarial prompts |
| 7 | Import and export, health, hub polish, dependency and licence review, independent review, public release | Installed end-to-end journeys recorded for the released artefact |

## 17. Non-goals and open questions

**Non-goals:** a hosted vault, a mandatory account, organisation billing, cloud recovery escrow, cloud language-model access to vault content, a universal autofill claim for every desktop app, hidden telemetry, and any feature that makes offline access depend on Orthic Labs continuing to operate anything.

**Open questions:** the exact Argon2id parameters per platform after benchmarking; whether share-link viewing on the web needs a hosted static page or stays app-only at first; the self-hosted sync service's authentication model for shared vaults; the minimum Windows build for the plugin passkey provider in general availability; the licence (MIT OR Apache-2.0 proposed).

## 18. References

- Cloudflare R2: S3 API compatibility and conditional writes, consistency model, API tokens, pricing. https://developers.cloudflare.com/r2/
- Microsoft: plugin passkey manager support on Windows 11. https://learn.microsoft.com/en-us/windows/apps/develop/security/third-party
- Apple: credential provider extensions (AuthenticationServices), Secure Enclave and access control for Keychain items. https://developer.apple.com/documentation/authenticationservices
- FIDO Alliance: Credential Exchange Format and Credential Exchange Protocol. https://fidoalliance.org/specs/cx/
- RFC 9106: Argon2. https://www.rfc-editor.org/info/rfc9106/
- SQLCipher design. https://www.zetetic.net/sqlcipher/design/
- passkey-rs (MIT or Apache-2.0). https://github.com/1Password/passkey-rs
- Chrome extensions: native messaging. https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging
- Have I Been Pwned: k-anonymity range API. https://haveibeenpwned.com/API/v3#SearchingPwnedPasswordsByRange
