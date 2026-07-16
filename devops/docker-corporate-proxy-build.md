# Docker Builds Behind Corporate Proxies

A practical guide to getting `docker build` working on a corporate network — proxy resolution, SSL interception, and pulling private Git dependencies mid-build.

## Table of Contents

1. [The Problem in Plain English](#the-problem-in-plain-english)
2. [What's Actually Happening](#whats-actually-happening)
3. [The Solution](#the-solution)
4. [Pulling Private Git Dependencies at Build Time](#pulling-private-git-dependencies-at-build-time)
5. [Full Example](#full-example)
6. [When You Need This](#when-you-need-this)

---

## The Problem in Plain English

**You see this error:**

```text
E: Failed to fetch http://deb.debian.org/... Could not resolve host: deb.debian.org
```

or

```text
× Failed to download package
  ╰─▶ invalid peer certificate: UnknownIssuer
```

or your `pip install` / `npm install` step just hangs and times out inside the build, even though the exact same command works fine on your host machine.

**The confusing part:** your laptop has network access. Docker Desktop even shows you're "online." But anything a `RUN` step tries to reach mid-build can't get out.

## What's Actually Happening

### The Story in Analogies

#### Your Host Machine

```text
You → Corporate Proxy → Internet
      ✓ Proxy env vars configured once, all your tools use them
```

#### A Docker Build

```text
Dockerfile RUN step → ??? → Internet
      ✗ Runs in an isolated container with its OWN empty environment
      ✗ Never inherits your shell's HTTP_PROXY/HTTPS_PROXY
      ✗ Can't even find "the host" by name unless told how
```

A build container is a fresh, isolated environment. It doesn't know:
1. That a proxy exists
2. Where the proxy lives (a PID on your host? A URL on the corporate network?)
3. That your host's CA bundle is what's signing intercepted HTTPS traffic

Three separate gaps, three separate fixes.

## The Solution

### 1. Tell the container how to reach the host

On Linux (and some Docker Desktop configurations), `host.docker.internal` — the DNS name that resolves to "your host machine" from inside a container — isn't set up automatically. You have to add it explicitly:

```bash
docker build \
  --add-host=host.docker.internal:host-gateway \
  -t myapp:latest .
```

`host-gateway` is a special value Docker resolves to whatever the container's default gateway actually is — usually the host. This matters when your proxy runs as a local service the container needs to reach (e.g., a corporate proxy client listening on the host), not just when going straight to an external `proxy.company.com:8080`.

### 2. Pass proxy vars as build-args, not just env vars

This is the part people miss: setting `HTTP_PROXY` in your shell only affects `docker build` the *command* — it does **not** automatically propagate into the image build steps. Each `RUN` in your Dockerfile executes in a clean environment unless you explicitly wire the proxy through.

```dockerfile
# Dockerfile
ARG HTTP_PROXY
ARG HTTPS_PROXY
ARG NO_PROXY

ENV HTTP_PROXY=${HTTP_PROXY}
ENV HTTPS_PROXY=${HTTPS_PROXY}
ENV NO_PROXY=${NO_PROXY}

RUN apt-get update && apt-get install -y curl
```

```bash
docker build \
  --add-host=host.docker.internal:host-gateway \
  --build-arg HTTP_PROXY="$HTTP_PROXY" \
  --build-arg HTTPS_PROXY="$HTTPS_PROXY" \
  --build-arg NO_PROXY="$NO_PROXY" \
  -t myapp:latest .
```

Note: `ARG` values are available during the build but don't persist into the final image unless also set as `ENV` — decide deliberately whether your *runtime* container should keep proxy settings (usually: no, if the deployment target has direct network access).

### 3. Handle SSL certificate interception

Corporate proxies often terminate and re-sign HTTPS traffic with an internal CA (see [UV and Corporate Proxies](./uv-corporate-proxy-setup.md) for the deeper explanation of *why*). If your build pulls packages over HTTPS through that proxy, tools inside the container need to trust that CA too:

```dockerfile
COPY certs/combined_ca.pem /etc/ssl/certs/combined_ca.pem
ENV REQUESTS_CA_BUNDLE=/etc/ssl/certs/combined_ca.pem
ENV SSL_CERT_FILE=/etc/ssl/certs/combined_ca.pem
ENV CURL_CA_BUNDLE=/etc/ssl/certs/combined_ca.pem
```

Build the combined bundle once on your host (system CA bundle + corporate root cert), commit it alongside the Dockerfile, and copy it in — cheaper than reinstalling/reconfiguring interception logic in every image.

## Pulling Private Git Dependencies at Build Time

A common corporate pattern: one internal library, installed via `pip install git+https://...` or `pip install -e ../shared-lib`, gated behind a personal access token (PAT).

```dockerfile
ARG GIT_TOKEN

RUN pip install "git+https://x-access-token:${GIT_TOKEN}@github.internal.company.com/org/shared-lib.git"
```

```bash
docker build \
  --build-arg GIT_TOKEN="$GIT_TOKEN" \
  -t myapp:latest .
```

**Security note:** `ARG` values are visible in `docker history` unless you take care to avoid it. For anything beyond a short-lived, low-privilege token, prefer BuildKit secrets instead:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=git_token \
    GIT_TOKEN=$(cat /run/secrets/git_token) \
    pip install "git+https://x-access-token:${GIT_TOKEN}@github.internal.company.com/org/shared-lib.git"
```

```bash
DOCKER_BUILDKIT=1 docker build \
  --secret id=git_token,env=GIT_TOKEN \
  -t myapp:latest .
```

Secrets mounted this way never land in an image layer or `docker history`.

## Full Example

```bash
docker build \
  --platform linux/amd64 \
  --add-host=host.docker.internal:host-gateway \
  --build-arg HTTP_PROXY="$HTTP_PROXY" \
  --build-arg HTTPS_PROXY="$HTTPS_PROXY" \
  --build-arg GIT_TOKEN="$GIT_TOKEN" \
  -t myregistry.example.com/myapp:dev \
  .
```

`--platform linux/amd64` is worth calling out separately: if you're building on Apple Silicon for deployment to an x86 cluster (most on-prem OpenShift/K8s), without this flag you'll ship an arm64 image that fails to start with an "exec format error" on the target nodes.

## When You Need This

- Corporate laptop, VPN, or on-prem network with a mandatory HTTP/HTTPS proxy
- `docker build` succeeds for `FROM` (image pull) but fails on `RUN` steps that hit the network
- SSL errors that only show up inside the build, never on the host
- Private Git dependencies (internal PyPI-style installs, monorepo shared libraries) needed during `pip install` / `npm install`
- Building on Apple Silicon for an x86_64 deployment target

---

## Related

- [UV and Corporate Proxies](./uv-corporate-proxy-setup.md) - Why proxy SSL interception breaks some tools but not others
- [Enterprise Python Setup](./enterprise-python-setup.md) - Comprehensive corporate Python environment guide
- [Docker Compose Multi-Service](./docker-compose-multi-service.md) - Wiring multiple containers together locally
