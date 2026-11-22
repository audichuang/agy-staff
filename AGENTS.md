# agy-staff

`skills/agy/` is the install payload: aghub-cli copies the whole directory to `~/.aghub/agy` on every
machine and overwrites it on update, so edit only here and keep anything that isn't payload
(notes, eval workspaces, a second `SKILL.md`) out of it.

- Tests: `node --test tests/*.test.mjs` (CI runs the same).
- Upstream `keli-wen/agy-staff` diverged at 0.7.3: we ship one self-contained skill, not its
  plugin and persona skills. Port only model and agy-compatibility changes, never merge
  (`../agy-accounts/decisions.md`, 2026-11-13). `docs/releases/` (all up to v0.7.3) are upstream's
  plugin release notes kept as history; their install and persona steps don't apply here.
