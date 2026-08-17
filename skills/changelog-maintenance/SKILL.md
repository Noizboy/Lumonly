---
name: changelog-maintenance
description: >
  Maintains the Lumonly CHANGELOG.md whenever a project change is made.
  Use for every feature, fix, behavior change, security change, dependency or infrastructure change,
  connector change, documentation change that affects usage, and release preparation in this repository.
---

# Lumonly Changelog Maintenance

Apply this skill to every relevant change in this repository. Update the root `CHANGELOG.md` in the same patch as the changed files; do not defer it to a later task.

## Procedure

1. Read the current root `CHANGELOG.md` before editing it.
2. Identify the user-visible or operational impact of the change.
3. Add one concise entry under `## [Unreleased]` in the appropriate section.
4. Preserve existing entries, heading order and release history. Do not rewrite or remove historical notes.
5. Verify that the changelog entry accurately describes the implemented change, without claiming work that is only planned or unverified.

## Sections

Use these headings, creating a heading under `[Unreleased]` only when it is needed:

- `### Added` — new capability, connector, dashboard, endpoint, documentation or integration.
- `### Changed` — modified behavior, architecture, configuration or supported scope.
- `### Fixed` — corrected defect or regression.
- `### Security` — authentication, authorization, redaction, secret handling or vulnerability improvement.
- `### Deprecated` — capability still available but scheduled for removal.
- `### Removed` — deleted capability or supported behavior.

## Entry style

- Write in English, using past tense or concise noun phrasing.
- Describe the result and scope, not internal implementation minutiae.
- Mention a feature or document name when it helps users find the change.
- Use one bullet per independently meaningful change.
- Do not add an entry for formatting-only edits, temporary diagnostics or changes that leave no user, operator or maintainer impact.

Good:

```md
### Added

- Added authenticated OTLP ingestion for logs, metrics, and traces per project.
```

```md
### Security

- Authorization headers and cookies are redacted before logs are exported.
```

Avoid:

```md
- Updated three files.
- Added the complete observability system.
```

## Releases

Only create a versioned release heading when the user explicitly requests a release/version. Move the applicable `[Unreleased]` entries beneath a heading such as `## [1.0.0] - YYYY-MM-DD`, leaving an empty `[Unreleased]` section at the top. Do not invent version numbers or release dates.

## Completion check

- [ ] `CHANGELOG.md` updated in the same change when the modification is relevant.
- [ ] Entry is placed in the correct section under `[Unreleased]`.
- [ ] Entry matches verified behavior and does not expose secrets or sensitive operational details.
