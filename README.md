# sbx kit for Michelangelo

<img width="1200" alt="Architecture: a single host machine running Docker Desktop. Docker Desktop runs a k3d Kubernetes cluster that hosts the whole Michelangelo platform (apiserver :15566, UI :8090, Cadence/Temporal, MinIO, KubeRay). An sbx microVM sandbox on the same host holds the coding agent and the michelangelo client (ma SDK/CLI), which talks to the platform over gRPC." src="https://raw.githubusercontent.com/ajeetraina/sbx-kits-michelangelo/main/docs/architecture.svg" />

Run [Uber's Michelangelo ML platform](https://michelangelo-ai.org/) on your host's **Docker Desktop**,
and use a [Docker Sandbox](https://docs.docker.com/ai/sandboxes/) microVM as an **isolated coding agent**
that talks to it — the architecture above.

## How it fits together

- **Michelangelo runs on the host.** Docker Desktop provides the Docker daemon; Michelangelo's
  [`ma sandbox create`](https://michelangelo-ai.org/docs/getting-started/sandbox-setup/) stands up a local
  **k3d** Kubernetes cluster that hosts the whole platform — apiserver (`:15566`), dashboard (`:8090`),
  workflow engine (Cadence/Temporal), object storage (MinIO) and a KubeRay compute cluster.
- **The sandbox holds the agent + client.** The [`client/`](./client) kit installs only the `michelangelo`
  SDK/CLI (`ma`) inside an sbx microVM and points it at the host platform over plaintext gRPC. It is
  credential-free and lightweight — nothing heavy runs in the sandbox.

## Run Michelangelo on the host

Prerequisites are all host-side (per the upstream
[sandbox setup](https://michelangelo-ai.org/docs/getting-started/sandbox-setup/)):

- **Docker Desktop** (or Colima / Docker Engine) — a running Docker daemon.
- **kubectl · k3d · Helm · Python 3.11 or 3.12 · Poetry** —
  `brew install kubectl k3d helm python@3.12 poetry`
  (Python **3.13+ fails**, so pin `python@3.12`.)

Then build the `ma` CLI and bring the platform up on Docker Desktop:

```bash
git clone https://github.com/michelangelo-ai/michelangelo.git ~/michelangelo
cd ~/michelangelo/python && poetry env use python3.12 && poetry install   # builds the `ma` CLI
export PATH="$PWD/.venv/bin:$PATH"
cd ~/michelangelo && bash scripts/kuberay/build-kuberay-images.sh          # local KubeRay images
ma sandbox create            # k3d cluster + platform on Docker Desktop (30–60 min first run)
ma sandbox demo pipeline     # smoke test → dashboard at http://localhost:8090
```

`ma sandbox create` wants roughly the upstream minimums — **4 vCPU / 8 GB RAM / 60 GB disk** — so give
Docker Desktop enough resources (Settings → Resources). Lifecycle: `ma sandbox sync` (redeploy),
`ma sandbox stop|start`, `ma sandbox delete`. On a partial/stalled bring-up, prefer `ma sandbox sync`
over delete-and-recreate. The web UIs are published to `localhost`: dashboard `:8090`, Grafana `:3000`,
Prometheus `:9092`, Cadence Web `:8088`; the gRPC API is `:15566`.

## Sandbox an agent against it

Layer the [`client/`](./client) kit onto any agent — it installs the `ma` client and wires it to the
host platform:

```console
sbx run --kit ./client claude
# or over git:
sbx run --kit "git+https://github.com/ajeetraina/sbx-kits-michelangelo.git#client" claude
```

The agent then drives Michelangelo (`ma` CLI + the `uniflow` SDK) with limited credentials and egress.
See [`client/README.md`](./client/README.md) for the wiring and the one caveat: the client needs a
**routable** backend the sbx proxy can forward to; a Michelangelo on the *same* macOS + Docker Desktop
as the sandbox is not routable from it.

---

## Alternative: run the whole platform inside the sandbox

The **repo root** is a separate `kind: mixin` kit that runs the *entire* Michelangelo platform inside a
single sbx microVM — no host Docker Desktop, fully isolated behind the hypervisor boundary. It is heavier
(the microVM needs a k3d-capable kernel and ~8–16 GB) but needs zero host setup: the sandbox **is** the
Linux + Docker host, so Colima is not needed and the host Docker/containerd is never touched.

### What the self-contained kit does

1. Installs the cluster toolchain as the agent user (`1000`) into `~/.local/bin`: **kubectl** `v1.31.4`,
   **k3d** `v5.7.4`, **Helm 3** (latest), and **Poetry**. `k3d` is installed as a thin **shim** (the real
   binary is `k3d.bin`) that patches `cluster create` for the microVM kernel — see
   [microVM compatibility](#microvm-compatibility) — so `ma sandbox create` runs unmodified.
2. Allows the network egress the platform needs: toolchain installers, `github.com`/PyPI for the repo and
   Python deps, and the container registries `ma sandbox create` pulls cluster images from
   (Docker Hub, GHCR, Quay). See [Network policy](#network-policy).
3. Is **credential-free** — Michelangelo runs entirely locally, so the kit declares no secrets.
4. Injects an `agentInstructions` note so the agent knows the toolchain exists and how to bring the
   platform up (`git clone` → `poetry install` → `ma sandbox create`).

### Launch and bring up

```bash
# 8 GB, since `ma sandbox create` wants ~4 vCPU / 8 GB / 60 GB
sbx run -m 8GB --kit docker.io/ajeetraina777/sbx-kits-michelangelo:latest claude

# Inside the sandbox: ONE command clones the repo, builds `ma`, and brings the cluster up
michelangelo-up --demo          # 30-60 min first run; add `--workflow temporal` to swap the engine
```

`michelangelo-up` runs the whole upstream flow (clone → build `ma` → build KubeRay images →
`ma sandbox create`) and is re-runnable; `~/.local/bin` is already on `PATH`. Publish the web UIs from
the host with `sbx ports <sandbox-name> --publish 8090:8090 --publish 15566:15566`, then open
<http://localhost:8090>.

### microVM compatibility

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
real binary. It relies on passwordless `sudo` for the one-time `mknod` (available in the sandbox). Only
k3s's built-in Traefik LoadBalancer speaker (`svclb`) can't run (no kernel netfilter) — Michelangelo uses
NodePort, not that LoadBalancer, so it is unaffected.

### Network policy

The kit declares (`permissions.network.allow`) the hosts needed to install the toolchain and pull the
common cluster images (Docker Hub, GHCR, Quay). This list is a **best-effort starting set** — Michelangelo
pulls a lot of images and the exact set can shift between versions. If a pull is blocked during
`ma sandbox create`, find the host with `sbx policy log` and add it to [`spec.yaml`](./spec.yaml).

> **Under centralized governance**, if `sbx policy ls` shows `Governance: Managed by <org>`, that managed
> policy is default-deny and **overrides the kit's allow list** — local `sbx policy allow` rules are
> rejected for org-managed domains. The governance owner must add an org rule covering the hosts that
> aren't already allowed org-wide:
>
> ```
> dl.k8s.io · get.helm.sh                       # kubectl + helm
> ports.ubuntu.com · archive.ubuntu.com · security.ubuntu.com   # apt in the KubeRay build
> proxy.golang.org · sum.golang.org             # Go modules in the KubeRay build
> cadence-workflow.github.io · go.temporal.io   # workflow-engine helm charts
> kubeflow.github.io · ray-project.github.io    # spark-operator + KubeRay helm charts
> istio-release.storage.googleapis.com · dl.min.io · quay.io    # istio · MinIO · KubeRay images
> ```
>
> The Go module hosts are easy to miss (the KubeRay build compiles Go); the helm-chart hosts are hit
> during `ma sandbox create` when it adds the engine/operator chart repos; `quay.io` serves the KubeRay
> operator image.

### Verify

```console
# Toolchain installed as the agent user, on PATH under ~/.local/bin
sbx exec -it <sandbox-name> bash -lc 'export PATH="$HOME/.local/bin:$PATH"; kubectl version --client && k3d version && helm version --short && poetry --version'

# The sandbox has its own private Docker daemon (this is what replaces Colima)
sbx exec -it <sandbox-name> bash -lc 'docker version --format "{{.Server.Version}}"'

# After `ma sandbox create`, the k3d cluster is up
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
