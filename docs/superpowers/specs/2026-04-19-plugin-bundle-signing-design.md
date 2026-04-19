# Plugin bundle + signing — sub-spec design

**Status:** V0 design — drafted 2026-04-19.
**Authored:** 2026-04-19 (Gavin + Cowork).
**Supersedes:** `docs/brainstorm/early-spec.md` treatment of skills and tools where it conflicts with MCP-as-skill-interface (P3) and default-deny capability grants (P7). The brainstorm's "tools as action agents" framing is deferred to sub-spec 4 (Skill process isolation), which will reconcile it with MCP skills.
**Resolves:** `design-principles.md` open question 2 (signing authority).

## Citations

- **Map:** [`docs/superpowers/specs/2026-04-17-v1-spec-map-design.md`](./2026-04-17-v1-spec-map-design.md)
- **Principles load-bearing:** P3 (primary — signed, capability-declared skills), P7 (install-time default-deny); P8 (every install event is audited)
- **Use cases cited:** UC3 (primary — IT-signed distribution, SOC-approvable audit chain), UC1 (solo-founder reuse of trusted skills across client contexts)
- **Tier:** T1 (no substrate dependencies at design time; at install time consumes Supervisor + Audit log)
- **Upstream:** none at design time
- **Downstream consumers:** Supervisor (T2 — bundle install approval flow, manifest registry), Broker + provenance (T3 — verified-manifest objects for capability matrix), Skill process isolation (T4 — runtime enforcement of declared manifest), Audit log (T1 — every install-event's payload shape is defined here)

## Purpose

Define the **install unit** of Mothership: a signed archive that declares everything the supervisor and broker need to sandbox, authorize, and audit extensions. Enforce P3's invariant: **an attacker with a fresh, unverified identity cannot publish a skill that reads user secrets on a default-configured Mothership; a signed skill cannot exceed its declared manifest at runtime.**

Plugin bundles are the only way to install skills, agents, and hooks into a Mothership. Configuration-editing or manual file-dropping to install is explicitly not a supported pathway in V1.

---

## Scope

### In V1

- **Bundle format** — tar+gzip archive with a fixed directory layout, content-hash manifest, and detached publisher signature.
- **Manifest schemas** — one per artifact kind: bundle, skill, agent, hook. Versioned, strict schema.
- **Signing model** — operator-local trust root with optional sigstore (Rekor transparency log) augmentation. No Mothership-operated CA in V1.
- **Publisher identity** — X.509 certs issued under an operator-chosen trust root; sigstore-keyless OIDC certs accepted when Rekor entry is present.
- **Install-time verification** — six-state journaled state machine (`received → descriptor-verified → reviewed → staged → committed → activated`), atomic, all-or-nothing, audited.
- **Revocation** — local revocation store managed by operator; optional CRL/OCSP endpoint polling per trust root.
- **Capability declarations** — skills and agents declare exact capability requirements; no implicit capabilities.
- **Install-event audit shapes** — bundle install events (requested / approved / rejected / signature-verified / signature-failed) carry structured payloads declared here for the Audit log's event-type registry.

### Out of V1 (explicit non-goals)

- **Mothership-operated CA.** Creates a central dependency and single point of compromise; contradicts the "operator-owned" posture. Deferred indefinitely.
- **Skill marketplace / discovery service.** Operators install via URL, path, or out-of-band channels. A discovery catalog is a V2+ product-surface consideration.
- **In-place skill updates with trust-preservation.** V1 re-installs are full bundle replacements requiring fresh signature verification. Delta updates are a V1.5+ optimization with its own threat model.
- **Claude Code plugin compatibility shim.** `.claude-plugin/plugin.json` wrapping is structurally possible (the bundle format is a superset) but the `mship-wrap-cc-plugin` tooling ships in V1.5+. Until then, CC plugins do not auto-install on Mothership; operators hand-author a Mothership manifest that wraps them.
- **SBOM generation or attestation binding.** Publishers may ship SBOMs inside the bundle as documentation, but the runtime does not enforce SBOM presence or validity in V1.

---

## Bundle format

### Architecture

A plugin bundle is a `.mship` file — a gzipped tar archive — that contains:

1. A **canonical JSON descriptor** (`descriptor.json`) that is the authoritative, signed representation of everything in the bundle.
2. A **DSSE envelope** (`signature.dsse`) over the canonical JSON descriptor, with embedded publisher cert chain and optional Rekor inclusion proof.
3. **Payload files** whose hash, size, mode, and path are all pre-declared in the descriptor. Payloads are never consumed as independent sources of truth — only as byte-addressable content whose identity is checked against the verified descriptor.

This inverts the earlier checksums-file approach: the descriptor is the one thing signed, and every payload file is validated against fields in the verified descriptor before any side effect occurs on the host system.

### Archive layout

```
descriptor.json                 # canonical JSON; the signed subject
signature.dsse                  # DSSE envelope (JSON): payload_type + signatures + cert chain + optional Rekor bundle
payloads/
  skills/<name>/...             # MCP server binary/scripts/assets
  agents/<name>/...             # agent binary/scripts (if bundle ships it)
  hooks/<name>/...              # hook script
```

Manifest YAMLs are an **authoring-side convenience only** — operators edit YAML, but the `mship-bundle-tool build` step converts them to canonical JSON which becomes part of `descriptor.json`. YAML does not appear in the installable bundle. This eliminates YAML parser divergence across Rust/TS/Python as a correctness input.

### `descriptor.json` contents

Canonical-JSON-serialized (RFC 8785 / JCS — same library used for audit log entries, same golden-vector parity requirement):

```json
{
  "schema_version": 1,
  "bundle_identity": {
    "bundle_id": "tamper-tantrum-labs/inbox-triage",
    "version": "1.4.2",
    "publisher_id": "tamper-tantrum-labs",
    "created_at": "2026-04-19T14:32:00Z"
  },
  "runtime_constraints": {
    "mship_runtime_min": "1.0.0",
    "supported_platforms": ["linux-x86_64", "darwin-arm64", "windows-x86_64"]
  },
  "entries": [
    {
      "path": "payloads/skills/gmail-mcp/server",
      "size": 4917628,
      "sha256": "a9b3c1...",
      "media_type": "application/vnd.mship.skill.binary",
      "mode": "0755",
      "entry_type": "file"
    }
    /* ... one object per PAYLOAD file (anything under payloads/**); no payload file may
       be extracted that is not in this list. descriptor.json and signature.dsse are
       structural/signature artifacts and are explicitly NOT included in entries[] —
       including them would create a circular dependency (the signature would need to
       hash itself). Their presence and byte-equality is verified by a separate rule
       at descriptor-verified state. */
  ],
  "skills":  [ /* canonical-JSON-serialized skill manifests, inlined */ ],
  "agents":  [ /* canonical-JSON-serialized agent manifests, inlined */ ],
  "hooks":   [ /* canonical-JSON-serialized hook manifests, inlined */ ],
  "capability_summary": {
    /* flattened, sorted list of all declared capabilities across all skills/agents/hooks;
       used for the install-time review UI. Derivable from the manifests but pre-computed
       to make the review UI deterministic and audit-reproducible */
  }
}
```

**All manifests live inside `descriptor.json`.** There are no separate manifest files inside the archive. This binds every manifest byte to the single signed subject.

### Invariants

- **Identity = hash of descriptor.** `bundle_hash = SHA-256(canonical_json(descriptor))`. This is what every audit event references. Two archives with identical `descriptor.json` bytes have the same `bundle_hash` regardless of how payloads are arranged on disk inside the tar.
- **No file materialized before descriptor verification.** Signature is verified → descriptor parsed → Rekor / chain / revocation checks pass → only then is any payload file written to disk, and only if that file's path+hash+size+mode+media_type matches an `entries[]` object exactly.
- **Archive parser is the adversary.** Tar entries with duplicate paths, `..`, absolute paths, Windows drive letters / UNC prefixes, NTFS ADS colons, path length > 1024 bytes, non-NFC Unicode, case-insensitive collisions (on case-insensitive hosts), symlinks, hardlinks, FIFOs, device files, pax headers with semantic side effects, or sparse files are all rejected at parse. Only regular-file tar entries are accepted.
- **Decompression bounds.** Uncompressed descriptor + payloads max 100 MiB in V1 (operator-configurable). Gzip compression ratio > 20:1 is rejected as a decompression-bomb signal before full decompression.
- **Canonical path comparison.** Paths in `descriptor.entries[]` are compared against tar entries after NFC normalization + forward-slash separators + explicit rejection of any `./` prefix, trailing slash, or empty segment. No ambiguity between `payloads/x/y` and `./payloads/x/y`.
- **File permissions are operator-owned, not bundle-owned.** Descriptor declares intended mode, but supervisor imposes authoritative modes at commit time (`0644` for files, `0755` only for executable payloads whose paths are explicitly listed in the bundle manifest's `executables` field (`bundle.yaml.executables`), as represented in the descriptor).

### Why DSSE over a canonical JSON descriptor (rather than a tar+checksums.txt+detached signature)

- `checksums.txt` is a text file with implicit format (line syntax, path escaping, sort order, newline behavior) — that's a canonicalization-drift source. Codex correctness review flagged it as the most likely production failure. Canonical JSON (JCS) is formally specified, widely implemented, and already used for audit chain MACs.
- DSSE (Dead Simple Signing Envelope) is the in-toto community's standard for signing arbitrary statements. It pre-prefixes the payload with its `payload_type` before signing, which defeats cross-use attacks (a signature over one envelope type cannot be replayed against a different envelope type).
- Payload-type framing: `application/vnd.mship.bundle-descriptor+json`. The signed bytes are DSSE's standard PAE encoding: `"DSSEv1" || len(payload_type) || payload_type || len(payload) || payload`.
- Supports multi-signer (future V2+ enterprise co-signing) without re-architecting.
- Integrates naturally with sigstore: the DSSE envelope is what Rekor logs; the inclusion proof embedded in `signature.dsse` carries the Fulcio cert chain and log entry.

---

## Manifest schemas

Manifest schemas are authored in YAML. Codegen emits Rust / TypeScript / Python types from a source-of-truth schema (`manifests/schema.yaml`), same pattern as the secrets registry and audit events registry. YAML is authoring-only and does not appear in the installable bundle; the shipped and verified bundle format is the canonical JSON `descriptor.json`, which is the authoritative representation for installers/verifiers.

### `bundle.yaml` (bundle-level)

```yaml
schema_version: 1
bundle_id: "tamper-tantrum-labs/inbox-triage"      # globally unique; publisher-namespace/artifact
version: "1.4.2"                                   # semver required
publisher:
  id: "tamper-tantrum-labs"                        # publisher ID; matches cert subject
  display_name: "TamperTantrum Labs"
  identity_method: "operator-trust" | "sigstore"   # which validation path applies
description: "Inbox triage agent with per-client isolation."
created_at: "2026-04-19T14:32:00Z"
mship_runtime_min: "1.0.0"                         # minimum supported runtime version
skills: ["gmail-mcp", "asana-mcp"]                 # names matching manifests/skills/*.yaml
agents: ["inbox-triage-reader", "inbox-triage-drafter"]
hooks: ["pre-send-review"]
executables:                                       # authoritative list of paths that should be chmod +x
  - "payloads/skills/gmail-mcp/server"
  - "payloads/agents/inbox-triage-reader/bin"
```

### `skills/*.yaml` (one skill = one MCP server)

```yaml
schema_version: 1
name: "gmail-mcp"
version: "0.3.1"
mcp_version: "0.5"                                 # MCP spec version this server speaks
launch:
  command: "./payloads/skills/gmail-mcp/server"    # resolved relative to unpacked bundle root
  args: []
  env:                                             # ONLY secret refs; literal values disallowed except for whitelist
    GMAIL_OAUTH_TOKEN_HANDLE: "secret_ref:mothership/user/gmail-oauth"
capabilities:                                      # declared, exhaustive; runtime enforces
  network:
    outbound: ["gmail.googleapis.com:443"]         # specific hosts, not wildcards
    inbound: []                                    # none
  filesystem:
    read: []
    write: []
  shell:
    exec: false
  secrets:
    reads: ["mothership/user/gmail-oauth"]
  intents:
    emit: ["mail.draft", "mail.read", "mail.list"]
    consume: []
supported_platforms: ["linux-x86_64", "darwin-arm64", "windows-x86_64"]
isolation_hints:                                   # advisory; sub-spec 4 has final word
  seccomp_profile: "strict-network-only"
```

### `agents/*.yaml` (agent manifest)

```yaml
schema_version: 1
name: "inbox-triage-reader"
version: "1.2.0"
agent_type: "reader" | "planner" | "executor" | "action-agent" | "monitor"
model:
  tier: "local-70b" | "local-7b" | "claude-haiku" | "claude-sonnet" | "claude-opus"
  provider: "ollama" | "bedrock" | "anthropic-direct"
  default_sensitive_review_tier: "local-7b"        # second-pass fast-model per design-principles P4
triggers:
  - type: "operator-invoked"                       # V1: operator-invoked is the default
  - type: "on-intent"
    intent: "mail.received"
  # - type: "heartbeat"                            # FORBIDDEN in V1 per design-principles non-goals
capabilities:
  intents_emit: ["mail.draft"]
  intents_consume: ["mail.received", "mail.triage-request"]
  skills_invoked: ["gmail-mcp"]                    # listed skill names (must exist in bundle or already installed)
  agents_handoff: ["inbox-triage-drafter"]
  secrets: []                                      # agents typically do NOT hold secrets; skills do
input_preprocessors:                               # trust-boundary pre-processors from design-principles P4
  - "delimiter-injection"
  - "canary-token"
  - "pattern-scrub-urls"
provenance_consumption:                            # what provenance classes this agent is permitted to ingest
  allowed: ["operator-typed", "trusted-local", "skill-output"]
  forbidden: ["web-content", "third-party-message"]
org_context_scope: "per-install"                   # "per-install" | "shared" | "operator-selected-at-install"
```

### `hooks/*.yaml` (broker lifecycle hook)

```yaml
schema_version: 1
name: "pre-send-review"
version: "1.0.0"
lifecycle_event: "PreToolCall"                     # named event from sub-spec 2's enumeration
applies_to:
  tools: ["mail.send"]                             # tool-class filter; empty = all tools
  provenance: ["skill-output"]                     # provenance filter
script:
  language: "javascript-sandboxed"                 # V1: JS-sandboxed only; Python / WASM in V1.5+
  entry: "./payloads/hooks/pre-send-review/hook.js"
  timeout_ms: 500                                  # hard timeout; exceed = hook fails closed
  memory_limit_mib: 64
capabilities:                                      # hooks are maximally restricted by default
  intents_emit: []
  intents_consume: []
  network: {}
  filesystem: {}
decision_schema:                                   # hook must return a value matching this shape
  allowed_returns: ["allow", "deny", "revise"]
```

### Schema validation

- Every field is typed; missing required fields fail install.
- Unknown fields fail install (strict mode) rather than being silently ignored. This prevents malicious bundles from smuggling future-version fields past current-runtime validation.
- Cross-manifest references are validated at install time against the manifests inlined in `descriptor.json`: any declared skill reference must match a skill manifest present in this bundle, and agent `skills_invoked` entries must name skills that exist in this bundle or are already installed.

---

## Signing model

### What is signed

The publisher signs a **DSSE envelope** whose payload is the canonical JSON serialization of `descriptor.json`. The signed bytes are DSSE's standard PAE:

```
PAE = "DSSEv1"
    || SP || ASCII_DIGITS(len(payload_type)) || SP || payload_type
    || SP || ASCII_DIGITS(len(payload))      || SP || payload
```

where `payload_type = "application/vnd.mship.bundle-descriptor+json"` and `payload = canonical_json(descriptor)`.

Because the descriptor embeds every manifest and every file entry (path + size + hash + mode + media_type), the DSSE signature transitively covers every byte that install would materialize.

### Algorithms (V1)

- **Primary:** Ed25519 (EdDSA). Small signatures, fast verification.
- **Accepted:** ECDSA on P-256 or P-384 with SHA-256/SHA-384 (required for sigstore Fulcio compatibility — Fulcio's default EKU issues ECDSA-P256 certs).
- **Accepted:** RSA-PSS with SHA-256, key size ≥ 3072 bits (supports publishers with existing RSA PKI).
- **Rejected:** RSA-PKCS#1 v1.5, DSA, ECDSA with nonrandom k, any SHA-1 or MD5 anywhere in the signature or cert chain.

The publisher's cert declares the algorithm; the DSSE envelope's signature algorithm must match. Cross-algorithm mixes are rejected at install.

### Trust roots and namespace constraints

An operator's trust store is a directory of PEM-encoded root CA certs with per-root annotation files (`/etc/mothership/trust/<name>.pem` + `/etc/mothership/trust/<name>.trust.yaml`; platform-equivalent on Windows/macOS).

The annotation file is where namespace constraints live:

```yaml
trust_level: "community" | "verified" | "organization"
publisher_namespace_prefixes:              # which publisher_ids this root may issue
  - "tamper-tantrum-labs/*"                # owns publishers starting with this prefix
  - "tamper-tantrum-labs-*"                # explicit alternates
revocation:
  crl_url: "https://..."
  ocsp_url: "https://..."
  policy: "strict" | "lenient"             # strict fails install on revocation unknown/unreachable
notes: "TT Labs organization root; owns tamper-tantrum-labs namespace."
```

**Namespace enforcement at step 5 of install verification.** The publisher cert's subject / URI SAN must match a prefix in the chain-verifying root's `publisher_namespace_prefixes`. A root that trust-chains but whose namespace prefixes do not cover the bundle's `publisher_id` causes install to abort with `publisher-namespace-not-owned`. This prevents two trusted roots from silently both issuing `tamper-tantrum-labs/*` — only one root may own that prefix in a given trust store, and `mothership trust add` detects collisions and refuses.

V1 ships with **no default trust roots**. Operator action is required before any bundle installs. Realizes P7 at installer layer.

### Sigstore-keyless option

Publishers who sign via sigstore:

1. Publisher authenticates to Fulcio via OIDC; Fulcio issues a short-lived cert binding their OIDC identity to a one-time signing key.
2. Publisher signs the DSSE envelope with the one-time key.
3. The envelope + Fulcio cert + Rekor inclusion proof are published; `signature.dsse` carries all three.

Install verification when `identity_method: sigstore` (step 5 expansion):

- Verify DSSE signature using the Fulcio cert's public key.
- Verify the Fulcio cert chain up to an operator-configured sigstore root (Sigstore's public CA or a private Fulcio instance). SCT present; SCT signed by operator-configured CT log keys.
- Verify the cert's OIDC identity (`SubjectAlternativeName: URI` field carrying the OIDC subject) matches the operator-configured sigstore identity allowlist entry, AND the OIDC issuer matches that allowlist entry's `issuer` field (prevents a Fulcio cert from a different OIDC issuer impersonating identity).
- Verify the cert was valid at `integrated_time` from the Rekor entry (not at install-time wall clock).
- Verify the Rekor inclusion proof: Merkle path reaches the published checkpoint; checkpoint signed by operator-configured Rekor root public key; entry body's signature digest matches DSSE envelope's signature digest.
- Verify consistency proof from a previously-seen checkpoint (if one exists locally) to the current checkpoint; persistence-of-log-view defeats a rollback of Rekor.

Operators who don't opt in to sigstore have zero entries in their sigstore identity allowlist and cannot install sigstore-signed bundles. The operator-local-trust path is orthogonal.

### Operator's sigstore identity allowlist

```yaml
# /etc/mothership/trust/sigstore-identities.yaml
identities:
  - subject: "bundle-publishers@tampertantrumlabs.com"
    issuer: "https://accounts.google.com"
    publisher_namespace_prefixes:          # identity is ALSO namespace-constrained
      - "tamper-tantrum-labs/*"
  - subject: "releases@apisec.example"
    issuer: "https://github.com/login/oauth"
    publisher_namespace_prefixes:
      - "apisec/*"
rekor_root_public_key: /etc/mothership/trust/rekor-root.pub
fulcio_root_certs: /etc/mothership/trust/fulcio-roots.pem
sigstore_ct_log_keys: /etc/mothership/trust/ct-log-keys.pem
```

Missing any one of `rekor_root_public_key`, `fulcio_root_certs`, `sigstore_ct_log_keys` at install-time aborts all sigstore-signed-bundle installs with `sigstore-trust-incomplete`; never a silent fallback to a weaker path.

---

## Publisher identity model

### Cert requirements (operator-trust path)

Publisher X.509 cert must carry:

- `Subject CN` or `URI SAN` — publisher ID matching `bundle.yaml.publisher.id`
- `Key Usage`: Digital Signature
- `Extended Key Usage`: `1.3.6.1.4.1.57264.2.1` (Mothership Plugin Signing OID; provisional, to be registered)
- `Not Before` ≤ signing time; `Not After` ≥ signing time (but cert may be expired by install time if Rekor proves signing time)

### Trust levels

Each operator-trusted root CA carries a **trust-level annotation** (in a sidecar file `<root>.trust.yaml`):

```yaml
trust_level: "community" | "verified" | "organization"
# Note: V1 does NOT honor any per-root "default_grants" block — pre-suggestion is
# out in V1. This schema reserves the key for a possible V1.5+ reintroduction
# gated on design-partner evidence (see §"Trust levels × default-deny interaction (V1)").
notes: "TT Labs community root; moderate trust."
```

Trust-level is **advisory**. **In V1, trust level is displayed as a badge only — it does not pre-suggest, pre-check, or flag any capability as "recommended"** (see §"Trust levels × default-deny interaction (V1)" below). Per P7, every grant is an explicit operator action. Pre-suggestion UX patterns are revisited in V1.5+ only with design-partner evidence that richer UX does not erode operator decision quality.

### Multiple publishers in one bundle (not supported V1)

V1 bundles have exactly one publisher signature. Multi-signature bundles (publisher A signs the skill; publisher B co-signs as reviewer) are a V2+ design concern — interesting for enterprise IT deployments but adds review-chain complexity.

---

## Install-time verification — journaled transaction

Install is a **journaled state machine**, not a sequence of ad-hoc steps. Each state transition is durable, idempotent, and crash-recoverable; install flow audit events are emitted for the defined install-event milestones declared later in this spec (`install-requested`, `signature-verified`, `signature-failed`, `install-approved`, `install-rejected`, `uninstalled`). Only one terminal state (`activated`) makes the bundle's manifests observable to Broker (T3) and Skill process isolation (T4).

### States

```
  received ──> descriptor-verified ──> reviewed ──> staged ──> committed ──> activated
      │              │                   │           │           │
      └── rejected ──┴── rejected ───────┴── rejected┴── rejected ┘
                                                              │
  <any terminal state> ── deactivated (on revocation / operator uninstall)
                                    │
                                    └── deleted (after grace period)
```

Each state is persisted to `/var/lib/mothership/installs/<install_event_id>/state.json` (platform-equivalent). Supervisor crash between states: on restart, resume from the last durable state; partial side effects are idempotent or explicitly rolled back.

### Per-state contract

**`received`** — the `.mship` archive has been placed in a quarantine directory at `installs/<install_event_id>/archive.mship`. No further files have been written. An `install-requested` audit event is emitted. The archive is not yet trusted.

- Size check: reject if > 100 MiB (V1 default; operator-configurable). → `rejected(size-exceeded)`.
- Decompression-bomb check: do not rely on gzip header inspection for uncompressed size (gzip's `ISIZE` is optional and 32-bit-truncated). Instead, stream-decompress the gzip layer into the tar parser with (a) a hard maximum on total decompressed bytes (100 MiB V1 default, operator-configurable) and (b) a running output/input ratio check; abort as soon as output exceeds the max or the running ratio exceeds 20:1. → `rejected(decompression-bomb)` without completing decompression or writing payload files to disk.
- Archive parse: stream-parse tar entries, rejecting any non-regular-file entry, any disallowed path shape, any duplicate path. Every accepted tar entry is hashed on-the-fly. Only two files are extracted to memory: `descriptor.json` and `signature.dsse`. Payload files are NOT written to disk yet; their hashes are retained for the next state.
- Transition → `descriptor-verified` or `rejected(path-escape|duplicate-entry|tar-structural|…)`.

**`descriptor-verified`** — signature, chain, revocation, manifest schema, cross-references all pass. Descriptor is durable; payload hashes are retained but no payload byte is yet on disk.

- Parse `signature.dsse` (DSSE envelope). Parse `descriptor.json` **from the envelope payload bytes**, not from the archive's `descriptor.json` entry. The archive's `descriptor.json` entry is only a convenience for operator inspection — the authoritative bytes are in the signed envelope.
- Verify the archive's `descriptor.json` byte-identical to the envelope payload — any drift → `rejected(descriptor-archive-mismatch)`.
- DSSE signature verification (PAE framing + cert public key). → `rejected(signature-invalid)` if fail.
- Cert chain verify up to a trusted root (operator-local or sigstore-path). → `rejected(chain-untrusted | sigstore-trust-incomplete | cert-expired | cert-not-yet-valid)`.
- Namespace check: publisher_id matches the chain-verifying root's `publisher_namespace_prefixes` (or the matched sigstore identity allowlist entry). → `rejected(publisher-namespace-not-owned)`.
- Revocation check (local store + optional CRL/OCSP per trust-root policy). Strict policy + revocation-source-unreachable → `rejected(revocation-check-unavailable)`. Lenient policy + unreachable → proceed with an audit warning.
- Manifest schema validation on every embedded skill/agent/hook manifest. → `rejected(manifest-schema-invalid | manifest-cross-ref-invalid)`.
- Verify the tar contains exactly one `descriptor.json` entry and exactly one `signature.dsse` entry at the archive root; missing or duplicate required metadata entries → `rejected(descriptor-tar-mismatch)`.
- **Verify every tar file under `payloads/**` has exactly one matching object in `descriptor.entries[]` by `(path, size, hash)`, and vice-versa** (descriptor↔tar bijection, payloads only; the two structural entries above are explicitly excluded); any extra payload tar entry OR any descriptor entry without a matching payload tar entry → `rejected(descriptor-tar-mismatch)`.
- Any tar entry outside `payloads/**` that is not exactly `descriptor.json` or `signature.dsse` → `rejected(tar-structural)`.
- `plugin.signature-verified` audit event emitted.
- Transition → `reviewed` (if transitions to review) or `rejected(<reason>)`.

**`reviewed`** — operator has been presented the capability-review UI via sub-spec 7 re-auth gate. Operator has approved or rejected.

- UI displays: publisher identity, identity_method, trust level (informational only — no pre-suggested grants in V1), bundle_id, version, bundle_hash, full capability tree from `descriptor.capability_summary`, per-capability toggle (default off), a final free-text confirm box the operator must type.
- Timeout (default 10 minutes) → `rejected(operator-timeout)`.
- Operator rejection → `rejected(operator-rejected)`.
- Operator approval → `plugin.install-approved` audit event **NOT yet emitted** (approval is recorded in the journal; the event is emitted at `activated` so the audit log reflects reality only after the bundle is actually live).
- Transition → `staged` or `rejected(<reason>)`.

**`staged`** — payload files have been materialized to disk in an isolated install directory (`installs/<install_event_id>/payload/`), each file verified against its declared `(path, size, sha256, media_type)` before any byte is fsynced to its final path. Declared tar/header `mode` is advisory metadata only and does **not** participate in pass/fail verification — file permissions are operator-owned (see below). Files are **not** yet visible to the registry or the broker.

- For each `descriptor.entries[]` object: re-open the tar archive, stream the entry's bytes, compute SHA-256 progressively, compare against declared hash, validate declared `media_type` is in the allowlist for bundle payloads, write to a temp path in `payload/.partial/`, fsync, rename atomically to `payload/<path>` only on full-match.
- Any per-entry content mismatch (size, SHA-256, or disallowed media_type) → `rejected(entry-hash-mismatch)`; all partially-materialized files are deleted; the tarball stays in quarantine for operator inspection. A tar-header `mode` differing from the descriptor's declared `mode` does **not** abort install — supervisor imposes authoritative modes at the next step.
- Supervisor-authoritative modes are applied **after** content verification: `chmod 0644` by default, `0755` only for entries whose paths appear in the bundle manifest's `executables` field (`bundle.yaml.executables`) as represented in the descriptor.
- Transition → `committed`.

**`committed`** — the bundle's manifests are registered with the supervisor's verified-manifest registry, grants are recorded in the grant ledger (from `reviewed` state's approvals), and the broker has **not yet** been told about them. This is the last rollback point.

- Journal records registry+grants transaction; registry+grants write is a single supervisor-in-process database transaction (candidate: SQLite WAL or an append-only typed log). Failure mid-write → restart from `staged` on recovery.
- Transition → `activated`.

**`activated`** — supervisor signals broker to reload manifests. Broker's capability matrix incorporates the new bundle's skills, agents, and hooks. `plugin.install-approved` audit event is emitted *here* with the full payload; the audit log now reflects a live bundle.

- Transition terminal.

### Journal format

Journal entries (one file per install_event_id) are append-only JSON-lines with per-state fields:

```json
{"state":"received","ts":"2026-04-19T15:02:00Z","install_event_id":"…","archive_path":"…","archive_sha256":"…"}
{"state":"descriptor-verified","ts":"…","bundle_hash":"…","publisher_id":"…","identity_method":"…","trust_root_fingerprint":"…"}
{"state":"reviewed","ts":"…","grants_approved":["…"],"operator_identity_fingerprint":"…"}
{"state":"staged","ts":"…","payload_root":"…"}
{"state":"committed","ts":"…","registry_txn_id":"…"}
{"state":"activated","ts":"…"}
```

Journal existence across supervisor restart is what makes install recoverable; the journal itself is a secondary audit source (the primary audit log gets the `install-*` events at state transitions).

### Rollback primitive

`mothership plugin rollback <install_event_id>` — valid only for installs in states `committed` or `activated`:

- If `activated`: deactivate (broker stops offering the bundle's skills/agents), revoke grants, emit `plugin.uninstalled(reason: operator-rollback)`.
- If `committed` or `activated`: unregister from manifest registry; move `payload/` to `installs/<install_event_id>/rolled-back-payload/` for N days (default 7), then purge.

Rollback is re-auth-gated and audited. Distinct from `uninstall` (which is the routine operator-initiated removal path). Rollback explicitly acknowledges the previous activation; uninstall does not require the previous-state pedigree.

### Install-event audit shapes

Declared here for import into the audit log's event registry:

- `plugin.install-requested` — `{install_event_id, archive_sha256, archive_size}`
- `plugin.signature-verified` — `{install_event_id, bundle_hash, publisher_id, algorithm, publisher_cert_fingerprint, identity_method, trust_root_fingerprint?, sigstore_identity_subject?, rekor_entry_id?}`
- `plugin.signature-failed` — `{install_event_id, bundle_hash?, publisher_id?, algorithm?, publisher_cert_fingerprint?, identity_method?, reason_code, reason_detail}` where `reason_code` is an enum subset limited to signature-verification-path failures: `signature-invalid`, `algorithm-mismatch`, `chain-untrusted`, `sigstore-trust-incomplete`, `cert-expired`, `cert-not-yet-valid`, `cert-revoked`, `revocation-check-unavailable`, `publisher-namespace-not-owned`, `trust-store-namespace-collision`, `trust-root-removed-mid-install`, `rekor-proof-invalid`, `oidc-identity-not-allowlisted`. Emitted at the `descriptor-verified` boundary when the signature-verification path aborts; the terminal `install-rejected` follows with the same `reason_code`.
- `plugin.install-rejected` — `{install_event_id, bundle_hash?, bundle_id?, version?, state_at_rejection, reason_code, reason_detail}` where `reason_code` is the full enum (`size-exceeded`, `decompression-bomb`, `path-escape`, `duplicate-entry`, `tar-structural`, `descriptor-archive-mismatch`, `signature-invalid`, `algorithm-mismatch`, `chain-untrusted`, `sigstore-trust-incomplete`, `cert-expired`, `cert-not-yet-valid`, `cert-revoked`, `revocation-check-unavailable`, `publisher-namespace-not-owned`, `trust-store-namespace-collision`, `trust-root-removed-mid-install`, `rekor-proof-invalid`, `oidc-identity-not-allowlisted`, `manifest-schema-invalid`, `manifest-cross-ref-invalid`, `descriptor-tar-mismatch`, `entry-hash-mismatch`, `bundle-already-installed`, `operator-rejected`, `operator-timeout`, `registry-txn-failed`). This remains the terminal install-failure event, including cases preceded by `plugin.signature-failed`.
- `plugin.install-approved` — `{install_event_id, bundle_hash, bundle_id, version, publisher_id, publisher_cert_fingerprint, identity_method, trust_level, grants_approved: [capability]}` (emitted at `activated`)
- `plugin.uninstalled` — `{bundle_hash, bundle_id, reason: "operator" | "operator-rollback" | "revoked-publisher" | "superseded-by-version", grants_revoked: [capability]}`
- `plugin.revocation-impact-reported` — `{revocation_event_id, revoked_cert_fingerprint, trust_root_fingerprint?, sigstore_identity_subject?, scope: "publisher" | "root" | "sigstore-identity", affected_bundles: [{bundle_hash, bundle_id, version, grants_active_count, skill_processes_running_count}], agents_with_in_flight_intents_count, action: "automatic-deactivate" | "operator-approve-required"}`. Emitted immediately before the corresponding cascade of `plugin.uninstalled(reason: "revoked-publisher")` entries, per the revocation flow in §"Revocation — deactivate first, delete later".

**`reason_detail` payload hygiene:** strictly-limited format per audit log payload-taint rules. Control-character-free ASCII or NFC-normalized UTF-8; max 256 bytes; no embedded bundle content (no YAML/JSON fragments from the bundle, no tar metadata, no paths from the bundle's `entries[]`). The audit linter fails CI if any `reason_code` maps to a `reason_detail` generator that can include unsanitized bundle content. Operator inspection of the quarantined bundle contents is separate from audit log — audit log sees only the reason code and a sanitized-digest of the failing field.

---

## Revocation — deactivate first, delete later

Revocation splits into three distinct operations with separate authority and risk profiles:

1. **Deactivate** — atomic, synchronous, non-destructive. Broker stops routing to the revoked bundle's skills/agents/hooks. Grants are revoked. Skill processes tied to the bundle are stopped. Bundle state remains on disk, manifests remain in the registry (flagged `deactivated`). Audit entry `plugin.uninstalled(reason: revoked-publisher)` emitted.
2. **Delete** — manual or scheduled after a grace period. Moves the bundle's payload out of the active install tree into `revoked-bundles/<bundle_hash>/` for operator forensic review; after N days (default 30) the payload is purged from disk.
3. **Re-activate** — possible during the grace period before delete. Operator may mark a revocation as a false alarm (re-auth gated, separately audited); bundle is returned to `activated` state. After delete, re-activation requires a fresh install.

### Local revocation store

`/etc/mothership/revoked/<cert-fingerprint>.yaml` (platform-equivalent). Operator-managed via CLI:

- `mothership plugin revoke <cert-fingerprint> --reason <text>` — adds to local store; triggers immediate deactivate-phase for all matching installed bundles; re-auth gated.
- `mothership plugin revocations` — list.
- `mothership plugin revoke-remove <fingerprint>` — removes from local store; re-auth gated; does NOT automatically re-activate bundles (must be explicit per bundle).

### CRL / OCSP polling

Per trust-root annotation: `revocation.policy: strict | lenient`.

- **Strict** (UC2 compliance): poll CRL + OCSP at install time and periodically (default hourly) while bundles from this root are installed. Polling failures at install time cause `rejected(revocation-check-unavailable)`. Post-install polling failures alert but do not deactivate.
- **Lenient** (default): polling failures alert; install proceeds; post-install polling failures alert.

CRL / OCSP responses are cached (signed, TTL-bounded) for offline operation; stale CRL beyond its `Next Update` → treated as unavailable.

### Impact report (pre-deactivation)

Before any revocation-triggered deactivation fires, the supervisor produces a **revocation impact report** and emits `plugin.revocation-impact-reported` audit event:

```
Revoked publisher cert fingerprint: <…>
Trust root: <…>
Bundles affected: <count>
  - bundle_id=<…> version=<…> grants_active=<count> skill_processes_running=<count>
  - …
Agents with in-flight intents at revocation time: <count>
Operator action: <automatic-deactivate | operator-approve-required>
```

For **operator-trust**-path revocations (publisher cert on the local trust store): deactivate is automatic. Impact report is emitted alongside deactivation for forensic clarity.

For **sigstore**-path revocations (identity removed from allowlist): deactivate is automatic.

For **root-level** revocations (an entire trust root removed): deactivate is automatic but also triggers an operator-presence-gated approval step before bundle payloads are scheduled for delete. This is a break-glass safety valve for the "IT accidentally removed the wrong root and deactivated 500 employees' bundles" scenario Codex flagged.

Auto-deactivate is always safe (skills just stop running; no data destroyed). Auto-delete is the blast-radius concern, hence the separate, reversible, re-auth-gated stage.

### Supersession by new version

When a bundle at a newer version is installed with the same `bundle_id`, the prior version transitions to `deactivated` on the new bundle's `activated` state, emits `plugin.uninstalled(reason: superseded-by-version)`. Delete of the prior version follows the same grace-period + reauth cleanup as revocation.

---

## Trust levels × default-deny interaction (V1)

Every grant is an explicit operator action (P7). **Trust level is informational only in V1** — no capability is pre-suggested, pre-checked, or flagged "recommended" regardless of trust level. The install-time review UI shows the trust-level badge next to the publisher identity and nothing more.

This is a deliberate walk-back from an earlier draft that had `verified` and `organization` roots pre-suggest low-risk capabilities. Codex flagged the UX-pressure risk — pre-suggestion gradually habituates operators into rubber-stamping, which erodes P7 compliance in the field even if every grant remains technically a click. Until there is operator-deployment evidence that a richer UI does not degrade decision quality (earliest V1.5 after design-partner feedback), pre-suggestion is out.

| Trust level | Grant prompt behavior (V1) |
|-------------|------------|
| `community` | All grants default off; operator toggles each deliberately; badge reads "community publisher". |
| `verified` | All grants default off; operator toggles each deliberately; badge reads "verified publisher". |
| `organization` | All grants default off; operator toggles each deliberately; badge reads "<org-name> publisher". |

Trust-level annotation is still useful for revocation policy selection and audit-entry context. It just does not influence the grant dialog in V1.

---

## Test strategy (4 levels)

### L1 — Schema + type-level controls

- Manifest schemas generated from `manifests/schema.yaml` authoring source; canonical-JSON representations in `descriptor.json` are what actually ship in bundles. Unknown fields, missing required fields, wrong-type fields all fail compile-time in Rust (codegen) and fail runtime parse in TS/Python verifiers.
- **Canonical-JSON bit-exact parity across Rust/TS/Python for descriptor serialization.** Same golden-vector suite pattern as audit log's canonical-JSON test — MUST produce identical bytes and identical SHA-256 digests in all three implementations. Divergence is a hard CI fail.
- Capability taxonomy is a closed enum. Extending it is a Mothership runtime upgrade, not a bundle extension point. Disallowed capability strings fail at schema validation with the exact field path.
- Reason-detail payload taint lint: every `reason_code → reason_detail` generator is proven (by the same recursive-taint walker as audit log) to produce only `audit_safe` or `opaque_hash`-tainted output — never bundle paths, bundle content, operator PII, or attacker-controllable strings.

### L2 — Unit tests

**Archive parsing (hostile corpus):**

- Rejected: absolute paths, `..`, single-`./` prefixes, trailing slashes, empty path segments, path > 1024 bytes, Windows drive letters, UNC prefixes, NTFS ADS colons, hardlinks, symlinks, FIFOs, device files, sparse files, pax headers with semantic side effects, extended attributes, duplicate paths after NFC normalization, case-insensitive collisions on HFS+/NTFS hosts, zero-byte files where descriptor declares non-zero size, non-regular-file entry types.
- Rejected decompression bombs: streaming output exceeds 100 MiB cap; running output/input ratio exceeds 20:1 mid-stream; nested compression.
- Accepted: normal regular-file entries with NFC-normalized UTF-8 paths and declared modes.

**Descriptor↔archive binding:**

- Extra tar entry not in `descriptor.entries[]` → `descriptor-tar-mismatch`.
- Missing tar entry declared in `descriptor.entries[]` → `descriptor-tar-mismatch`.
- Tar entry with correct path but wrong size → `descriptor-tar-mismatch`.
- Tar entry with correct path+size but wrong content (hash differs) → `entry-hash-mismatch` at staging.
- Tar entry with correct path+size+content but wrong declared mode in descriptor → normalized at commit (supervisor-authoritative mode); no abort.
- `descriptor.json` file inside archive differs byte-by-byte from DSSE envelope payload → `descriptor-archive-mismatch`.

**DSSE signature verification:**

- Ed25519 / ECDSA-P256 / RSA-PSS 3072 / RSA-PSS 4096: good signatures pass.
- Tampered payload, same signature → fail.
- Tampered signature, same payload → fail.
- Wrong algorithm declared in DSSE envelope vs. algorithm in cert → fail (algorithm-mismatch).
- RSA-PKCS#1 v1.5 signature → fail (rejected algorithm).
- SHA-1 anywhere in cert chain → fail (rejected digest).
- DSSE PAE framing mismatch (length prefixes wrong) → fail.
- Signature valid for `payload_type = application/json` replayed against `application/vnd.mship.bundle-descriptor+json` → fail (payload-type binding).

**Canonicalization differential suite (Codex's top concern):**

- Bundles differing only in: duplicate YAML keys in authoring source, YAML anchors/merge keys (authoring-time — must be flattened to canonical JSON by build tool), case-folded paths, NFC vs NFD, `./` prefix, CRLF vs LF, trailing spaces, visually confusable Unicode in publisher_id. Every case must produce identical bundle_hash OR fail at the build-tool stage before signing.
- Build tool must reject ambiguous authoring inputs (duplicate keys, visually-confusable publisher_id vs. trust-root `publisher_namespace_prefixes`).

**Cert chain + namespace:**

- Self-signed operator-added root → passes.
- Cert chain up to an un-listed root → `chain-untrusted`.
- Chain valid but publisher_id outside the matched root's `publisher_namespace_prefixes` → `publisher-namespace-not-owned`.
- Two trust roots in the store both claiming the same prefix → `mothership trust add` refuses to add the second; if added out-of-band, install aborts with `trust-store-namespace-collision`.
- Expired intermediate CA → `cert-expired`.
- Clock-skew (`cert Not Before` after local wall-clock) → `cert-not-yet-valid` with NTP-recommended diagnostic.

**Sigstore / Rekor:**

- Valid Fulcio cert + correct Rekor inclusion proof + matched identity allowlist entry → passes.
- Tampered Rekor log-body (entry body SHA differs from DSSE signature digest) → `rekor-proof-invalid`.
- Rekor checkpoint not signed by operator-configured Rekor public key → `rekor-proof-invalid`.
- Rekor checkpoint inconsistent with previously-seen checkpoint (rollback attack) → `rekor-proof-invalid`.
- Fulcio cert OIDC `iss` claim mismatches allowlist entry's `issuer` → `oidc-identity-not-allowlisted` even if `sub` matches.
- Fulcio cert SCT signed by a CT log key not in operator's `sigstore_ct_log_keys` → `rekor-proof-invalid`.
- Missing any of `rekor_root_public_key` / `fulcio_root_certs` / `sigstore_ct_log_keys` → `sigstore-trust-incomplete`; no sigstore-signed bundle installs.

**Revocation:**

- Publisher cert fingerprint present in local revocation store → `cert-revoked` at install-time; matching already-installed bundles auto-deactivated.
- CRL strict + unreachable responder → `revocation-check-unavailable`.
- OCSP soft-fail in lenient mode → proceed with audit-warning entry; no rejection.
- Stale CRL past `Next Update` → treated as unreachable.
- Revoked intermediate CA → chain rejected even if leaf cert is still valid.
- Trust-root removal while install is running → install in any pre-`committed` state aborts with `trust-root-removed-mid-install`; any `committed`+ state runs to completion (the state was reached with valid trust at that moment) and is then subject to the revocation-scan on root removal.
- Local revocation store mutation race during install (concurrent writer) → install sees a consistent snapshot (transaction-scoped read) and does not flap.

**Journal + state machine:**

- Crash-injection at each state transition: supervisor killed between every adjacent state pair; on restart, journal is replayed; state is either safely resumed or safely rolled back; no partial side effect persists.
- Concurrent installs of the same `bundle_id`+`version` → only one reaches `activated`; the second sees `bundle-already-installed` at `descriptor-verified` and aborts cleanly.
- Journal write failure (disk full) during `received` → install aborts; archive is left in quarantine.
- Registry transaction failure at `committed` → rollback to `staged`; payload remains but is not visible to broker; operator can retry via `mothership plugin retry <install_event_id>`.

**Rollback + deactivate:**

- `mothership plugin rollback <install_event_id>` on `activated` bundle → broker stops seeing it; grants revoked; audit entry emitted; files moved to rolled-back-payload dir.
- Rollback on a `committed` (not yet `activated`) bundle → same primitives but shorter (no broker notification needed).
- Re-activate during the 30-day grace period after revocation → requires re-auth; audit entry distinct from install-approved.

### L3 — Integration tests

- Full install end-to-end: bundle built → DSSE-signed → installed → audit chain shows `requested → signature-verified → install-approved`.
- Operator-presence integration: `reviewed` state blocks on sub-spec 7 re-auth; timeout → `operator-timeout`.
- Post-install revocation cascade: revoke publisher → impact report emitted → deactivate fires → grant ledger updated → skill processes stopped → broker matrix refreshed → audit chain consistent across all of it.
- Root-level revocation with break-glass: removing a root triggers deactivate, impact report, and operator-presence-gated approval before delete stage.
- Cross-spec integration: installed skill's verified-manifest object is consumed by broker; attempt to invoke beyond declared capability triggers `broker.intent-denied`; audit event correlates across bundle_hash and broker denial.
- UC3 scenario: IT installs an org-signed bundle; operator accepts; employee can use; IT revokes → all employee bundles deactivate within the revocation-poll interval; audit log is SIEM-shippable.

### L4 — Acceptance tests (CI gates)

The map's spec-specific acceptance test: *"An unsigned or tamper-modified plugin bundle cannot install to a default-configured Mothership; an install attempt is rejected by the supervisor and produces an audit entry."*

Expanded to 15 scenarios, all must pass:

1. **Unsigned bundle** (missing `signature.dsse`) → `signature-invalid`.
2. **Tampered payload** (byte flip in a `payloads/*` file, descriptor unchanged) → `entry-hash-mismatch` at staging.
3. **Tampered descriptor** (byte flip in `descriptor.json` entry in archive, signature unchanged) → `descriptor-archive-mismatch`.
4. **Tampered signature** (byte flip in DSSE envelope signatures field) → `signature-invalid`.
5. **Forged key re-sign** (attacker re-computes descriptor + signs with their own key, presents their own cert) → `chain-untrusted`.
6. **Unknown publisher** (valid cert, unlisted root) → `chain-untrusted`.
7. **Revoked publisher** → `cert-revoked` + impact report for already-installed affected bundles.
8. **Expired cert (operator-trust path)** → `cert-expired`.
9. **Path-escape in archive** (`../../etc/passwd`) → `path-escape` at `received`.
10. **Manifest schema violation** → `manifest-schema-invalid` or `manifest-cross-ref-invalid`.
11. **Sigstore bundle with Rekor proof for different content** → `rekor-proof-invalid`.
12. **Sigstore bundle with OIDC issuer mismatch** → `oidc-identity-not-allowlisted`.
13. **Publisher-namespace collision attempt** (cert chains to a root, but publisher_id is outside that root's `publisher_namespace_prefixes`) → `publisher-namespace-not-owned`.
14. **Decompression bomb** (streaming output exceeds cap or running ratio exceeds 20:1 mid-stream) → `decompression-bomb` at `received`; no payload byte materialized.
15. **Crash-during-install recovery** (power-fail after `descriptor-verified` but before `reviewed`) → on supervisor restart, install state is resumed or rolled back cleanly; no partial install visible to broker.

For each scenario: rejection audit entry carries the correct `reason_code`, the correct `state_at_rejection`, no bundle content leak into audit log (reason_detail is taint-safe), and no partial filesystem side effects remain on abort paths.

### Complementary T4 acceptance test (cross-referenced, not this sub-spec)

P3's runtime half: *"a signed skill cannot exceed its declared manifest at runtime."* T1 (this sub-spec) guarantees manifest integrity through install; T4 (Skill process isolation) enforces manifest at the OS boundary. Both halves must pass for P3 end-to-end.

---

## Error modes

- **Corrupted archive** → `install-rejected(size-exceeded|path-escape|descriptor-tar-mismatch|entry-hash-mismatch)`. Quarantine retained for operator inspection.
- **Unknown publisher** → `install-rejected(chain-untrusted)`. No bundle state persists.
- **Expired cert** (non-sigstore) → rejected. Sigstore path accepts if Rekor confirms valid-at-signing.
- **Clock skew** (local clock ahead of cert Not Before) → rejected with `cert-not-yet-valid`; operator diagnostic recommends NTP.
- **Cert-store inaccessible at startup** → supervisor halts with actionable diagnostic. No bundles loadable; P7 preserved.
- **Revocation polling endpoint down** → alerting in non-strict mode; install proceeds. In strict mode, install aborts with `revocation-check-unavailable`.
- **Manifest parse error** → `install-rejected(manifest-schema-invalid)` with `reason_detail` limited to sanitized location metadata (for example, a JSON-pointer-like field path and optional line/column), plus at most an opaque hash/fingerprint of the offending value. Raw parser output, manifest snippets, and offending values MUST NOT be copied into `reason_detail`.
- **Quarantine write failure** (disk full) → supervisor halts install; bundle returned to operator unchanged.

---

## Interface produced (consumed by downstream sub-specs)

- **Bundle format spec** (this document, §"Bundle format") — for supervisor (install) and `mothership-bundle-tool` authoring tool (operator side).
- **Verified-manifest objects** — typed structs with `bundle_hash`, `publisher_id`, `publisher_cert_fingerprint`, `trust_level`, `identity_method`, and typed skill/agent/hook data. Consumed by Supervisor (T2) for the manifest registry and Broker (T3) for capability-matrix evaluation.
- **Install-event audit shapes** — exported to `audit-events.yaml` as declared above.
- **`mothership plugin` CLI subcommands** — `install <path-to-mship>`, `list`, `inspect <bundle_id>`, `uninstall <bundle_id>`, `revoke <cert-fingerprint>`, `revocations`, `revoke-remove <fingerprint>`, `quarantine list`, `quarantine purge`, `trust add <root.pem>`, `trust list`, `trust remove <fingerprint>`.
- **Mothership bundle build helpers** — a V1.5+ CLI `mothership-bundle-tool build <source-tree>` that produces signed `.mship` bundles; design forthcoming but the bundle format in this spec is the artifact it must emit.

---

## Principle compliance notes

| Principle | Compliance check | Result |
|-----------|------------------|--------|
| P3 | Can an attacker with a fresh, unverified identity publish a skill that reads user secrets on a default-configured Mothership? | **No.** Default trust store is empty (V1). No root trusted → no bundle installable → no skill runnable. Even with a trusted root, the cert chain must lead to it; fresh unverified identities cannot chain. Sigstore path requires operator-configured OIDC allowlist. |
| P3 (runtime half) | Can a signed skill exceed its declared manifest at runtime? | **Guaranteed here only up to manifest integrity.** Runtime enforcement is sub-spec 4's responsibility (OS-level process sandboxing per declared capabilities). Both halves must hold for P3 end-to-end. Acceptance test #10 for T4 is the complementary check. |
| P7 | After installing Mothership and N skills with no explicit grants, can any skill read filesystem, open shell, access secrets, or make external network requests? | **No.** Install step 8 defaults every grant to off; operator must explicitly toggle. Trust level influences UX suggestion but never auto-grants. |
| P8 | Do install-related events produce audit entries? | **Yes.** Six declared event types covering request/reject/approve/signature-verified/signature-failed/uninstall, with typed payload hygiene. |
| P2 | Can a compromised agent cause bundle install? | **No.** Install is supervisor-owned, operator-presence-gated, and re-auth-required at step 8. No agent-addressable API can trigger install or grant changes. |

### Principle tensions

- **P3 vs. developer-friction (sigstore path).** Requiring publishers to maintain long-lived trust roots raises the bar for hobbyist skill authors. Sigstore-keyless relaxes this while preserving integrity via Rekor transparency. Both paths are V1; operators choose which they trust. No principle violation — operators who disable sigstore accept a higher authoring bar.
- **Trust-level UX vs. default-deny.** A `verified` trust level pre-suggesting grants could drift into a de facto auto-grant if operators habitually click "approve all." Mitigation: the grant UI is required to force at least one click per grant (no "approve all" button in V1); every grant is individually logged; operators habituated to rubber-stamping are a UX problem beyond substrate scope but the substrate at least makes the decisions auditable.

---

## Open implementation-level questions (for writing-plans, not design)

1. **Tar library choice.** `tar-rs` (Rust) is mature; verify hardlink/symlink/pax attack resistance under a hostile-corpus test suite before adoption. Candidate fallback: custom streaming parser with an allowlist of tar record types. Evaluate in writing-plans.
2. **Cert parsing library.** `rustls-pki-types` + `webpki` vs. `x509-parser` vs. `rcgen` + manual chain walk. Ed25519 + ECDSA-P256 + RSA-PSS support all required; sigstore compatibility is the deciding factor.
3. **DSSE library.** `sigstore-rs` provides a DSSE implementation; alternative is a direct PAE implementation since the format is ~30 lines. Prefer direct impl for audit simplicity; vendor-only-for-Rekor-client aspects of sigstore-rs.
4. **Rekor client.** Implement protocol directly vs. depend on `sigstore-rs`. Protocol is stable; direct-impl reduces supply-chain dependency surface. Prototype and measure in writing-plans.
5. **Journal storage.** SQLite with WAL mode vs. append-only JSONL vs. platform-native (Windows Registry, macOS plist). Prefer JSONL for audit-parity with the audit log's format; SQLite only if transactional semantics on `committed` state prove insufficient with JSONL.
6. **Registry transaction store.** Where do verified-manifest objects live while a bundle is `committed` or `activated`? Candidate: same JSONL-based append-only store as journal, separate filename. Must support SIGHUP reload and crash-recovery.
7. **Grace period defaults.** 30 days between revocation-deactivate and auto-delete. Needs operator survey + UC2 compliance design-partner input; likely operator-configurable with a floor of 7 days.
8. **Build tool canonical-JSON serializer.** `serde_jcs` (Rust) + `json-canonicalize` (TS) + `jcs` (Python) must all agree bit-exactly for the build tool to produce a descriptor that verifies identically across languages. Golden-vector suite shared with audit log's canonical-JSON tests.
9. **Provisional OID.** `1.3.6.1.4.1.57264.2.1` is a placeholder EKU; actual IANA registration for Mothership Plugin Signing is deferred until V1.5. Interim: document in operator trust-root setup guide.
10. **CT log key rotation.** Sigstore's public CT logs rotate keys; operator's `sigstore_ct_log_keys` must be updatable without service restart and with a transparent window (accept both old and new keys during the rotation period). Design the mechanism in writing-plans.

These are implementation choices; resolving them does not change any contract in this sub-spec.

---

## Exit checklist (per map §"Uniform exit checklist")

- [ ] Design doc committed under `docs/superpowers/specs/`
- [ ] Implementation passes unit + integration test suite (L1 + L2 + L3)
- [ ] Integration tests with direct tier-dependencies pass (Audit log event shapes registered in `audit-events.yaml`; Supervisor manifest registry consumes typed verified-manifest objects)
- [ ] Principle compliance check re-run; any tensions logged with resolution (two tensions noted above with resolution paths)
- [ ] Spec-specific acceptance test passes (all 15 L4 tamper-and-trust scenarios in CI; audit entry correctness verified)

---

## Amendments log

| Date | Change | Reason |
|------|--------|--------|
| 2026-04-19 | V0 design drafted (Gavin + Cowork). | Third sub-spec authored under the V1 spec map (T1, design Phase I). Resolves `design-principles.md` open question 2 (signing authority = operator-local trust + optional sigstore; no Mothership CA in V1). |
| 2026-04-19 | V0.1 revision post-Codex review. | Switched from `checksums.txt` + detached signature to **DSSE envelope over a canonical JSON descriptor** (eliminates canonicalization drift; binds all manifests + file metadata in a single signed subject). Reframed install as a **journaled state machine** (`received → descriptor-verified → reviewed → staged → committed → activated`) with crash-recoverable transitions and no payload materialization before descriptor verification. Added **publisher-namespace constraints** on trust roots (prevents two roots both issuing the same publisher_id). **Dropped trust-level grant pre-suggestion** in V1 (Codex flagged rubber-stamping risk; badges-only for now, revisit in V1.5 with design-partner evidence). Revocation **split into deactivate / delete / re-activate** phases with operator-presence-gated break-glass for root-level revocations + impact report before deactivation. Expanded sigstore verification to detail Fulcio chain, SCT via CT log keys, Rekor checkpoint consistency, OIDC issuer+subject binding, namespace-constrained sigstore identity allowlist. Replaced YAML-in-bundle with **canonical JSON descriptor** (YAML is authoring-side only; eliminates parser divergence across Rust/TS/Python). Added reason_detail taint rules. 15 L4 acceptance tests, extensive L2 hostile-corpus + differential tests for canonicalization drift. See Codex Review section below. |
| 2026-04-19 | V0.2 consistency fixes post-Copilot PR review. | Fixed 11 consistency issues flagged by Copilot on PR #1: (a) cross-manifest references point to inlined descriptor manifests, not separate YAML files; (b) declared explicit `plugin.signature-failed` event shape; (c) added missing `reason_code` enum variants (`algorithm-mismatch`, `trust-store-namespace-collision`, `trust-root-removed-mid-install`, `bundle-already-installed`) that were referenced but undeclared; (d) fixed `MThip\bundle` typo; (e) aligned `executables` field at bundle level (not per-manifest); (f) clarified YAML is authoring-only; (g) removed contradictory pre-suggestion language from the publisher-identity section and commented-out `default_grants` in the trust-root annotation schema; (h) replaced gzip-header-inspection decompression-bomb check with streaming-ratio + max-byte-cap approach (ISIZE is unreliable); (i) softened "each state transition emits audit event" overstatement; (j) aligned "corrupted archive" error-mode terminology to actual enum codes; (k) constrained manifest-parse-error `reason_detail` to sanitized location + opaque hash per taint rules. No contract changes to downstream sub-specs. |
| 2026-04-19 | V0.3 second-round Copilot fixes. | Fixed 6 additional consistency issues on PR #1: (a) Scope now says "six-state journaled state machine" (was "nine-step procedure" leftover from pre-Codex draft); (b) `descriptor.entries[]` clarified to cover payload files only — `descriptor.json` + `signature.dsse` are structural/signature artifacts excluded from the bijection (circular-dependency note); (c) descriptor↔tar bijection scoped to `payloads/**`; added explicit structural-entry check + `tar-structural` on any non-payload non-metadata entry; (d) `staged` state verification tuple is `(path, size, sha256, media_type)` only — tar-header `mode` is advisory, never fails install; (e) declared `plugin.revocation-impact-reported` audit event shape alongside `plugin.uninstalled`; (f) exit checklist updated from "10 L4 scenarios" to "15 L4 scenarios" to match the L4 count. No contract changes to downstream sub-specs. |

---

## Codex Review

**Triggers matched:** auth/authorization (signing, cert chain, trust roots, publisher identity), infrastructure (bundle format, install flow, quarantine, journal), irreversible one-shots (install writes executables to disk; revocation triggers uninstall cascade).
**Codex effort:** default
**Reviewed:** 2026-04-19

### Codex findings (summary)

**Rollback.** Install described as "atomic" but was actually a multi-write sequence (move bundle → register manifests → create grants → emit audit) with no transaction semantics. Mid-commit failure could leave bundle on disk but unregistered, registered without grants, granted without audit, or visible to Broker before complete. Post-install revocation's "automatic uninstall" conflated deactivation with deletion; no dependency handling; no `mothership plugin rollback` command; no fault-injection tests.

**Blast radius.** Highest-risk substrate boundary — path by which third-party code becomes trusted executable. Signature/identity binding not fully specified: canonical encoding, Unicode normalization, case sensitivity, semver, publisher-namespace ownership, and whether `bundle_id` is parsed before or after signature verification all underspecified. Unpack-before-signature-verify is a big attack surface: tar parser + filesystem are both adversary-reachable before trust is established. Audit log could become a blast multiplier if `reason_detail` carries bundle content, terminal control sequences, or huge payload fragments. UC3 (enterprise): bad revocation could disable many employees simultaneously; no staged revocation or impact analysis.

**Missing tests.** No canonicalization tests for `SIGNED_BYTES`: YAML duplicate keys, anchors, merge keys, Unicode normalization, CRLF/LF, trailing spaces, null bytes, case-folded publisher IDs, semver equivalences, ambiguous `bundle_id`. No bind-to-bundle-identity attack tests (signature over X claiming manifest Y; same hash different publishers; etc.). Archive tests missed hardlinks, device files, FIFOs, pax global headers, sparse files, NTFS ADS, case-insensitive collisions, duplicate entries, decompression bombs, decompressed-size enforcement. Revocation missed OCSP soft-fail, stale CRL, unreachable responder, clock skew, revoked intermediate, trust-root removal mid-install, concurrent mutation. No fault-injection tests across install state transitions. Capability schema closure under-tested (wildcards, path normalization, secret-ref namespace spoofing, agent references to skills with broader capabilities than operator reviewed).

**Wrong assumptions.** `sha256(checksums.txt)` insufficient without canonical format for that file — line syntax, path escaping, sort order, duplicate path behavior, newline requirements, encoding all undefined. Unpack-before-verify unacceptable because archive parser and filesystem are part of attack surface. Rekor "proves signing time" is too high-level — verifier must validate entry body, integrated_time, log identity, checkpoint consistency, Fulcio chain, SCT/CT behavior, OIDC issuer/audience, signature-digest binding. Publisher identity = cert subject match is not enough — namespace control needed (two trusted roots could both issue same name). YAML parsing cannot be made safe by post-parse schema when Rust/TS/Python parse YAML differently; anchors/duplicate-keys/merge-keys/implicit types are security inputs. "Pre-suggested but unchecked" grants turn into rubber-stamping; pre-suggestion must be treated as policy influence or removed.

**Cheapest failure.** Canonicalization drift — installer verifies one representation, UI displays another, registry enforces a third. Will happen through YAML parsing, path normalization, duplicate archive entries, Unicode/case differences, or `checksums.txt` syntax long before exotic crypto failure. Catch with hostile corpus + differential tests that vary only in canonicalization inputs and assert every component derives identical canonical verified-manifest objects.

**Alternatives.** Stop inventing bespoke signing and archive verification. Use **DSSE envelope over a canonical JSON descriptor** (in-toto-style signed statement) that includes bundle identity + publisher identity + version + file digests + sizes + media types + capability manifests. Verify envelope + cert + Rekor + revocation + namespace constraints **before** extracting any payload. Stream-extract only files whose digest/size/type/normalized-path match the verified descriptor. YAML stays as authoring; installable artifact is canonical JSON. Install as a **journaled transaction** with states `received → verified → reviewed → staged → committed → activated`. Trust roots should be **namespace-constrained**. Revocation should split **disable from delete** with impact reports before auto-delete.

### Claude's resolution

- **Rollback finding:** **Accepted.** Rewrote install as a six-state journaled state machine with durable per-state persistence, crash-recoverable transitions, idempotent or rollback-able side effects, and an explicit `mothership plugin rollback <install_event_id>` command. Registry + grants is a single in-process transaction at `committed`; broker notification only at `activated`. Fault-injection tests added for every state transition.
- **Blast radius finding:** **Accepted.** Rewrote bundle identity binding to a canonical JSON descriptor whose hash is `bundle_hash` — one signed subject binds publisher_id, version, every file's path/size/hash/mode/media_type, and all manifests. Added recursive-taint payload hygiene for `reason_detail` (no bundle content, no control chars, 256-byte max). Added UC3 staged revocation, impact report, operator-presence break-glass for root-level revocations.
- **Missing tests finding:** **Accepted.** Added extensive L2 hostile-corpus + differential tests for canonicalization drift, bind-to-bundle-identity attacks (6 scenarios), archive attack surface (hardlinks, FIFOs, pax, sparse, NTFS ADS, case-collisions, duplicates, bombs), revocation edge cases (OCSP soft-fail, stale CRL, unreachable, skew, revoked intermediate, trust-root removal mid-install, concurrent mutation), journal crash-injection, concurrent installs, rollback + deactivate + re-activate.
- **Wrong assumptions finding:** **Accepted.** Replaced bespoke `checksums.txt` with canonical JSON descriptor (RFC 8785 / JCS, same golden-vector-parity rule as audit log). Flipped to **verify-before-unpack**: only `descriptor.json` + `signature.dsse` are extracted to memory at `received`; payload files are not materialized to disk until `staged` state passes entry-by-entry hash verification. Expanded Rekor verification to detail SCT via CT log keys, Fulcio chain, integrated_time, checkpoint consistency (defeating log rollback), OIDC issuer+subject binding, signature-digest binding. Added publisher-namespace constraints on trust roots and sigstore identity allowlist. Moved YAML to authoring-side only; installable bundle ships canonical JSON descriptor. Dropped trust-level grant pre-suggestion in V1 (revisit with design-partner evidence in V1.5).
- **Cheapest failure finding:** **Accepted.** Codex's differential-testing prescription is now an L2 test suite: bundles varying only in YAML authoring-source duplicates/anchors/merge-keys, path case/normalization, `./` prefix, CRLF/LF, visually-confusable publisher_ids — all must either produce identical `bundle_hash` or be rejected at the build-tool stage before signing. Build tool explicitly rejects ambiguous authoring inputs.
- **Alternatives finding:** **All accepted.** DSSE + canonical JSON descriptor, journaled transaction, namespace-constrained trust roots, disable-then-delete revocation with impact reports — all in the revised spec.

### Summary

Codex's review was correct on every structural point; the original spec would have shipped with canonicalization drift, unsafe unpack-before-verify behavior, and a non-transactional install. The revision replaces bespoke checksums + detached signature with DSSE-over-canonical-JSON-descriptor (industry-standard in-toto pattern), flips install order so no attacker-supplied file is materialized before full cryptographic verification, adds a journaled state machine for crash-recoverable installs, constrains trust roots by publisher-namespace, and separates revocation's deactivate-from-delete. The resulting spec is materially harder to exploit and measurably more testable; its downstream contracts to Supervisor (T2), Broker (T3), Skill process isolation (T4), and Audit log (T1) are all unchanged, so adjacent specs do not need revision.
