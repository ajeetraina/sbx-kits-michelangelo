# sbx kit for Michelangelo — **client** (thin client → host platform)

A `kind: mixin` Docker Sandboxes kit that turns any sandbox agent into a **thin
client** for a [Michelangelo ML platform](https://michelangelo-ai.org/) running
**on the host**. The sandbox holds only the client (`ma` CLI + `michelangelo`
Python SDK); the platform itself runs outside the microVM.

This is the **counterpart** to the self-contained kit at the repo root:

| | repo-root kit | this `client/` kit |
|---|---|---|
| What runs in the microVM | the **whole platform** (k3d + KubeRay + Cadence…) | just the **client** |
| Where Michelangelo runs | inside the sandbox | on the **host** |
| Pattern | platform-in-a-box | Grafana / Ollama "talk to a host service" |
| Cost | k3d shim, image pulls, 8–16 GB, 30–60 min | `pip install`, seconds, tiny |

Use the root kit for a disposable, zero-host-setup demo. Use **this** kit when a
Michelangelo already runs on your host and you just want a sandboxed agent to
drive it.

## Status & known limitation (read first)

This kit is **validated on the client side** — `pip install michelangelo` lands the
`ma` client and it talks to a Michelangelo apiserver correctly. What it needs is a
**routable backend the sbx egress proxy can forward to** (a real host/domain, the
way `pypi.org` is reachable).

A backend running on the **same macOS + Docker Desktop as the sandbox is NOT
reachable** — proven exhaustively: direct `host.docker.internal` access returns
"empty reply" (sbx doesn't forward raw host-loopback TCP), and the proxy path,
even with `gateway.docker.internal:<port>` allowed in policy, returns `502` (the
proxy can't route to the macOS host). This is an sbx + Docker-Desktop topology
boundary, not a kit bug. See the repo memory note
`michelangelo-client-kit-and-sbx-host-access` for the full trace.

**Use this kit against a routable/remote Michelangelo** (shared cluster, LAN host
with a hostname the org policy allows). Set `MACTL_ADDRESS=<that-host>:15566`, keep
that host OUT of `NO_PROXY` (it must go *through* the proxy), and allow the
`host:port` in the policy. For a purely local Michelangelo, use the self-contained
kit at the repo root, or just run/use the platform on the host directly.

## Prerequisites

1. **A Michelangelo platform running on the host.** The supported way is the
   upstream local install (`ma sandbox create`) on your Docker Desktop / Colima —
   it stands up a k3d cluster with the apiserver on `127.0.0.1:15566` and MinIO on
   `127.0.0.1:9091`.
2. **The host bridge** (below) — because `ma sandbox create` binds those services
   to **loopback**, which a separate microVM cannot reach.

## Host bridge (required)

`ma sandbox create` publishes the apiserver/MinIO on `127.0.0.1`, unreachable from
the sandbox VM. Re-expose them on `0.0.0.0` so `host.docker.internal` resolves to
a routable address. Run on the **host** and leave running:

```bash
kubectl port-forward --address 0.0.0.0 -n default svc/michelangelo-apiserver 25566:15566 &
kubectl port-forward --address 0.0.0.0 -n default svc/minio                    29091:9091  &
```

(Ports `25566` / `29091` avoid colliding with the `127.0.0.1:15566` / `:9091`
publishes k3d already holds. If you pick other ports, set `MACTL_ADDRESS` /
`AWS_ENDPOINT_URL` in the sandbox to match.)

## Usage

```console
sbx run --kit ./client claude
# or over git:
sbx run --kit "git+https://github.com/ajeetraina/sbx-kits-michelangelo.git#client" claude
```

Inside the sandbox the `ma` client is on `PATH` and already pointed at the host:

```console
ma mactl ...          # talks to host.docker.internal:25566 (plaintext gRPC)
```

## How it works

- **Install:** `pip install michelangelo` — the same package that ships the
  platform, but here only its client half (`cli/mactl` + the `uniflow` SDK) is
  used. No k3d, no Helm, no image pulls.
- **Wiring:** `michelangelo/cli/mactl/config.py` reads env overrides, so the kit
  sets them directly (no config file):
  - `MACTL_ADDRESS=host.docker.internal:25566` — the apiserver
  - `MACTL_USE_TLS=false` — the OSS apiserver speaks plaintext gRPC (no cert/SAN)
  - `AWS_ENDPOINT_URL` / `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — MinIO
    (`minioadmin`/`minioadmin`)
  - `NO_PROXY` includes `host.docker.internal` so the client bypasses the proxy
- **Egress:** only `pypi.org` (to install the client) and the two
  `host.docker.internal` ports. Nothing else — no chart hosts, no registries.
- **Credentials:** none. The local Michelangelo is credential-free, so unlike a
  SaaS client kit there is no token to inject.

## Troubleshooting

- `UNAVAILABLE: connections to all backends failing` → the host bridge isn't
  running, or `ma sandbox create` isn't up on the host. Verify with
  `lsof -iTCP:25566 -sTCP:LISTEN` on the host.
- `ma: command not found` → ensure `~/.local/bin` is on the agent's `PATH`.
