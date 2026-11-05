# Fork differences

This file records deliberate fork extensions so upstream updates can preserve
and verify them independently of unrelated changes.

## Plaintext Trojan (`skip-tls`)

This fork supports `skip-tls: true` on Trojan outbounds. The key defaults to
`false`; absent or false retains the existing TLS handshake and certificate
verification behavior. A present `skip-tls` must be a boolean.

With `skip-tls: true`, the adapter writes the Trojan request directly to the
server stream. It does not construct a TLS layer; `sni` and
`skip-cert-verify` have no effect. This supports compositions such as Leaf's
`[shadowsocks, trojan]`, where another hop supplies encryption. Configure the
outer hop using the existing relay or `dialer-proxy` mechanism:

```yaml
proxies:
  # Define an encrypted outbound named outer-ss separately.
  - name: inner-trojan
    type: trojan
    server: trojan.internal.example
    port: 443
    password: replace-with-server-password
    skip-tls: true
    dialer-proxy: outer-ss
    udp: true
```

**Security boundary:** plaintext Trojan provides no confidentiality or server
authentication. The SHA-224 password token and traffic are exposed to anyone
who can observe the unencrypted leg. An outer encrypted hop protects only the
portion of the path it covers; verify the entire path to the Trojan server.
The adapter also permits direct plaintext connections when explicitly selected.

Only the default TCP transport is supported: `network` may be absent, empty,
or `tcp` (case insensitive). Other values are rejected rather than silently
ignored. This restriction concerns the server transport; `udp: true` still
supports Trojan UDP framing over that stream. Existing sing-mux options also
work over the plaintext stream.

### Compatibility and maintenance

Treat this as a fork extension, not a promised mihomo-compatible option.
The mihomo Meta implementation inspected during review has no `skip-tls`
option in [`adapter/outbound/trojan.go`](https://github.com/MetaCubeX/mihomo/blob/Meta/adapter/outbound/trojan.go). The TCP-only restriction is this
fork's implementation boundary, not an upstream compatibility claim.

The extension is localized to the Trojan adapter and its config parser.
`TrojanAdapter::new` retains its upstream signature and TLS behavior;
`new_plaintext` is the explicit opt-in entry point. Both use one initializer
and share TLS wrapping across ordinary and mux connections. No new shared
traits, dependencies, or Cargo features are required.

After incorporating upstream changes, run:

```sh
cargo test -p meow-config --lib parse_trojan_skip_tls
cargo test -p meow-proxy --test trojan_integration
cargo test -p meow-proxy --no-default-features --features trojan --test trojan_integration
```

These cover config-to-wire behavior, malformed options, default TLS,
plaintext TCP, relay-supplied streams, UDP multi-destination framing, and the
plaintext mux handshake. The mux test verifies the wire prefix and stream
opening; it is not a full remote mux interoperability test.

## Provider-local `dialer-proxy` references

Provider nodes can reference siblings in the same provider. This supports a
subscription that contains both an inner Trojan node and its outer
Shadowsocks dependency without copying the dependency into top-level
`proxies:`. This is an additive fork extension, not a mihomo parity claim.

Resolution uses the existing global namespace first (top-level proxies and
groups), then the referencing node's own provider slot. A remote provider
cannot shadow an operator-defined global hop or resolve another provider's
private node names. Existing provider-level and `override.dialer-proxy`
precedence remains unchanged. This applies to node chaining; the provider's
HTTP fetch `proxy:` setting continues to use the global namespace.

The full provider slot supplies sibling lookup. A **group's** `filter` or
`exclude-filter` can hide an outer dependency from the selectable list while
keeping it available to inner nodes. A **provider's** own filters remove
nodes from that slot, so they must retain dependencies needed by the chain.

Successful refreshes atomically replace the slot used for sibling lookup;
retained nodes resolve the current sibling, including its updated endpoint.
Failed refreshes retain the last-good slot. Removed or missing hops fail
closed rather than dialing directly (ADR-0002's Class A security boundary).
Cycles are bounded by the existing chain-depth guard (mux session cycles
remain bounded by their existing setup timeout).

The lookup holds the provider slot weakly and reads the existing node set;
it does not maintain a second registry or introduce a provider ownership
cycle. Siblings are never promoted into the global registry. Lookup scans
the slot once per unresolved global hop, not per packet.

Regression checks:

```sh
cargo test -p meow-config --lib provider_sibling
cargo test -p meow-proxy --lib provider_target
```

These exercise a forward reference through an actual CONNECT chain,
group filtering, failed refresh preservation, removal of a referenced
sibling, provider isolation, global precedence, live slot replacement,
cycle rejection, and weak ownership.

## Hidden dependency nodes (`hidden: true`)

This fork accepts a boolean `hidden` on leaf proxy nodes, defaulting to
`false`. This extends the existing group-level `hidden` display option;
node-level behavior is a fork extension. Names and protocol types have no
special meaning: `外层-` is not hardcoded, and being referenced by another
node does not automatically hide a proxy that may also be a usable exit.

A hidden node remains in the full provider slot or global registry. Explicit
rules, relay members, `dialer-proxy` references and named inspection/probe
requests still work. Hidden does not mark a node dead or forbid routing.
It opts a dependency out of ordinary exit discovery and standalone bulk
probes:

- Provider-backed `use:` / `include-all*` groups use a live visible view.
- `include-all-proxies` and auto-created GLOBAL exclude hidden leaf nodes.
- Explicit group `proxies:` lists remain authoritative, including a group
  deliberately used to select between hidden outer hops.
- Bulk provider/group probes and scheduled sweeps skip hidden leaf nodes
  without recording failed delays. Inner-node probes still exercise the
  full chain through those dependencies.
- Proxy/provider management responses retain nodes and expose `hidden: true`;
  visible leaf nodes keep their existing response shape. Dashboards can
  honor the flag, and the CLI's normal proxy list omits hidden nodes.

The full slot and its filtered views update together on successful refresh;
changes to `hidden` therefore change group membership without discarding
chain dependencies. A failed strict refresh preserves the last-good state.
A group's own `hidden` remains a GUI-only option and does not disable its
routing or health checks.

Checks:

```sh
cargo test -p meow-config --lib hidden
cargo test -p meow-config --lib provider_sibling
cargo test -p meow-api --test api_test hidden_nodes_keep_named_access
```
