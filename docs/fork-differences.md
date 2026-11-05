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
