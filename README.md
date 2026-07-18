# packetsandpods
Hugo blog

## Local Development

From the site directory:

```bash
cd packetsandpods
hugo server -D
```

## CI/CD

The GitHub Actions workflow in [.github/workflows/deploy-blog.yml](.github/workflows/deploy-blog.yml) runs on pushes to `main` and publishes a container image to GHCR.
