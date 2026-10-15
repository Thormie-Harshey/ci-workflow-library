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

5. **Support window.** `v1` keeps receiving backported security fixes until the end-of-support date, so no team is punished for not moving yet.
6. **Teams migrate themselves** by changing `@v1` to `@v2` and following the guide.
