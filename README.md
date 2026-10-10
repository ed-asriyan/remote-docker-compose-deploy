# remote-docker-compose-deploy

Reusable GitHub Actions workflows that build the images of a Docker Compose project on a GitHub runner and deploy them to a server over SSH. No container registry is involved: the images travel to the server as a tarball.

- `build.yml` builds every service that has a `build:` section and uploads the images as a workflow artifact.
- `deploy.yml` downloads that artifact, copies the compose file (and a generated `.env`) to the server, loads the images and runs `docker compose up -d`.

The two are separate so that one build can be deployed to several hosts.

## How it works

1. `build.yml` builds the images with `docker buildx bake`, tags them with the commit sha and saves them to `images.tar`.
2. `deploy.yml` logs in to the server with SSH, wipes `~/apps/<project>` (the `name:` of the compose file), uploads the compose file, the `.env` and the images there.
3. On the server: `docker load`, `docker compose up -d --no-build --remove-orphans`, then it prunes stopped containers and the previous images of the same project. If Docker is missing, it is installed (this needs root).

Named volumes and bind mounts outside `~/apps/<project>` survive a deploy; the directory itself does not.

## Requirements

**Compose file**

- A top-level `name:`. It names the directory on the server and the compose project.
- Every service with `build:` has an `image:` that contains `${TAG}`, for example `image: myapp/web:${TAG:-latest}`. `TAG` is the commit sha in both workflows.

**Server**

- Reachable over SSH with a key. Docker with the compose plugin (or `docker-compose`) is installed if absent, provided the user is root.

**Repository settings**

| Name | Kind | Purpose |
|------|------|---------|
| `SSH_PRIVATE_KEY` | secret | private key to log in with |
| `SSH_USERNAME` | secret | user to log in as |
| `SSH_HOST` | secret | host to deploy to, unless you pass the `host` input |

A GitHub environment (`production` by default) may hold them instead of the repository.

## Usage

```yaml
name: CD

on:
  push:
    branches: [master]
  workflow_dispatch:

concurrency:
  group: CD
  cancel-in-progress: true

jobs:
  build:
    uses: ed-asriyan/remote-docker-compose-deploy/.github/workflows/build.yml@v1
    with:
      compose-file: docker-compose.yml

  deploy:
    needs: build
    uses: ed-asriyan/remote-docker-compose-deploy/.github/workflows/deploy.yml@v1
    with:
      compose-file: docker-compose.yml
      host: deploy.example.com
    secrets: inherit
```

### Environment file

`env-file` is a template for the `.env` written next to the compose file. `${NAME}` is replaced with the secret or variable `NAME`; a placeholder that matches nothing is left as written.

```yaml
    with:
      compose-file: docker-compose.yml
      env-file: |
        BOT_TOKEN=${BOT_TOKEN}
        LOG_LEVEL=info
    secrets: inherit
```

The `ENV_FILE` secret, if set, is appended to the file verbatim.

### Several hosts

```yaml
  deploy:
    needs: build
    strategy:
      matrix:
        host: [a.example.com, b.example.com]
    uses: ed-asriyan/remote-docker-compose-deploy/.github/workflows/deploy.yml@v1
    with:
      compose-file: docker-compose.yml
      host: ${{ matrix.host }}
    secrets: inherit
```

### Several compose files

Run `build.yml` and `deploy.yml` once per file. The images artifact is named after the compose file, so the pairs do not collide.

## Reference

### `build.yml`

| Input | Required | Description |
|-------|----------|-------------|
| `compose-file` | yes | path to the compose file |

| Output | Description |
|--------|-------------|
| `artifact` | name of the uploaded images artifact |

### `deploy.yml`

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `compose-file` | yes | | the same path that was passed to `build.yml` |
| `host` | no | `SSH_HOST` secret | host to deploy to |
| `port` | no | `22` | SSH port |
| `environment` | no | `production` | GitHub environment the job runs in |
| `env-file` | no | empty | `.env` template, see above |

| Secret | Description |
|--------|-------------|
| `SSH_PRIVATE_KEY` | required in practice, see the next section |
| `SSH_USERNAME` | required in practice |
| `SSH_HOST` | used when the `host` input is empty |
| `ENV_FILE` | appended to the `.env` |

The secrets are declared optional only so that `secrets: inherit` is accepted when they live in the GitHub environment; the first step of the job fails with a clear message if any of the three is missing.

## Secrets and security

- A called workflow runs with the permissions of your repository and sees the secrets you give it. With `secrets: inherit` that is **all** of them, because `env-file` placeholders can name any secret. Pin the workflow to a release tag (`@v1`) or, for the strictest setting, to a commit sha, never to a branch.
- Secrets stored in a GitHub **environment** reach `deploy.yml` only through `secrets: inherit`: a caller job that uses a reusable workflow cannot select an environment itself, so it cannot read the environment's secrets to pass them explicitly. Repository and organization secrets can be passed one by one under `secrets:`.
- `.env` is written with mode `600` on the runner; it is uploaded over SSH and is not printed to the log.
- `inherit` is documented for workflows in the same organization or enterprise; if your account setup does not allow it, pass `SSH_PRIVATE_KEY`, `SSH_USERNAME` and (for the host) the `host` input explicitly, and put the `.env` content in the `ENV_FILE` secret.

## Releases

Releases are git tags: `v1` moves forward with every compatible change, `v1.2.3` never moves.

## License

MIT
