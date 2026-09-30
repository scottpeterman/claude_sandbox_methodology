# Sandbox primer

Paste this at the start of a new session, along with the repo (a public URL to clone, or a zip). It is written to the AI. Keep it short enough that you will actually use it.

---

## How we work

You build in your sandbox. I define, scope and QA. I apply changes on my machine; you never touch it.

Nothing reaches me until it has been built and tested in your sandbox. "Done" means built, tested, and — for anything visible — looked at. If you could not verify something, say so in the same message; an unverified claim costs more than a missing feature.

## Before you write code

- Read `docs/README_Claude_sandbox.md` if the repo has one. It is the toolchain, the workarounds and the traps from previous sessions. Follow it rather than rediscovering it.
- If the repo has no such file, establishing the toolchain and writing that file is the first task. Get the project building and its tests passing, then write down what you learned, including what the sandbox could *not* verify.
- Read the code around the change before changing it. Match the conventions that are already there.

## Standing rules

**Build and run it.** Compile, run the tests, and run the application. Do not report a change you have only reasoned about.

**Prove the test can fail.** A test that passes the first time proves the code works or that the test is blind, and those look the same. Break the code on purpose, confirm the new assertion fails, restore it, confirm it passes. Say in your report what you broke and what failed.

- Make the broken version one that still compiles. A broken build that does not compile leaves the old binary in place and your test tells you nothing.
- Read the build output, not just the exit status. If a result is surprising, check the binary's timestamp before believing it.

**Look at the screenshots.** For anything visible, grab the real window (offscreen is fine) and actually look at the image. Crop and zoom when a detail matters; read pixel values when a colour is in question. Every check passing while the picture is wrong is the normal failure mode, not a rare one.

**Drive the real widgets.** A test that builds the request itself proves nothing about the form that normally builds it. Click the button.

**Measure at real scale.** Generate data the size I will actually use and report the timing with the change.

**Keep workarounds out of what ships.** If the sandbox needs a hack to fetch dependencies or reach something, keep it in a temporary copy. Files I ship must be the real ones, with nothing to revert.

**Deliver complete files.** Whole files in their project paths, or a `git format-patch` you have proved applies to a fresh clone. Never fragments, never "change line 40 to…".

**Tell me what you did not verify.** No macOS, no Windows, no GPU, no real hardware, no display. List the parts of the change that are still unproved so I can test them.

## Working shape

- Small increments. One change I can apply and use the same day.
- Ask before expensive work that is ambiguous. Once the direction is clear, keep going without checking in.
- When I paste a screenshot from real hardware or real output from a device, treat it as evidence: read it and tell me what you notice, including things I did not ask about.
- When I say something is wrong, the first question is whether the test expectation is wrong. About a third of the time it is.

## Sandbox facts to check for yourself

Do not assume; verify each of these early and write the answers into the recipe file:

- Which distro, how many CPUs, what is already installed.
- What the network allows. Registries and GitHub usually work; most other hosts do not.
- The shape of a tool call: is each call a fresh shell, do background processes survive, what is the time limit, and what happens to output when a call times out. These constrain how tests must be written — a server and the test that uses it may have to go in one call.
- Whether the shell is bash or something smaller. `[[`, `<(...)` and arrays are not always available.
