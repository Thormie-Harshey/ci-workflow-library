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
