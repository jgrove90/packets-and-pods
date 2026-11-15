# packetsandpods
Hugo blog

## Local Development

From the site directory:

```bash
cd packetsandpods
hugo server -D
```
Rebuild via:

```hugo --minify```

## CI/CD

The GitHub Actions workflow in [.github/workflows/deploy-blog.yml](.github/workflows/deploy-blog.yml) runs on pushes to `main` and publishes a container image to GHCR.

## Setup

The theme is a git submodule:

```bash
git clone --recurse-submodules <repo-url>
# or, in an existing clone:
git submodule update --init --recursive
```

Requires Hugo extended (CI uses 0.162.1).

## Build & deploy notes

`public/` is not tracked. CI builds the site with Hugo, then the `Dockerfile` copies `public/` into an nginx image, so run `hugo --minify` first if building the image locally:

```bash
cd packetsandpods
hugo --minify
docker build -t packetsandpods .
```

Images are published to `ghcr.io/<owner>/<repo>` tagged `latest`, the commit sha, and any `v*` tag.
