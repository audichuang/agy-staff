# agy-staff

`skills/agy/` is the install payload: aghub-cli copies the whole directory to `~/.aghub/agy` on every
machine and overwrites it on update, so edit only here and keep anything that isn't payload
(notes, eval workspaces, a second `SKILL.md`) out of it.

- No code ships since 0.8.0: the skill is text, and agy starts only through Orca (agy-accounts
  `decisions.md`, 2026-11-26/27). Don't bring back a headless runner or account switching.
- After changing model defaults or the "Pick a model" guidance, have a fresh agent read only
  `skills/agy/SKILL.md` and pick task type and model for about ten everyday requests (survey, PR review,
  typo, multi-hour migration, "strongest model", low quota). Nothing automated catches a wrong pick:
  in 0.7.10 that check found a multi-hour task sent to one Carbon run and low quota dropping a
  security review to Hydrogen.
- Upstream `keli-wen/agy-staff` diverged at 0.7.3: we ship one self-contained skill, not its
  plugin and persona skills. Port only model and agy-compatibility changes, never merge
  (`../agy-accounts/decisions.md`, 2026-11-13). `docs/releases/` (all up to v0.7.3) are upstream's
  plugin release notes kept as history; their install and persona steps don't apply here.
