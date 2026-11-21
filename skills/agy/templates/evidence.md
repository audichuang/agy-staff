## Finding evidence

- Start with the task's paths, symbols, or diff. Prefer `rg -l` or `rg -n` to locate relevant code.
- Read the relevant line ranges in large files. Follow callers, tests, and configuration when they affect the task.
- Scope command output by path and pattern. Refine truncated results to retrieve the missing evidence.
