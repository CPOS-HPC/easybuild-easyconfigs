# EasyBuild repository guidance

Read [SKILL.md](SKILL.md) completely before creating, changing, reviewing, or validating an easyconfig or patch.
Treat it as the repository-specific workflow and convention reference.

- Inspect the closest current easyconfigs before editing; prefer local precedent over remembered conventions.
- Preserve unrelated worktree changes and limit edits to the requested recipes and supporting files.
- Use `apply_patch` for edits and `rg` for repository searches.
- Resolve dependencies with EasyBuild's robot and run an isolated build for new recipes or structural easyblock changes
  when feasible.
- Review the final diff, run `git diff --check`, and report exactly which validation completed.
- Do not commit or push unless the user explicitly requests it.
