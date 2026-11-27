# agy-staff

`skills/agy/` is the install payload: aghub-cli copies the whole directory to `~/.aghub/agy` on every
machine and overwrites it on update, so edit only here and keep anything that isn't payload
(notes, eval workspaces, a second `SKILL.md`) out of it. The one exception is
`evals/must_keep.json`: skillgenie's lint demands it when the description changes; synthetic cases only.

- This repo is public: no user transcripts or verbatim requests, private paths, Doppler
  project/config names or internal host names. Eval sets built from real sessions stay outside the
  repo (both slipped in once on 2026-11-27).
- `gemini-api-run` in SKILL.md comes from the private agent-config repo (`machine/bin/`, deployed by
  its `sync.sh`); rename or change its arguments in both repos together.
- After changing an Orca launch or collect command, run one Hydrogen/low task on a Mac following
  SKILL.md literally. Dry-run evals passed while `--screen`, `--wait-submit`, `selector_not_found`
  and a missing `Status:` line in the result file were all wrong.
- No code ships since 0.8.0: the skill is text, and agy starts only through Orca (agy-accounts
  `decisions.md`, 2026-11-26/27). Don't bring back a headless runner or account switching.
- After changing model defaults or the "Pick a model" guidance, have a fresh agent read only
  `skills/agy/SKILL.md` and pick task type and model for about ten everyday requests (survey, PR review,
  typo, multi-hour migration, "strongest model", low quota). Nothing automated catches a wrong pick:
  in 0.7.10 that check found a multi-hour task sent to one Carbon run and low quota dropping a
  security review to Hydrogen.
- Upstream `keli-wen/agy-staff` diverged at 0.7.3: we ship one self-contained skill, not its
  plugin and persona skills. Port model and agy-compatibility changes, plus worker-template and lead guidance
  that upstream's benchmark backs (its 0.8.0 templates and lead points 1/2/6 came over 2026-11-27,
  with "no further delegation" narrowed so agy's own subagents stay allowed); never merge
  (`../agy-accounts/decisions.md`, 2026-11-13). `docs/releases/` (all up to v0.7.3) are upstream's
  plugin release notes kept as history; their install and persona steps don't apply here.
