# README_Claude_sandbox.md — template

Copy this to `docs/README_Claude_sandbox.md` in your project and have the AI fill it in during the first session. Delete the guidance in *italics* as you go.

The goal is that a new session reads this one file and is productive immediately, instead of spending twenty tool calls rediscovering the toolchain. Update it whenever a session loses time to something that could have been written down.

---

# README_Claude_sandbox.md

Recipe for building and testing *\<project\>* in a Claude sandbox, so changes get
**built, run and tested** there rather than reasoned about from the source.

Verified end to end on *\<distro and version\>* with *\<the versions of every tool that matters\>*.

---

## 0. What the sandbox is

*State the properties that decide what can be proved here. Verify each one; do not assume.*

- **Distro, CPU count, what is preinstalled.**
- **Network egress.** *Which hosts are reachable. Name the ones that are not and that you needed — that is the line a future session will otherwise waste time on.*
- **The shape of a tool call.** *Fresh shell each time? Do environment variables carry over? Do background processes survive the call that started them? What is the time limit, and what happens to output when a call exceeds it? These decide how tests have to be written.*
- **How files arrive and leave.**
- **What is absent.** *No display, no GPU, no keyring, no real hardware, one architecture. Section 10 is what that leaves unproved.*

## 1. Toolchain

*The exact commands, copy-pasteable, one per line.*

```
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y <packages>
pip install --break-system-packages <packages>
```

*For each package, one line on why it is needed — a future session will otherwise drop it as unused.*

*If a distro package is too old, record how to get the right version, and why the obvious route does not work.*

## 2. Workarounds

*Anything the sandbox cannot reach, and how it is worked around **without changing files that ship**.*

*Record the things learned the hard way here. They are the highest-value lines in the file.*

## 3. Build and tests

```
<build command>
<test command>
```

*Note which test runs have to be split out and why — a suite that exceeds the tool-call limit returns no output at all, which reads as a hang rather than a timeout.*

*State the rule: prove a test catches its bug. Break the code on purpose and confirm the test fails. List the tests that were blind until this was done — that list is what makes the rule stick.*

## 4. The application

```
<configure>
<build>
<run the tests>
```

*The traps:*

- *Which environment must be in the same call as which command.*
- *The stale-binary trap: if an early step fails, old binaries are still there and still run. How to tell.*
- *What the probe covers, in one paragraph.*

**Look at the grabs, not just the exit status.**

```
QT_QPA_PLATFORM=offscreen <probe> <output prefix> <args>
```

*Then view the PNGs. List what the grabs have caught that every assertion missed — that list is the argument for doing it.*

*Note how the headless renderer differs from a real desktop, so a grab is not mistaken for proof of the desktop's own style.*

## 5. *\<Anything held to an external reference\>*

*A port held to an original implementation, golden files, a corpus. How to regenerate them, and what is not in this repository.*

## 6. End to end

*The full path with fakes standing in for what the sandbox cannot reach. One command block if background processes die with the call.*

*Use a throwaway `HOME` so default paths get exercised without touching anything real. The first run on a real machine failing on a directory nobody had created is a bug this catches.*

## 7. Scrub before anything leaves

```
<DENYLIST_VAR>=/path/to/denylist scripts/scrub-check.sh
```

*Where the denylist lives (outside the repo) and why. What the check does that a plain grep does not.*

## 8. The tool shell

*The specifics: which shell, what it lacks, the time limit, background processes, and any file-handling traps (line endings, encodings) that have cost a session before.*

## 9. Delivering

```
<the commands that produce the deliverable>
```

*What must be true before anything leaves: formatter clean, scrub clean, tests green. What to say in the report when a dependency changed.*

## 10. What the sandbox cannot tell you

*The explicit list of what is still unproved. This is the standing honest answer to "is it ready?", and every line on it is a QA task for a human.*

- *No \<platform\>, no \<architecture\>.*
- *No display: what the offscreen run does and does not prove.*
- *No \<hardware\>: what the fakes prove, and what they cannot.*
- *Anything that has only ever been verified on a real machine, and what was found there.*
