# Docker Build & Push

Builds a Docker image and pushes it to any container registry. Generic for
single-app repos and monorepos alike: tags the image with the short commit
SHA and `latest` (plus any extra tags you want), and exposes the resulting
image refs as outputs so downstream jobs (e.g. a deploy job) can consume
them without re-deriving the name.

## What it does

1. (Optional) Checks out the repository — skip this if your job already did.
2. Logs in to the registry via `docker/login-action`.
3. Derives the image name from `registry` + `repository` + `app-name` (or
   lets you override it outright with `image-name`).
4. Builds the image with `docker build`, tagging it with:
   - `latest`
   - the short (7-char) commit SHA
   - any `extra-tags` you pass in
5. Pushes all of those tags (unless `push: false`).

## Usage

### Single-app repo

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.build.outputs.image }}
    steps:
      - uses: alexandergv2117/actions/docker-build-push@main
        id: build
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          # app-name left empty -> image named ghcr.io/org/repo
```

### Monorepo (multiple apps, each with its own Dockerfile)

```yaml
jobs:
  build-api:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.build.outputs.image }}
    steps:
      - uses: alexandergv2117/actions/docker-build-push@main
        id: build
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          app-name: api
          dockerfile: apps/api/Dockerfile
          context: .
          # -> image: ghcr.io/org/repo/api:<sha>

  build-web:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image: ${{ steps.build.outputs.image }}
    steps:
      - uses: alexandergv2117/actions/docker-build-push@main
        id: build
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          app-name: web
          dockerfile: apps/web/Dockerfile
          build-args: |
            NEXT_PUBLIC_API_URL=https://api.example.com
            NEXT_PUBLIC_APP_ENV=production
          # -> image: ghcr.io/org/repo/web:<sha>
```

### Build-only (no push), e.g. to validate a Dockerfile on a PR

```yaml
- uses: alexandergv2117/actions/docker-build-push@main
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
    push: false
```

### Consuming the output image in a later job

```yaml
jobs:
  build:
    # ... as above ...
    outputs:
      image: ${{ steps.build.outputs.image }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image }}"
```

## Inputs

| Input         | Required | Default               | Description |
|---------------|----------|------------------------|-------------|
| `registry`    | yes      | —                       | Container registry host (e.g. `ghcr.io`, `docker.io`, `123456789.dkr.ecr.us-east-1.amazonaws.com`). |
| `username`    | yes      | —                       | Registry username. |
| `password`    | yes      | —                       | Registry password or token. **Pass as a secret.** |
| `app-name`    | no       | `""`                    | App name, appended to the image name (`registry/repository/app-name`). Useful for monorepos. Leave empty for a single-app repo to name the image after the repository itself. |
| `repository`  | no       | `${{ github.repository }}` | Namespace to put the image under (e.g. `owner/repo`). Lower-cased automatically (registries reject uppercase). |
| `image-name`  | no       | `""`                    | Override the full image name (`registry/namespace/app`) instead of deriving it from `registry` + `repository` + `app-name`. Takes priority over both when set. |
| `dockerfile`  | no       | `"Dockerfile"`          | Path to the Dockerfile, relative to the repo root. For monorepos set the full path (e.g. `apps/api/Dockerfile`). |
| `context`     | no       | `"."`                   | Docker build context. |
| `build-args`  | no       | `""`                    | Extra build args, one `KEY=VALUE` per line. Passed through as `--build-arg`. Useful for baking public env vars (e.g. `NEXT_PUBLIC_*`) into a frontend image at build time. |
| `extra-tags`  | no       | `""`                    | Extra tags to apply, one per line, in addition to `latest` and the short SHA (e.g. a release tag or `stable`). |
| `push`        | no       | `"true"`                | Whether to push the built image to the registry. Set `false` to only build (e.g. to validate a Dockerfile on a PR). |
| `checkout`    | no       | `"true"`                | Whether this action should check out the repository first. Set `false` if the calling job already checked it out (e.g. with a non-default ref). |

## Outputs

| Output         | Description |
|----------------|-------------|
| `image`        | Full image ref tagged with the short commit SHA (`registry/namespace/app:sha`). This is what you normally hand to a deploy job. |
| `image-latest` | Full image ref tagged with `latest` (`registry/namespace/app:latest`). |
| `image-name`   | Image name without a tag (`registry/namespace/app`). |
| `short-sha`    | The short (7-char) commit SHA used for the sha tag. |

## Image name resolution order

1. `image-name` if set — used verbatim.
2. Otherwise, if `app-name` is set — `registry/repository(lowercased)/app-name`.
3. Otherwise — `registry/repository(lowercased)` (good for single-app repos).

## Notes

- `repository` is lower-cased automatically before use; registries like GHCR
  reject uppercase in image names, so you don't need to do this yourself
  even if `github.repository` has mixed case.
- The short SHA is always derived from `github.sha` (`GITHUB_SHA::7`), i.e.
  the commit that triggered the workflow — not from the Dockerfile's
  contents or any input.
- Required permissions on the job: `contents: read` (to checkout) and
  `packages: write` (to push to GHCR). Adjust if you use a different
  registry.
- Requires `jq`-free bash only — no extra tool dependencies beyond Docker
  itself and what `docker/login-action` needs.
