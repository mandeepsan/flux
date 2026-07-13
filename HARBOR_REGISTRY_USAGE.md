# Harbor Registry Usage

Harbor is available at `harbor.otroid.net`.

## Public Images

Use the `public` project for images that anyone can pull without credentials:

```bash
docker pull harbor.otroid.net/public/nginx-demo:latest
```

Pushing still requires a Harbor login with a user in the Pocket ID `devops` group.

## Private Images

Use the `private` project for images that require Docker or Helm authentication:

```bash
docker login harbor.otroid.net -u <pocketid-username>
docker pull harbor.otroid.net/private/app:latest
```

When using Pocket ID/OIDC, use the Harbor CLI secret from your Harbor user profile as the Docker password. The browser login proves your identity to Harbor; the CLI secret is what Docker and Helm use for registry API requests.

## Push Flow

```bash
docker login harbor.otroid.net -u <pocketid-username>
docker tag nginx:latest harbor.otroid.net/public/nginx-demo:latest
docker push harbor.otroid.net/public/nginx-demo:latest
```

For private images, replace `public` with `private`.

## Vulnerability Scanning

Trivy is enabled inside Harbor. The `public` and `private` projects are configured to scan images automatically on push, and Harbor runs a daily scan-all schedule at `02:00`.

Pull blocking is intentionally disabled for now. After scan results look sane, the first enforcement step should be blocking pulls only for `Critical` vulnerabilities.
