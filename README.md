# Gitea Link Package Action

A focused Gitea Action that links Docker packages to repositories in Gitea after they've been built and pushed.

## Features

- Links pushed Docker packages to repositories via Gitea API
- Automatic HTTP redirect handling
- Graceful error handling with clear status messages
- Fail-fast behavior for linking errors
- Debug mode for troubleshooting
- Single responsibility: only handles package linking

## Usage

Use this action after building and pushing your Docker image:

```yaml
- name: Build and push Docker image
  id: build
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}

- name: Link package to repository
  uses: aroberts/gitea-link-package@v1
  with:
    registry: ${{ env.REGISTRY }}
    package-owner: ${{ gitea.repository_owner }}
    package-name: ${{ gitea.repository }}
    repository-name: ${{ gitea.repository }}
    token: ${{ secrets.PACKAGE_TOKEN }}
```

## Required Inputs

| Input | Description |
|-------|-------------|
| `registry` | Container registry URL (e.g., `gitea.example.com`) |
| `package-owner` | Package owner (usually `${{ gitea.repository_owner }}`) |
| `package-name` | Package name (usually `${{ gitea.repository }}`) |
| `repository-name` | Repository name to link package to |
| `token` | Authentication token for registry and API access |

## Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `debug` | Enable debug mode for verbose output | `false` |

## Debug Mode

Enable debug mode for troubleshooting by setting `debug: true`. This provides:

- Input parameter values (with token masked for security)
- Constructed API URL
- HTTP redirect information (count and final URL)
- Raw HTTP response body
- Detailed curl command execution trace

Example with debug enabled:
```yaml
- name: Link package to repository
  uses: aroberts/gitea-link-package@v1
  with:
    registry: ${{ env.REGISTRY }}
    package-owner: ${{ gitea.repository_owner }}
    package-name: ${{ gitea.repository }}
    repository-name: ${{ gitea.repository }}
    token: ${{ secrets.PACKAGE_TOKEN }}
    debug: true
```

## Error Handling

The action handles package linking with clear status reporting:

- **HTTP 2xx**: Success, package linked
- **HTTP 409**: Package already linked (treated as success)
- **Other errors**: Action fails with error details

## Environment Requirements

- Gitea instance with package registry enabled
- Authentication token with package and API permissions

## Complete Example Workflow

```yaml
name: Build & Push Docker Image

on:
  push:
    branches: [main]

env:
  REGISTRY: gitea.example.com
  IMAGE_NAME: ${{ gitea.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ gitea.repository_owner }}
          password: ${{ secrets.PACKAGE_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Link package to repository
        uses: aroberts/gitea-link-package@v1
        with:
          registry: ${{ env.REGISTRY }}
          package-owner: ${{ gitea.repository_owner }}
          package-name: ${{ gitea.repository }}
          repository-name: ${{ gitea.repository }}
          token: ${{ secrets.PACKAGE_TOKEN }}
          debug: true  # Optional: enable for troubleshooting
```
