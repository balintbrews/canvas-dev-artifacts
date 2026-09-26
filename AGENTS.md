# Repository conventions

- Store every artifact linked to a GitLab work item under
  `<gitlab-work-item-id>/`. Numeric directory names refer to GitLab work items.
- Store a standalone artifact that has no work item under
  `YYYY-MM-DD-descriptive-slug/`, dated to the day only (no time). Check that
  the directory does not exist yet; on a collision, append a short numeric
  suffix (for example `2026-09-26-descriptive-slug-2/`). Never overwrite an
  existing artifact.
- Expect multiple artifacts and artifact types per directory. Use descriptive
  filenames; group related files in subdirectories as needed.
- Commit messages: `<type>(<gitlab-work-item-id>): <short description>` for
  issue-linked changes (e.g. `docs(3592101): add testing guide`), and
  `<type>: <short description>` for standalone artifacts (e.g.
  `docs: add astro to canvas headless guide`). Do not invent an issue ID.
