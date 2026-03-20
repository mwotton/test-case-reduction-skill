# Test-case reduction skill

This is a skill for teaching agents to use test-case reducers, especially [shrinkray](https://github.com/DRMacIver/shrinkray).

It's currently extremely experimental and mostly untested, so you'll likely run into problems when you try it in practice.
When you do, please ask your agent to reflect on the problems it ran into, refine the skill, and open a pull request for you.

Recent practical lessons that are now encoded here:

- For stateful tools, the interestingness test should create fresh temp state on every run and pass explicit paths for any DB/cache/worktree inputs.
- For structured inputs like JSONL, reduce whole records and normalize irrelevant fields before asking shrinkray to do generic byte-level cleanup.
- When shrinkray appears idle, inspect the candidate artifact itself; some runs plateau for a long time without producing useful UI output.
