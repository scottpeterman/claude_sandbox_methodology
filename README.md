# Sandbox-First Development with an AI Partner

A working method for building real applications with an AI model: desktop apps, games and network tools. The AI does the implementation **inside its own isolated Linux sandbox**. The human provides definition, scope and QA, and only receives changes that have already been built, tested and looked at.

This is not "agent on your laptop" development. Nothing runs on your machine until you choose to apply it, and everything that reaches you has already been through a build, a test suite, and usually a screenshot.

The examples come from real projects built this way:

- [Easel](https://github.com/scottpeterman/easel): a layered image editor, C++20/Qt 6, GPU rendering, CI for Windows, Linux and macOS.
- [Bounty Hunter](https://github.com/scottpeterman/bountyhunter): a single-file HTML raycaster game.
- Go/Fyne desktop tools and Python network automation.

This document has two jobs. Read it to understand the method. Then hand [`SANDBOX_PRIMER.md`](SANDBOX_PRIMER.md) to a new session to put an AI to work this way from the first message.

---

## The division of labour

| The human | The AI in the sandbox |
| --- | --- |
| Says what to build and why | Picks the approach and writes the code |
| Sets scope: what's in, what's next, what's out | Builds it with the real toolchain |
| Tests on real hardware, real data, real workflows | Writes and runs automated tests |
| Judges the result: "that's the inverse", "there's a halo" | Takes screenshots and looks at them before reporting |
| Applies and pushes the changes | Delivers patches that are verified to apply |

The human writes little or no code. The human's job is closer to **product owner plus QA lead**, and that job is not optional. The problems that matter most are found by using the tool on real work, not by the test suite.

## Why the sandbox rather than an agent on your machine

- **Nothing executes on your machine until you apply it.** The AI has no credentials, no access to your repo, no ability to push. That is a safety property, not a limitation to work around.
- **It runs inside an ordinary chat session.** No agent loop on your hardware, no local tool-permission prompts, no second bill.
- **Verification comes with the change.** Because the sandbox can build and run the product, "done" means built, tested and looked at, not "here is some code that should work".

The cost is that the sandbox is Linux-only, has no GPU and no access to your devices. Sections 10 and "Limits" below cover what that leaves for you and CI.

---

## 1. Choose stacks the sandbox can fully build and see

The single most important decision. If the sandbox can compile, run and **render** the application, the AI can verify its own work. If it can't, you become the only test harness.

Stacks that work end to end in a headless Linux container:

| Stack | How it runs headless | How the AI sees it |
| --- | --- | --- |
| **Qt 6 (C++ or Python)** | `QT_QPA_PLATFORM=offscreen`, or `xvfb-run` | `QWidget::grab()` to PNG. GPU paths (QRhi) run on Mesa's software OpenGL. |
| **HTML/JS** | Headless Chromium through Playwright | Page screenshots; `page.evaluate()` to read state |
| **Go + Fyne** | `xvfb-run` with Mesa GL | Window captures |
| **Node / Python CLIs and services** | Directly | stdout, files, HTTP responses |

Prefer **open source toolkits and libraries**. The AI can read their source to settle how something behaves rather than guessing, and can build them from source when a version has to match yours.

Keep the build scripted (`scripts/build.sh` or similar) so that the sandbox, your machine and CI all run the same commands.

---

## 2. Get the exact toolchain into the sandbox

The sandbox usually reaches package registries and GitHub, not the whole internet. Within that:

- **Distro packages first** (`apt`) for compilers, Mesa, xvfb, image libraries.
- **Language registries**: `pip`, `npm`, Go modules.
- **Build from source when the version matters.** Easel targets Qt 6.10.x, the version on the developer's machines, so Qt 6.10.2 was cloned from GitHub and built in the sandbox (including `qtshadertools`), rather than accepting an older distro Qt with different behaviour.
- **Clone dependencies directly** from GitHub when a package isn't available or a specific commit is needed.

Match what you run. Building against a different major or minor version than your machine produces bugs that only show up on your side.

**Work around what the sandbox can't reach without changing what ships.** When the sandbox cannot see the Go module proxy, fetch `golang.org/x/*` from GitHub mirrors through a *copy* of `go.mod` in `/tmp`, and point every Go command at the copy. The `go.mod` and `go.sum` in the tree stay the shipping ones, so there is nothing to revert before delivering. Any workaround that edits a file you ship is a workaround that will eventually ship.

---

## 3. Write the sandbox recipe into the repo

This is the highest-leverage artifact in the whole method, and the one most people miss.

The sandbox is temporary. Every session starts empty and has to rediscover the toolchain, the workarounds and the traps. Put that knowledge in the repo as `docs/README_Claude_sandbox.md`, and a new session becomes productive in one tool call instead of twenty.

One project's recipe has ten sections:

0. **What the sandbox is** — the distro, the CPU count, the network allowlist, the shape of a tool call.
1. **Toolchain** — the exact `apt` and `pip` lines, and how to get a Go newer than the distro's.
2. **The workaround** — the modfile copy, and the three things learned the hard way about it.
3. **Build and test** — the exact commands, including which test runs split out and why.
4. **The C surface and the Qt app** — configure, build, `ctest`, and the stale-binary trap.
5. **The parsing engine and its Python original** — how to regenerate the golden files.
6. **End-to-end against fake devices** — one command block, because the fake device dies with the tool call that started it.
7. **Scrub before anything leaves.**
8. **The tool shell** — dash, not bash; the time limit; background processes.
9. **Delivering.**
10. **What the sandbox cannot tell you** — the explicit list of what is still unproved.

Section 10 matters as much as the rest. It is the standing honest answer to "is this ready?": no macOS, no Windows, no arm64, no display, no OS keyring, no real hardware. Everything on that list is a QA task for a human, and writing it down stops it being quietly assumed away.

There is a starting template in [`templates/PROJECT_SANDBOX_RECIPE.md`](templates/PROJECT_SANDBOX_RECIPE.md). Have the AI fill it in during the first session, and update it whenever a session loses time to something that could have been written down.

---

## 4. Verify visually, not just by "it compiled"

After every visible change, the AI takes a screenshot and **looks at it** before telling you it's done.

- **Screenshot harnesses** live outside the product: small programs that open the real main window, drive it through a scenario and save a PNG. For Easel, `shot_select.cpp` draws a 14×9 sprite pixel by pixel with the real brush tool, selects it with the ellipse tool, pastes and drags it, then grabs the window with the marching ants showing.
- **Pixel probes**: when a screenshot looks slightly wrong, read the actual RGB values (Python/PIL) rather than guessing from a scaled-down image.
- **Crops**: a 1400×860 grab scaled to fit is too small to judge a label. Crop the region and look at that.
- **Headless browser screenshots** do the same job for HTML: load the page, set the state, capture.
- **Contact sheets** for anything with variants. The sprite "turn" function was checked by rendering four of the game's real sprites at seven angles into one PNG, straight from the game file's own packed data.

The point is not that the screenshot exists. It is that **every check passed while the picture showed something wrong**. From one project's grabs: a results table with the DETAIL column pushed off-screen, a decisions list scrolling sideways, a search highlight unreadable after a theme change, dialogs with a light grey body on the dark theme (widgets that autofill use the palette, not the stylesheet), a device name squeezed to an ellipsis, a clipped checkbox label, an empty form row, and a panel still titled for the previous file. Each became an assertion after it was seen.

Screenshots also catch design mistakes, not only bugs. A Color to Alpha workflow looked right in theory. The first screenshot showed it turning a dark sprite see-through, and the method was changed before it ever reached the user.

**The loop runs both ways.** You paste screenshots from real hardware; the AI reads them and finds things you weren't asking about. A capture screenshot from a real lab produced three findings in one pass: a hard-coded plural ("1 devices"), a folder checkbox that ticked devices hidden by the filter, and a device tree too short for a real inventory.

---

## 5. Test through the real toolkit

Unit tests on the core logic, plus tests that drive the actual UI:

- **Qt**: QtTest, or a purpose-built probe, with synthesized mouse, keyboard and tablet events against the real `MainWindow`, run offscreen. GPU rendering tests read back the framebuffer.
- **Browser**: Playwright clicking and typing into the real page.
- **Keep an end-to-end test per user workflow**, such as select → copy → paste → drag → commit → undo. These catch integration bugs unit tests can't.

A pattern worth copying: **one probe binary that drives the real window through the whole product and exits with the failure count.** One network tool's `app_probe` opens the application offscreen, plays a scripted run, creates a vault, edits a credential through the real dialog, captures from fake devices, browses the store, diffs two versions, searches, imports a topology map, edits the inventory, and grabs every view in every theme. It is one `ctest` entry. When it passes, the product works end to end; when it fails, it names the check.

Lessons from doing this at volume:

- **Drive the widgets, not the model.** A test that builds the request JSON itself proves nothing about the form that normally builds it. The first version of one probe called `startCapture()` with hand-written JSON, so the entire form was untested — and the form had a bug.
- **Shortcuts need an active window.** Synthesized key events don't trigger window shortcuts until the test calls `activateWindow()` and waits for the window to be active.
- **Test the workflow the user actually runs.** One bug (Move couldn't pick up pixels while a drag was in progress) passed every unit test and was caught only by a test that dragged with the real mouse path.
- **Write down wrong test expectations as they happen.** When a test fails, check the expectation before the code. About a third of failures are the test being wrong.

---

## 6. Prove the test can fail

A test that passes the first time proves the code works **or** that the test is blind. Those look identical in a green run.

So: break the code on purpose, confirm the test fails, restore, confirm it passes. Every new assertion, while it is still cheap.

Real examples, each of which found a blind test or confirmed a real one:

- Reverting a one-line fix so the window stopped following the store path: one assertion failed, and the other two that "covered" it passed — they were checking a device list that was already correct for the wrong reason.
- Sending ticked devices as host patterns instead of exact keys: three checks failed, which is what the port-forwarded-device case is there to catch.
- Making a diff engine emit delete-everything-then-insert-everything: the property test comparing against a brute-force LCS failed, which is what proves the edit script is minimal rather than merely correct.
- Skipping the paging-disable command in a fingerprint path: the test hung at `--More--` and timed out, which is exactly the production failure it exists to prevent.

**And the honest one.** A blindness check "passed" when it should have failed. The deliberately broken build had not compiled, the error was missed in a grep, and the probe ran the previous binary. Nothing about the output said so.

That produced two standing rules:

- **Read the build output, not just the exit code**, and make the broken version a version that still compiles.
- **Check the timestamps.** If a change seems to do nothing, confirm the binary was actually relinked before believing any result.

The stale-binary trap is the same failure in normal work: when an early build step fails, the old binaries are still there and still run.

---

## 7. Measure at realistic scale

Performance claims come from running, not reasoning. Create synthetic data at the size you'll actually use and time the operations:

- An 8000×8000 image, to find that opening a file froze the window (the fix: load on a worker thread; the longest UI stall is now 20 ms).
- A 2172×724 sprite sheet, to time the magic wand (~100 ms), Color to Alpha (1.1 s, then ~95 ms after replacing `pow()` with lookup tables) and trim (~10 ms).
- A 2000-line config, to confirm a line diff of a one-line change stays instant — and to set the cap past which a diff is refused rather than computed.

Report the numbers with each change. "Fast enough" becomes a figure you can check.

---

## 8. Simulate the things the sandbox can't reach

The sandbox has no routers, no tablets and no production network. So it fakes them:

- **Network devices**: stand up listening servers that behave like the device. An SSH server that returns canned `show` output for Junos, EOS or IOS lets collectors, parsers and terminal tools run end to end. SNMP responders and plain TCP listeners do the same for polling and discovery code. A larger version of this is a full emulated network (many devices, consistent topology) for demos and scale tests.
- **Make the fake reproduce the failure, not just the happy path.** A fake device that answers everything cannot prove a hang. Adding a pager — long output stops at `--More--` and never returns a prompt — turned "the paging command matters" from an argument into a test.
- **Fakes earn their keep.** Two fake devices behind port-forwards on one address found two real identity bugs: devices filed under their dial address instead of their prompt name, and two devices merging into one.
- **Real output as fixtures**: paste real CLI output from your lab devices into the chat once, and it becomes permanent test data. Vendor output formats are where guessing is most dangerous.
- **Input devices**: synthesize pen events with pressure to test pressure curves. Then verify on a real tablet, because that isn't optional.

---

## 9. Deliver changes you can apply in one step

The sandbox can't push to your repo, and it shouldn't. Two delivery methods, depending on the project:

**Open source projects: patches from GitHub.**

1. The AI clones or fetches the public repo read-only and syncs to the latest `origin/main` before starting.
2. It commits in the sandbox, then runs `git format-patch origin/main`.
3. It clones the repo fresh and runs `git am` on the patch **to prove it applies**.
4. You run:

```
cd ~/github/easel
git am ~/Downloads/0001-Magic-wand-Color-to-Alpha-Trim-and-selection-masks.patch
git push
```

The commit keeps its message, authorship and co-author line, and CI runs on the push.

**Private or very large projects: complete files in the project tree.** The AI returns whole files (never fragments or "change line 40 to…"), laid out in the project's directory structure, so you copy them over the tree in one step. A zip with the right paths works well.

Either way, no snippets to paste by hand. A change you have to reassemble is a change you'll get wrong.

---

## 10. Let CI cover the platforms the sandbox can't

The sandbox is Linux. GitHub Actions builds and tests Windows (MSVC) and macOS on every push, and publishes the installers (AppImage, dmg, Windows zip). The AI writes and fixes the workflow from the CI logs you paste back. The first Windows failure ("qt.conf path is not an absolute path") was fixed that way.

Some cross-platform work the sandbox *can* do, if the stack allows it. Pure Go command-line tools cross-compile from Linux to Windows and macOS with no C toolchain, and the sandbox can verify the result is genuinely a Windows binary (`file` reporting `PE32+ executable (console) x86-64`) before you ever open a Windows machine. Anything that links Qt or cgo has to be built on the target.

Things neither the sandbox nor CI can verify go on an explicit list: real pen pressure, Metal on a real Mac, a real display's colour, an OS keyring, a real router. Those are your QA tasks. Platform packaging in particular resists sandbox verification, and the failures are specific: an ad-hoc-signed macOS bundle signed with the hardened runtime passes `codesign --verify` and then dies at launch on library validation; a Qt WebEngine helper needs its own path back to the app's frameworks or the renderer never starts. Expect to find those on the real machine, and expect the fix to be one line.

---

## 11. Keep experiments out of the product

Scratch work (screenshot harnesses, benchmarks, spikes) goes in a separate folder that pulls the product in with CMake `add_subdirectory` or an import. It links against the real code without being committed to the repo. The product tree stays clean, and the harnesses stay available for the next change.

---

## 12. Work in small, shippable increments

- A **spec document** with milestones (for Easel: M0 canvas → M7 import/export polish) keeps a long project coherent across many sessions.
- Each request becomes one patch the user can run the same day. On one day of Easel: selections, cut, copy and paste → sprite grid and whole-pixel zoom → magic wand, Color to Alpha and trim. Three patches, each used on real art before the next was started.
- **Real use redirects the plan.** Sprite work on a real sheet pulled layers-style features forward and pushed others back. That's the point of shipping small.

---

## 13. Keep sensitive information out of the session

Never upload sensitive information into a public Claude account. That means no configs, captures, screenshots, hostnames or addresses from a network you don't own. Use lab devices and synthetic data instead, which is what the fakes in section 8 are for.

---

## What the human must still do

- **Decide what matters.** The AI can build almost anything. Choosing what is worth building is the job.
- **Test with real work.** Real images, real devices, real configs. The most important problems in the examples above ("I got the inverse", a faint halo only visible on the real sprite, a DMG whose app could not find its helper) were found this way, with every automated test passing.
- **Say when the answer is wrong, not just when it fails.** "That's the inverse" is worth more than a stack trace.
- **Own anything that touches production.** Tools that only read state are low risk. Anything that writes config goes through review and a change process, with commit-confirmed and rollback, however it was built.

---

## Limits to plan around

- **No real GPU.** Rendering is tested on software OpenGL (Mesa). Driver-specific issues show up on real hardware.
- **Limited network.** Registries and GitHub usually work; arbitrary websites don't. Bring external files in by attaching them.
- **The sandbox is temporary.** Anything worth keeping goes into git (or into the files delivered to you) before the session ends.
- **Long sessions get summarised.** Keep the plan in a spec document and the state in the repo, not only in the conversation.
- **No credentials.** The AI can read public repos but not push, which is also a safety property.
- **Tool calls have a shape, and it constrains test design.** Each call is a fresh shell, so environment variables don't carry over. Background processes die with the call that started them, which is why a fake server and the test that uses it go in one call. There is a time limit, and a call that exceeds it returns *no* output at all — so long work writes to a file and prints the file, and a test suite that approaches the limit gets split before it silently starts telling you nothing.

---

## Starting a new session

1. Give the AI the repo (a public URL it can clone, or a zip).
2. Paste [`SANDBOX_PRIMER.md`](SANDBOX_PRIMER.md).
3. Point at `docs/README_Claude_sandbox.md` in your repo. If there isn't one yet, ask for it as the first task — have the AI establish the toolchain, get the project building and testing, and write down what it learned.
4. Then start on features.

The primer is the standing instructions. The recipe is what makes session five as fast as session four.

---

## Minimal recipe

1. Pick a stack from section 1.
2. Put the project on GitHub, public if you can.
3. Ask for a build script and CI for every platform you target, and get the empty app building and running before any features.
4. Ask for `docs/README_Claude_sandbox.md` and keep it current.
5. For each feature: describe it, receive a tested patch plus a screenshot, apply it, use it on real work, report what's wrong.
6. Keep a spec document with milestones; revise it as real use teaches you what matters.