# Reusable workflows

The reusable workflows live in [`.github/workflows/`](../.github/workflows/), because GitHub can only call workflows stored in that folder.

| File | Purpose | Inputs | Secrets | Outputs |
|---|---|---|---|---|
| `build-and-test.yml` | Installs dependencies and runs tests | `node-version`, `test-command` | None | `node-version` |
| `docker-build-push.yml` | Builds and pushes a Docker image tagged with the commit SHA | `image-name`, `registry`, `registry-username` | `registry_password` | `image`, `digest` |
| `deploy-to-kubernetes.yml` | Deploys a Helm chart, rolling back on failure | `chart-path`, `namespace`, `values-file`, `release-name`, `image` | `kubeconfig` | `revision` |

See [`../caller-example/pipeline.yml`](../caller-example/pipeline.yml) for usage, and [`../DESIGN.md`](../DESIGN.md) for versioning, breaking changes and secrets.
