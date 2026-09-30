# Claude Sandbox Methodology

A way of building real applications with Claude where the AI builds, tests
and screenshots everything in its own Linux sandbox, and you define, scope
and QA. Nothing runs on your machine until you apply it.

## What's here

- **[SANDBOX_PRIMER.md](SANDBOX_PRIMER.md)**: paste this at the start of a
  session. The standing rules: build and run it, prove each test can fail,
  look at the screenshots, deliver whole files or verified patches, and say
  what wasn't verified.
- **[templates/PROJECT_SANDBOX_RECIPE.md](templates/PROJECT_SANDBOX_RECIPE.md)**:
  copy to `docs/README_Claude_sandbox.md` in your project and have the AI fill
  it in during the first session. Every session starts from an empty sandbox;
  this file holds the toolchain, workarounds and traps so the next session
  doesn't rediscover them.

## Starting a session

1. Give the AI the repo (a public URL to clone, or a zip).
2. Paste the primer.
3. Point at `docs/README_Claude_sandbox.md`, or ask for it as the first task.
4. Start on features.

## Why

The reasoning and the lessons behind each rule: [docs/METHOD.md](docs/METHOD.md).

Built this way: [Easel](https://github.com/scottpeterman/easel),
[Bounty Hunter](https://github.com/scottpeterman/bountyhunter).