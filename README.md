# sbx kit for Michelangelo

<img width="1200" alt="Architecture: a single host machine running Docker Desktop. Docker Desktop runs a k3d Kubernetes cluster that hosts the whole Michelangelo platform (apiserver :15566, UI :8090, Cadence/Temporal, MinIO, KubeRay). An sbx microVM sandbox on the same host holds the coding agent and the michelangelo client (ma SDK/CLI), which talks to the platform over gRPC." src="https://raw.githubusercontent.com/ajeetraina/sbx-kits-michelangelo/main/docs/architecture.svg" />

Run [Uber's Michelangelo ML platform](https://michelangelo-ai.org/) on your host's **Docker Desktop**,
and use a [Docker Sandbox](https://docs.docker.com/ai/sandboxes/) microVM as an **isolated coding agent**
that talks to it - the architecture above.

## How it fits together

- **Michelangelo runs on the host.** Docker Desktop provides the Docker daemon; Michelangelo's
  [`ma sandbox create`](https://michelangelo-ai.org/docs/getting-started/sandbox-setup/) stands up a local
  **k3d** Kubernetes cluster that hosts the whole platform - apiserver (`:15566`), dashboard (`:8090`),
  workflow engine (Cadence/Temporal), object storage (MinIO) and a KubeRay compute cluster.
- **The sandbox holds the agent + client.** The [`client/`](./client) kit installs only the `michelangelo`
  SDK/CLI (`ma`) inside an sbx microVM and points it at the host platform over plaintext gRPC. It is
  credential-free and lightweight - nothing heavy runs in the sandbox.

## Run Michelangelo on the host

Prerequisites are all host-side (per the upstream
[sandbox setup](https://michelangelo-ai.org/docs/getting-started/sandbox-setup/)):

- **Docker Desktop** (or Colima / Docker Engine) - a running Docker daemon.
- **kubectl · k3d · Helm · Python 3.11 or 3.12 · Poetry** -
  `brew install kubectl k3d helm python@3.12 poetry`
  (Python **3.13+ fails**, so pin `python@3.12`.)

Then build the `ma` CLI and bring the platform up on Docker Desktop:

```bash
git clone https://github.com/michelangelo-ai/michelangelo.git ~/michelangelo
cd ~/michelangelo/python && poetry env use python3.12 && poetry install   # builds the `ma` CLI
export PATH="$PWD/.venv/bin:$PATH"
cd ~/michelangelo && bash scripts/kuberay/build-kuberay-images.sh          # local KubeRay images
ma sandbox create            # k3d cluster + platform on Docker Desktop (30-60 min first run)
ma sandbox demo pipeline     # smoke test → dashboard at http://localhost:8090
```

`ma sandbox create` wants roughly the upstream minimums - **4 vCPU / 8 GB RAM / 60 GB disk** - so give
Docker Desktop enough resources (Settings → Resources). Lifecycle: `ma sandbox sync` (redeploy),
`ma sandbox stop|start`, `ma sandbox delete`. On a partial/stalled bring-up, prefer `ma sandbox sync`
over delete-and-recreate. The web UIs are published to `localhost`: dashboard `:8090`, Grafana `:3000`,
Prometheus `:9092`, Cadence Web `:8088`; the gRPC API is `:15566`.

## Sandbox an agent against it

Layer the [`client/`](./client) kit onto any agent - it installs the `ma` client and wires it to the
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

## Publish the kit

```console
./scripts/push-kit.sh                 # validates, then pushes :latest to Docker Hub
TAG=v1 ./scripts/push-kit.sh          # pushes :v1
```

The push script runs `sbx kit validate` before `sbx kit push`, so a bad spec fails before anything reaches
the registry. CI ([`.github/workflows/publish.yaml`](.github/workflows/publish.yaml)) does the same on every
push to `main` that touches the spec.
