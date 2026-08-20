# sbx kit for Michelangelo

A standalone [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) kit (`kind: mixin`) that turns
any sandbox into a one-command dev environment for [Uber's Michelangelo ML platform](https://michelangelo-ai.org/).

Michelangelo's [sandbox setup](https://michelangelo-ai.org/docs/getting-started/sandbox-setup/) runs the
whole platform, API server, workflow engine (Cadence/Temporal), object storage, and a KubeRay compute
cluster as a local **k3d** Kubernetes cluster. k3d needs a Linux host with a Docker daemon. A Docker Sandbox is *already* a Linux microVM with its **own private
Docker daemon**, so this kit drops Colima entirely: the sandbox **is** the host, k3d runs natively inside
it, and everything stays behind the hypervisor boundary, the host Docker/containerd is never touched.

## What the kit does

Four observable things, so each is independently verifiable:

1. Installs the cluster toolchain as the agent user (`1000`) into `~/.local/bin`: **kubectl** `v1.31.4`,
   **k3d** `v5.7.4`, **Helm 3** (latest), and **Poetry**. `k3d` is installed as a thin **shim** (the real
   binary is `k3d.bin`) that patches `cluster create` for the microVM kernel, see
   [microVM compatibility](#microvm-compatibility) so `ma sandbox create` runs unmodified.
2. Allows the network egress the platform needs: toolchain installers, `github.com`/PyPI for the repo and
   Python deps, and the container registries `ma sandbox create` pulls cluster images from
   (Docker Hub, GHCR, Quay). see [Network policy](#network-policy).
3. Is **credential-free** ~ Michelangelo runs entirely locally, so the kit declares no secrets.
4. Injects an `agentInstructions` note so the agent knows the toolchain exists and how to bring the
   platform up (`git clone` → `poetry install` → `ma sandbox create`).

> **Why no baked-in `ma` binary?** The `ma` CLI isn't a standalone download, it's produced by
> `poetry install` inside the [michelangelo repo](https://github.com/michelangelo-ai/michelangelo). The
> kit installs the *host toolchain*; you clone the repo into your workspace and build `ma` there (mirroring
> the upstream docs). The repo is not baked into the image.

## Prerequisites

### Launch the sandbox with the kit

Layer the mixin onto an agent. From the published image:

```console
sbx run --kit docker.io/ajeetraina777/sbx-kits-michelangelo:latest claude
```

Or straight from this repo over git:

```console
sbx run --kit "git+https://github.com/ajeetraina/sbx-kits-michelangelo.git" claude
```

Or from a local clone (the kit spec lives at the repo root):

```console
git clone https://github.com/ajeetraina/sbx-kits-michelangelo.git
sbx run --kit ./sbx-kits-michelangelo/ claude
```

The trailing `claude` is the coding agent that runs inside the sandbox — a separate axis from the kit.
Any supported agent works (`sbx run --help` lists them); swap in `codex`, `gemini`, or `shell` for a bare
environment.

### Size the sandbox

`ma sandbox create` stands up a real k3d cluster plus KubeRay and wants roughly the upstream minimums —
**4 vCPU / 8 GB RAM / 60 GB disk**. Give the sandbox enough memory at creation time:

```console
sbx run -m 8GB --kit docker.io/ajeetraina777/sbx-kits-michelangelo:latest claude
```

If the cluster is OOM-killed or pods won't schedule, the sandbox was started too small ~ `sbx rm` it and
recreate with a larger `-m`. (Sizing is a `sbx run` flag, not part of the kit spec.)

## Bring up the platform

Once attached, from inside the sandbox:

```console
export PATH="$HOME/.local/bin:$PATH"          # kubectl/k3d/helm/poetry/uv live here

git clone https://github.com/michelangelo-ai/michelangelo.git
cd michelangelo
# The base image ships Python 3.14, but Michelangelo needs 3.11/3.12. Point
# Poetry at the kit-provisioned python3.12 BEFORE installing (else numpy tries
# to compile from source and fails — there is no C compiler here).
(cd python && poetry env use python3.12 && poetry install)   # 3.12 venv + `ma` CLI
export PATH="$PWD/python/.venv/bin:$PATH"       # put `ma` on PATH (or use `poetry run`)

bash scripts/kuberay/build-kuberay-images.sh   # build local KubeRay images
ma sandbox create                              # 30-60 min first run; pulls cluster images
ma sandbox demo pipeline                        # smoke test
```

Use Temporal instead of the default Cadence engine with `ma sandbox create --workflow temporal`.
Lifecycle commands: `ma sandbox sync` (redeploy), `ma sandbox stop|start`, `ma sandbox delete`.

### Reaching the web UIs

Michelangelo exposes its dashboard on `:8090` and the gRPC API on `:15566`. Port mappings on a sandbox are
set **post-hoc from the host** (services inside must bind `0.0.0.0`, which the cluster's load balancer
does):

```console
sbx ports <sandbox-name> --publish 8090:8090 --publish 15566:15566
```

Then open <http://localhost:8090>. Port mappings are dropped when the sandbox stops — re-publish after
`sbx start`.

## microVM compatibility

The sandbox microVM kernel differs from a normal Linux host in three ways that would otherwise make a
stock `k3d cluster create` (and therefore `ma sandbox create`) fail. The kit installs `k3d` as a shim
that fixes all three transparently on every `cluster create`, so **you pass no extra flags** — plain
`ma sandbox create` just works:

| microVM difference | Symptom without the fix | What the shim does |
| --- | --- | --- |
| No `/dev/kmsg` | k3s kubelet aborts: `failed to create kubelet: open /dev/kmsg: no such file or directory` | `sudo mknod /dev/kmsg` on the host + bind-mounts it into every node |
| No VXLAN | k3s exits: `flannel ... failed to register flannel network: operation not supported` | forces `--flannel-backend=host-gw` (fine for a single host) |
| API advertised on `0.0.0.0` (unroutable as a destination here) | host `kubectl`/`helm` hang with `unexpected EOF` | rewrites the kubeconfig endpoint to `127.0.0.1` after create |

The shim only rewrites `k3d cluster create`; every other `k3d` subcommand passes straight through to the
real binary. It relies on passwordless `sudo` for the one-time `mknod` (available in the sandbox). The
private Docker daemon, ClusterIP/NodePort service routing, and Michelangelo's `127.0.0.1`-bound NodePort
port mappings all work; only k3s's built-in Traefik LoadBalancer speaker (`svclb`) can't run (no kernel
netfilter) — Michelangelo uses NodePort, not that LoadBalancer, so it is unaffected.

## Network policy

The kit declares (`permissions.network.allow`) the hosts needed to install the toolchain and pull the
common cluster images (Docker Hub, GHCR, Quay). This list is a **best-effort starting set** — Michelangelo
pulls a lot of images and the exact set can shift between versions.

If a pull is blocked during `ma sandbox create`, find the host and add it:

```console
sbx policy log        # on the host — shows "Blocked by network policy: domain <host>:443"
```

Add the named host to `permissions.network.allow` in [`spec.yaml`](./spec.yaml) and re-launch.

> **Under centralized governance**, if `sbx policy ls` shows `Governance: Managed by <org>`, that managed
> policy is default-deny and **overrides the kit's allow list** — local `sbx policy allow` rules are
> rejected for org-managed domains (`network policy … is managed by your organization`). The governance
> owner must add an org rule (e.g. `allowmichelangelo`) covering the toolchain-install hosts that aren't
> already allowed org-wide:
>
> ```
> dl.k8s.io            # kubectl binary (served directly, no redirect)
> get.helm.sh          # helm release archives
> ports.ubuntu.com     # apt inside the KubeRay image build (arm64)
> archive.ubuntu.com   # apt inside the KubeRay image build (amd64)
> security.ubuntu.com  # apt security updates during that build
> ```
>
> PyPI (`pypi.org`, `files.pythonhosted.org`) is used for Poetry + `poetry install` and is typically
> already allowed org-wide; GitHub (for k3d + cloning the repo) likewise. Image pulls during
> `ma sandbox create` come from Docker Hub / GHCR / Quay — add any that `sbx policy log` reports blocked.

## Verify

```console
# 1. Toolchain installed as the agent user, on PATH under ~/.local/bin
sbx exec -it <sandbox-name> bash -lc 'export PATH="$HOME/.local/bin:$PATH"; kubectl version --client && k3d version && helm version --short && poetry --version'

# 2. The sandbox has its own private Docker daemon (this is what replaces Colima)
sbx exec -it <sandbox-name> bash -lc 'docker version --format "{{.Server.Version}}"'

# 3. After `ma sandbox create`, the k3d cluster is up
sbx exec -it <sandbox-name> bash -lc 'export PATH="$HOME/.local/bin:$PATH"; k3d cluster list && kubectl get nodes'
```

## Publish the kit

```console
./scripts/push-kit.sh                 # validates, then pushes :latest to Docker Hub
TAG=v1 ./scripts/push-kit.sh          # pushes :v1
```

The push script runs `sbx kit validate` before `sbx kit push`, so a bad spec fails before anything reaches
the registry. CI ([`.github/workflows/publish.yaml`](.github/workflows/publish.yaml)) does the same on every
push to `main` that touches the spec.


