# reusable-workflows

Reusable GitHub Actions workflows.

## `docker-build.yml`

Builds a Docker image and pushes it to a private registry and (optionally) to Docker Hub.

| Trigger                    | Image tags                                                 |
|----------------------------|------------------------------------------------------------|
| push to `feature/<name>`   | `:feature-<name>`, `:feature-<name>-<sha7>`                |
| new tag `<tag>`            | `:<tag>` (and `:latest` if `latest-on-tag: true`)          |
| anything else              | job fails, nothing is pushed                               |

Characters not allowed in Docker tags (e.g. `/`) are replaced with `-`.

### Secrets

Add them in the application repo: Settings → Secrets and variables → Actions → New repository secret.

| Secret               | Required | Description                                                   |
|----------------------|----------|---------------------------------------------------------------|
| `REGISTRY_URL`       | yes      | Private registry incl. path, e.g. `registry.example.com/team` |
| `REGISTRY_USERNAME`  | yes      | Private registry user                                         |
| `REGISTRY_PASSWORD`  | yes      | Private registry password / token                             |
| `DOCKERHUB_USERNAME` | no       | Docker Hub user; if set, the image is also pushed to `docker.io/<user>/<image>` |
| `DOCKERHUB_TOKEN`    | no       | Docker Hub access token (Account settings → Personal access tokens, Read & Write) |

The private registry address is only kept in a secret, so it is masked (`***`) in the
workflow logs, and the job summary only says "private registry".

### Usage

`.github/workflows/docker.yml` in the application repo:

```yaml
name: Docker

on:
  push:
    branches: ["feature/**"]
    tags: ["v*"]

jobs:
  docker:
    uses: jvincze84/reusable-workflows/.github/workflows/docker-build.yml@main
    with:
      image-name: my-app          # optional, default: repository name
    secrets:
      REGISTRY_URL: ${{ secrets.REGISTRY_URL }}
      REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
      REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
```

### Inputs

| Input                 | Default                 | Description                                  |
|-----------------------|-------------------------|----------------------------------------------|
| `image-name`          | caller repository name  | Image name (lowercased)                      |
| `dockerhub-namespace` | `DOCKERHUB_USERNAME`    | Docker Hub user/organization to push to      |
| `context`             | `.`                     | Build context                                |
| `dockerfile`          | `<context>/Dockerfile`  | Dockerfile path                              |
| `target`              | last stage              | Dockerfile stage to build                    |
| `platforms`           | `linux/amd64`           | e.g. `linux/amd64,linux/arm64`               |
| `latest-on-tag`       | `false`                 | Also push `:latest` for tag builds           |
| `feature-prefix`      | `feature/`              | Branch prefix treated as feature branch      |
| `dockerhub-readme`    | `false`                 | Update the Docker Hub repository overview (see below) |
| `dockerhub-readme-file` | `README.md`           | Markdown file for the overview (path from repo root) |
| `dockerhub-short-description` | empty (unchanged) | Docker Hub short description (max 100 chars) |

Outputs: `tags` (tag names without registry, newline separated), `digest`.

### Docker Hub repository overview (optional)

With `dockerhub-readme: true`, release (tag) builds also replace the Docker Hub
"Repository overview" with `dockerhub-readme-file` (via
[peter-evans/dockerhub-description](https://github.com/peter-evans/dockerhub-description)).
Feature-branch builds never touch it. Relative links and images in the Markdown are
rewritten to absolute GitHub URLs. Skipped when the Docker Hub secrets are not set.

- `DOCKERHUB_TOKEN` must have **Read, Write, Delete** scope (Read & Write is not enough
  to edit the repository description).
- Docker Hub limits the overview to 25,000 bytes.

```yaml
    with:
      image-name: my-app
      dockerhub-readme: true
      dockerhub-short-description: "One-line description"   # optional
```
