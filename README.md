# Design: Reusable GitHub Actions Workflow Library

## Purpose

Fifteen teams maintained copy-pasted pipelines, so a fix to the Docker build step had to be applied in fifteen repositories. This library holds three reusable workflows in `.github/workflows/` that every team calls. A fix is made once, released once, and reaches every team.

## 1. Versioning

Releases follow semantic versioning (MAJOR.MINOR.PATCH):

- **PATCH** (`v1.2.1`): bug or security fixes. Callers change nothing.
- **MINOR** (`v1.3.0`): new optional inputs or outputs. Existing callers keep working.
- **MAJOR** (`v2.0.0`): breaking changes, such as a renamed input or a new required input.

Every release gets an exact tag (`v1.2.1`). The platform team also moves a major tag (`v1`) to the newest release in that line.

| Reference | Example | Gets fixes automatically | Trade-off |
|---|---|---|---|
| Major tag | `@v1` | Yes | Trusts that the tag is only moved by the platform team |
| Exact tag | `@v1.2.1` | No | Fixes need a manual bump |
| Commit SHA | `@<40-character SHA>` | No | Cannot be changed or tampered with; most secure |
| Branch | `@main` | Yes | Not allowed. Any commit, including a broken one, reaches every team at once |

**My simple recommendation.** Most teams pin to `@v1`, so security fixes such as a Docker build change reach them with no action. Teams with strict supply-chain requirements pin to the full commit SHA, add the version as a comment, and enable Dependabot to raise a pull request for each new release:

```yaml
uses: Thormie-Harshey/ci-workflow-library/.github/workflows/build-and-test.yml@<40-character SHA> # v1.2.1
```
## 2. Handling breaking changes

The aim is that no team's pipeline breaks because of a change they did not choose.

### Avoid the break where possible

Many changes can be made backward compatible. To rename `node-version` to `node_version`, release `v1.3.0` (MINOR) that accepts both names, uses the new one if given, and falls back to the old one:

```yaml
node-version: ${{ inputs.node_version || inputs.node-version }}
```

Callers using the old name see a deprecation warning but keep working. The old name is only removed later, in a major release.

### When can a break not be avoided?

1. **Release side by side.** Publish `v2.0.0` and a new `v2` tag. The `v1` tag never moves to a breaking release, so teams on `@v1` are unaffected.
2. **Warn in v1.** Every `v1` run prints a deprecation notice with the end-of-support date:

```yaml
   - name: Deprecation notice
     run: echo "::warning::v1 is deprecated. Move to v2 before March 1, 2027. See the migration guide."
```

3. **Support window.** `v1` keeps receiving backported security fixes until the end-of-support date, so no team is punished for not moving yet.
4. **Teams migrate themselves** by changing `@v1` to `@v2` and following the guide.

## 3. Passing secrets securely

### Explicit passing, never inherit

Each reusable workflow declares exactly the secrets it accepts, and callers pass only those:

| Workflow | Secret | Purpose |
|---|---|---|
| `build-and-test.yml` | None | |
| `docker-build-push.yml` | `registry_password` | Log in to the image registry |
| `deploy-to-kubernetes.yml` | `kubeconfig` | Access the target cluster |

```yaml
secrets:
  kubeconfig: ${{ secrets.KUBECONFIG }}
```

`secrets: inherit` is not used. It hands every caller secret to shared code, so one faulty or compromised release could read any secret in all fifteen teams.

### Protections inside the workflows

- **Secrets are never inputs.** Inputs are written in plain text in caller files and printed in logs. The registry username is an input because it is not sensitive.
- **Masking.** GitHub replaces secret values with `***` in logs.
- **Environment variables, not inline expressions.** Values reach shell commands through `env:`, never `${{ }}` inside `run:`, which prevents script injection.
- **Least-privilege permissions.** Workflows default to `contents: read`. The caller grants `packages: write` only to the build-push job.
- **Short-lived files.** The kubeconfig is written with `chmod 600` on a temporary runner that is destroyed when the job ends.

### Masking is not enough on its own

Masking only hides the exact secret text. An encoded copy, or a secret sent to an outside server, is not caught. The library therefore relies on defence in depth:

- The library repository uses branch protection and a `CODEOWNERS` file, so every change is reviewed by the platform team.
- Version pinning means teams only run reviewed releases, never unreviewed commits.
- `GITHUB_TOKEN` is created per run and expires when the job ends.
- Production secrets are stored in a GitHub environment with required reviewers, so they unlock only after approval.
- For cloud clusters such as EKS, OIDC is the recommended next step. The runner receives short-lived cloud access, so no long-lived kubeconfig or password is stored at all.

## 4. Design decisions

- **Workflow location.** GitHub can only call workflows stored in `.github/workflows/`, so the three files live there. `reusable-workflows/` holds a README pointing to them. Keeping one copy of each file avoids recreating the copy-paste problem this library solves.
- **"Runs tests on push".** A reusable workflow can only be triggered by `workflow_call`. The caller holds `on: push` and calls `build-and-test.yml` on every push.
- **Extra inputs beyond the brief.**
  - `registry-username`: login needs a username. It is not sensitive, so it is an input, not a secret.
  - `release-name`: Helm needs a name for each release.
  - `image` (optional): passes the image built by `docker-build-push.yml` into the deploy, so the image that was tested is the one deployed. It overrides the values file using `--set`.
- **Outputs.** `build-and-test.yml` returns `node-version`, `docker-build-push.yml` returns `image` and `digest`, and `deploy-to-kubernetes.yml` returns the Helm `revision`.
- **Pinned tools.** Helm is pinned to v3.16.4, because Helm 4 renamed the `--atomic` flag.
