# Upstream

This repository is an NI-governed fork of the upstream Vale-MCP project.

## Adopted upstream revision

- Upstream repository: ChrisChinchilla/Vale-MCP
- Upstream commit: `e196ba7e5cbbf11c2704d208f38afff40d62d877`
- Last reviewed: 2026-11-05

Upstream changes are not synchronized directly to `main`.
Upstream revisions are evaluated on a `sync-upstream-*` branch and
incorporated through a reviewed pull request.

# Non-upstream (NI-only) changes

Some changes exist only in this fork and have no corresponding upstream
commit — for example, deployment tooling, internal integrations, or
transport options added for NI's use cases. These changes are not expected
to be contributed to or synced from `ChrisChinchilla/Vale-MCP`.

## Branching and review

- Use a `ni-feature/<short-description>` branch name (for example,
  `ni-feature/streamable-http`). This keeps non-upstream work clearly
  distinct from `sync-upstream-*` branches, which are reserved for
  evaluating upstream revisions.
- Non-upstream changes follow the same `main` branch protection as any
  other change: a pull request with at least one approval.
- If a change adds or modifies network-facing surface area (new listeners,
  endpoints, authentication, or data handling), read [SECURITY.md](./SECURITY.md) policy and report any security issues as instructed.

## Traceability

To ensure that a future upstream sync can tell NI-only code apart from
upstream-inherited code at a glance, do the following:

- Tag the pull request with the `ni-only` label.
- Add an `NI-Change: <short description>` trailer to the merge/squash
  commit message, alongside the existing `Signed-off-by` trailer.
- Mark new NI-only files with a header comment, for example:
  `// NI-ONLY: not present upstream; see UPSTREAM.md#non-upstream-ni-only-changes`.
  For NI-only edits inside a file that is otherwise synced from upstream,
  wrap the changed block in matching `// NI-ONLY: start` / `// NI-ONLY: end`
  comments.
- Record the change in the table below, including every upstream-shared
  file it touches, so a sync reviewer knows where to expect conflicts.
- Give the change its own entry in [CHANGELOG.md](../CHANGELOG.md), under a
  dedicated "NI-only changes" heading, separate from upstream-inherited
  entries.

| Feature | Files touched (shared with upstream in bold) | Introduced in | Notes for future syncs |
|---|---|---|---|
| _(example)_ Streamable HTTP transport | `src/http.ts` (new), **`src/index.ts`**, **`package.json`** | PR #TBD | `index.ts` was refactored to export `createValeServer()`/`initializeValeConfig()`; re-check this export shape against any upstream `index.ts` changes during the next sync. |
