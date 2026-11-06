# Getting Started

This page takes you from a fresh clone to a running proxy with a live dashboard.

## Prerequisites

- **Rust 1.89+** (the workspace pins a minimum `rust-version`). Install via
  [rustup](https://rustup.rs).
- A C toolchain (C/C++ compiler, cmake, libclang) is needed to build:
  BoringSSL via `boring-sys` is a hard dependency of the default
  (`full`) and `minimal` feature sets.

## Build

```bash
git clone --branch fork/main https://github.com/icy37785/meow-rs.git
cd meow-rs
cargo build --release
```

The binary lands at `./target/release/meow`. It is a single static-ish executable of
roughly 6 MiB; you can copy it to any matching host and run it directly.

::: tip Fork binaries and updates
Download the artifacts of a successful
[Fork Build run](https://github.com/icy37785/meow-rs/actions/workflows/fork-build.yml).
The active targets are Apple Silicon macOS (`aarch64-apple-darwin`),
Windows x86_64 MSVC and Linux x86_64 musl. Builds use the `full` feature set;
archive names and `meow -v` identify the fork and source commit.
:::

There is no kernel binary self-updater or `/upgrade` API. Verify the downloaded
archive's SHA-256, stop the service, replace the binary and restart, preserving
your config and data directory. Use fork artifacts: upstream releases do not
include this fork's subscription/chaining extensions. Subscription and geodata
refresh update data only and do not install a new kernel.

## Configure

Copy the sample config and edit in your servers:

```bash
cp config.example.yaml config.yaml
```

A minimal working config looks like this:

```yaml
mixed-port: 7890                 # one port for both HTTP and SOCKS5
external-controller: 127.0.0.1:9090   # REST API + dashboard

proxies:
  - name: hk
    type: trojan
    server: example.com
    port: 443
    password: "your-password"
    sni: example.com

proxy-groups:
  - name: Proxy
    type: select
    proxies: [hk, DIRECT]

rules:
  - GEOIP,CN,DIRECT          # stay home if it's local
  - MATCH,Proxy              # everything else clears the wall
```

See the [Configuration overview](./configuration) for every key, and
[Proxies](./proxies) for each protocol's fields.

## Validate

Before running for real, ask meow-rs to parse and validate the config without starting:

```bash
./target/release/meow -f config.yaml -t
```

This catches typos, unknown rule types, duplicate listener ports, and invalid proxy
fields up front. A clean exit means the config loads.

## Run

```bash
./target/release/meow -f config.yaml
```

Then open the dashboard at **<http://127.0.0.1:9090/ui>** to switch proxies, edit rules,
and watch live traffic.

## Connect a client

Point any HTTP/SOCKS-aware tool at the mixed port:

```bash
export https_proxy=http://127.0.0.1:7890
export http_proxy=http://127.0.0.1:7890
curl https://ipinfo.io
```

For system-wide, device-wide capture without per-app proxy settings, set up a
[transparent proxy](./transparent-proxy) instead.

## Where to next

- [Configuration overview](./configuration) — the full top-level key reference.
- [Rules](./rules) — every rule type and how matching works.
- [CLI & Service](./cli) — flags, plus installing meow-rs as a systemd / launchd service.
