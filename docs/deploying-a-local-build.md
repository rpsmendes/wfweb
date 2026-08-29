# Deploying a local build to a remote Docker host

How to compile wfweb yourself, ship the resulting Docker image to another
machine on your LAN (no registry required), and swap it into an existing
`docker compose` stack.

This is the workflow for running a **patched** wfweb — e.g. a feature branch —
in place of the published `k1fm/wfweb` image.

- **Build machine:** any host with Docker (Windows + Docker Desktop, macOS, or
  Linux). Docker Desktop builds Linux images, so the output runs on a Linux
  server unchanged.
- **Target machine:** an Ubuntu server on the LAN already running other
  containers via `docker compose`, reachable over SSH.

---

## 1. Build the image

From a clone of the branch you want to run:

```bash
git submodule update --init --recursive
```

```bash
docker build -f docker/Dockerfile -t wfweb-local .
```

Run locally (Optional)
```bash
docker run --rm -it -p 8080:8080 -p 8081:8081 wfweb-local --lan 192.168.1.100 --lan-user admin --lan-pass secret
```

The `docker build` **is** the compile. If the C++ fails to build, it fails
here. The first build takes ~15–20 min (it pulls Ubuntu, Qt 5, and builds the
RADE custom Opus + codec2); later rebuilds after a code change only re-run the
`qmake && make` layer and finish in a minute or two.

`docker/Dockerfile` is a multi-stage build — Ubuntu 24.04 to compile, Ubuntu
25.10 for the ~150 MB runtime image.

### Architecture

`docker save` / `docker load` copies the image exactly as built. It works only
if the build machine and the target agree on CPU architecture.

| Build machine | Target | OK? |
|---|---|---|
| Windows / Linux x86-64 | x86-64 server | yes |
| Apple Silicon / ARM | x86-64 server | no — rebuild with `docker buildx build --platform linux/amd64 ...` |
| x86-64 | ARM server (e.g. Raspberry Pi) | no — rebuild with `--platform linux/arm64` |

Check the target with `uname -m` (`x86_64` = amd64, `aarch64` = arm64).

---

## 2. Tag it

Give the build a versioned tag so the stock image stays available for rollback:

```bash
docker tag wfweb-local:latest wfweb:9700-memfix
```

Use whatever label identifies the change (`wfweb:<branch>`, `wfweb:<date>`, …).

---

## 3. Copy it to the server over LAN

No registry needed — stream the image through SSH, or copy a tarball.

> **Windows:** run these from **Git Bash**, not PowerShell. PowerShell corrupts
> binary data in pipes and redirects.

### Option A — piped (one step)

```bash
docker save wfweb:9700-memfix | gzip | ssh user@192.168.1.X 'gunzip | docker load'
```

### Option B — tarball (two steps)

```bash
docker save wfweb:9700-memfix -o wfweb-9700-memfix.tar
```

```bash
scp wfweb-9700-memfix.tar user@192.168.1.X:/tmp/
```

```bash
ssh user@192.168.1.X 'docker load -i /tmp/wfweb-9700-memfix.tar && rm /tmp/wfweb-9700-memfix.tar'
```

### Verify (on the server)

```bash
docker images | grep wfweb
```

Expect a row for `wfweb   9700-memfix`. The published `k1fm/wfweb` image is
untouched, so rollback stays a one-line change.

---

## 4. Point docker compose at the local image

Edit the `wfweb` service in your `docker-compose.yml`. Only two lines change:
the `image:` reference, and a `pull_policy: never` so `docker compose` never
tries to pull a tag that exists only on this host.

```yaml
  wfweb:
    image: wfweb:9700-memfix        # was: k1fm/wfweb
    container_name: wfweb
    restart: unless-stopped
    pull_policy: never
    ports:
      - "3030:8080"
      - "3031:8081"
    command:
      - "--lan"
      - "192.168.1.229"       # IC-9700 IP
      - "--lan-user"
      - "XXXX"                # radio network user
      - "--lan-pass"
      - "XX_XXXXX"            # radio network password
```

Apply — compose notices the image change and recreates just this container:

```bash
docker compose up -d wfweb
```

Watch it start and identify the rig:

```bash
docker compose logs -f wfweb
```

Then open `http://<server-ip>:3030`.

---

## 5. Rollback

Revert the two lines:

```yaml
  wfweb:
    image: k1fm/wfweb
    # (remove the pull_policy: never line)
```

```bash
docker compose up -d wfweb
```

The stock image is still cached locally, so this is instant and offline-safe.

---

## Iterating

Every new build needs a fresh tag and another `save` / `load`
(`wfweb:9700-memfix2`, …) — `docker load` will not overwrite an in-use tag
cleanly otherwise, and reusing a tag makes it ambiguous which build is running.

If you expect to update often, publish to a registry instead — your own Docker
Hub repo or GHCR — and let the server `docker compose pull` like it does for
every other service:

```bash
docker tag wfweb-local:latest youruser/wfweb:9700-memfix
docker push youruser/wfweb:9700-memfix
```

```yaml
    image: youruser/wfweb:9700-memfix   # pull_policy back to default
```
