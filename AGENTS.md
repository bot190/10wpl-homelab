# Agent Instructions

## Helm Development

- Before starting development on any application, **always check the upstream sources for the latest available Helm chart version and container image version**. Do not rely on versions already present in the repository or on cached knowledge.
- Define Helm chart sources as OCI registries.
- Keep each chart's OCI registry definition alongside the application that consumes it.

## Git Commits

- Use the Conventional Commits standard for commit messages.
- Format commit subjects as `<type>(<scope>): <description>` when a scope is useful, or `<type>: <description>` when no clear scope applies.
- Prefer common types such as `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, and `build`.
- Include a reasonably detailed commit body for non-trivial changes. The body should explain what changed, why it changed, and any important operational or validation notes.
- Keep the subject concise and imperative. Put detail in the body instead of making the subject long.
