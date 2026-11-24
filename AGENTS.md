# agy-staff

`skills/agy/` is the install payload: aghub-cli copies the whole directory to `~/.aghub/agy` on every
machine and overwrites it on update, so edit only here and keep anything that isn't payload
(notes, eval workspaces, a second `SKILL.md`) out of it.

- Tests: `node --test tests/*.test.mjs` (CI runs the same).
- After changing model defaults or the "Pick a model" guidance, have a fresh agent read only
  `skills/agy/SKILL.md` and pick mode and model for about ten everyday requests (survey, PR review,
  typo, multi-hour migration, "strongest model", low quota). The tests can't catch a wrong pick:
  in 0.7.10 that check found a multi-hour task sent to one Carbon run and low quota dropping a
  security review to Hydrogen.
- Upstream `keli-wen/agy-staff` diverged at 0.7.3: we ship one self-contained skill, not its
  plugin and persona skills. Port only model and agy-compatibility changes, never merge
  (`../agy-accounts/decisions.md`, 2026-11-13). `docs/releases/` (all up to v0.7.3) are upstream's
  plugin release notes kept as history; their install and persona steps don't apply here.
