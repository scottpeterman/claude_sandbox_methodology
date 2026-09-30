# Method: the technical side

How the sandbox is actually used: the commands, the harnesses and the traps. The rules they serve are in [`SANDBOX_PRIMER.md`](../SANDBOX_PRIMER.md). Where a detail depends on the environment (tool-call limits, network access), it's described as it was in the sessions this was written from, as of September 2026. Verify it in yours and record the answer in your project's `docs/README_Claude_sandbox.md`.

The fake device, the Playwright script, the pixel probe, the Go mirror workaround and the cross-compile check were all run in a sandbox while this was written. The Qt examples come from Easel's own harnesses.

---

## 1. Pick a stack the sandbox can build, run and see

| Stack | Run headless | See the result |
| --- | --- | --- |
| Qt 6 widgets (C++ or Python) | `QT_QPA_PLATFORM=offscreen` | `QWidget::grab()` to PNG |
| Qt 6 with GPU rendering (QRhi, OpenGL, `QRhiWidget`) | `xvfb-run` with Mesa (software OpenGL) | `grab()`, or read back the framebuffer |
| HTML / JS | Headless Chromium through Playwright | Page screenshots, `page.evaluate()` |
| Go + Fyne | `xvfb-run` with Mesa | Window captures |
| Go command-line tools | Directly; cross-compile for other platforms (section 10) | stdout, files |
| Python services and CLIs | Directly | stdout, files, HTTP responses |

The `offscreen` platform has no OpenGL context. A GPU-rendered Qt app needs a real X server, even a virtual one:

```
apt-get install -y xvfb libgl1-mesa-dri
xvfb-run -a -s "-screen 0 1920x1080x24" ctest --test-dir build --output-on-failure
```

Keep the build in a script (`scripts/build.sh`) so the sandbox and your machines run the same commands.

---

## 2. Get the exact toolchain

Package registries and GitHub are usually reachable; most other hosts aren't. Check early:

```
curl -s -o /dev/null -w "%{http_code}\n" --max-time 8 https://pypi.org/simple/
curl -s -o /dev/null -w "%{http_code}\n" --max-time 8 https://proxy.golang.org/
```

**Distro packages, then registries.** Record every package in the recipe with one line on why it's needed, or a later session will drop it as unused.

```
apt-get update
DEBIAN_FRONTEND=noninteractive apt-get install -y build-essential cmake ninja-build xvfb libgl1-mesa-dri
pip install --break-system-packages playwright asyncssh pillow
```

**Build from source when the version matters.** Match the minor version you run. Easel targets Qt 6.10, which the distro didn't have, so Qt was built from GitHub source with only the modules needed. Roughly:

```
git clone --branch v6.10.2 https://github.com/qt/qt5.git qt-src
cd qt-src
./init-repository --module-subset=qtbase,qtshadertools
mkdir build
cd build
../configure -prefix /opt/qt/6.10.2 -release -nomake examples -nomake tests
cmake --build . --parallel
cmake --install .
```

Put the exact commands that worked into the recipe; it takes long enough that no session should have to rediscover them.

A trap from the same project: since Qt 6.9, private modules have to be named explicitly (`find_package(Qt6 REQUIRED COMPONENTS CorePrivate GuiPrivate)`). It fails on your machine and in CI identically, so it's worth writing down once.

**Work around unreachable hosts without touching files that ship.** Go example: when `proxy.golang.org` is blocked but GitHub isn't, clone the dependency from its GitHub mirror and point a *copy* of `go.mod` at it. The `go.mod` and `go.sum` in the tree never change:

```
git clone --depth 1 --branch v0.21.0 https://github.com/golang/text /tmp/gomirror/text
mkdir -p /tmp/modwork
cp go.mod /tmp/modwork/go.mod
touch /tmp/modwork/go.sum
GOPROXY=off GOFLAGS=-mod=mod go mod edit -modfile=/tmp/modwork/go.mod -replace golang.org/x/text=/tmp/gomirror/text
GOPROXY=off GOFLAGS=-mod=mod go build -modfile=/tmp/modwork/go.mod ./...
```

Use a directory replacement, not a version on the mirror's module path. A `github.com/golang/text@v…` replacement fails because the mirror's `go.mod` declares `golang.org/x/text`.

---

## 3. Know the shape of a tool call

These decide how tests have to be written. In the sessions this was written from:

- **Each call starts a fresh shell.** Environment variables and `cd` don't carry over. Put `QT_QPA_PLATFORM=offscreen` or `xvfb-run` on the same line as the command.
- **Background processes may not survive the call that started them.** Start a fake server and run the test that uses it in the same call, and kill the server at the end.
- **There's a time limit, and a call that exceeds it may return no output at all**, which looks like a hang. Write long output to a file and print the tail. Split a test suite before it gets close to the limit.
- **The shell may be `dash`, not `bash`.** No `[[`, no `<(...)`, no arrays. Write POSIX or call `bash -c`.

---

## 4. Look at the result

### Qt: a screenshot harness that drives the real window

Keep harnesses outside the product (section 11). This one opens the real main window, sets up a scenario through the same calls the UI makes, and saves the window:

```cpp
#include "mainwindow.h"
#include "canvasview.h"
#include <QApplication>
#include <QEventLoop>
#include <QTest>
#include <QTimer>

static void pump(int ms)
{
    QEventLoop loop;
    QTimer::singleShot(ms, &loop, &QEventLoop::quit);
    loop.exec();
}

int main(int argc, char **argv)
{
    QApplication app(argc, argv);
    MainWindow w;
    w.resize(1400, 900);
    w.show();
    w.activateWindow();
    QTest::qWaitForWindowActive(&w);       // shortcuts don't fire until this is true
    w.newDocument(QSize(200, 90), Qt::black);
    CanvasView *v = w.canvasView();
    v->setZoomCentered(8.0);
    const auto at = [&](QPointF c) { return v->canvasToView().map(c).toPoint(); };
    QTest::keyClick(v, Qt::Key_M);          // the real shortcut, not a method call
    QTest::mouseClick(v, Qt::LeftButton, Qt::NoModifier, at({80, 20}));
    pump(1000);                             // let rendering and timers settle
    w.grab().save(argv[1]);
    return 0;
}
```

```
xvfb-run -a -s "-screen 0 1920x1080x24" ./build/shot_grid /tmp/grid.png
```

Then open the PNG and look at it. That's the step that matters.

### HTML: Playwright

```python
"""Load a local HTML page headless, let it run, press a key, save a screenshot."""
import sys
from pathlib import Path

from playwright.sync_api import sync_playwright

page_path, out_png = Path(sys.argv[1]).resolve(), sys.argv[2]
with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page(viewport={"width": 1280, "height": 800})
    errors = []
    page.on("pageerror", lambda e: errors.append(str(e)))
    page.goto(page_path.as_uri())
    page.wait_for_timeout(1000)          # let requestAnimationFrame run
    page.keyboard.press("ArrowUp")        # drive it like a player would
    page.wait_for_timeout(500)
    page.screenshot(path=out_png)
    browser.close()
print("page errors:", errors or "none")
```

Collecting `pageerror` matters: a canvas game can throw every frame and still draw something plausible.

### Pixel probes and crops

A 1400×900 grab scaled to fit a viewer is too small to judge a one-pixel line or a label. Crop the region, and read values when a colour is in question:

```python
from PIL import Image

im = Image.open("/tmp/grid.png").convert("RGB")
im.crop((330, 120, 620, 470)).resize((580, 700), Image.NEAREST).save("/tmp/grid_crop.png")
for y in range(132, 142):
    print(y, im.getpixel((200, y)))
```

### Contact sheets

For anything with variants (angles, themes, zoom levels), render them all into one image and look once. The sprite turn function was checked by rendering four real sprites from the game's packed data at seven angles on one sheet.

---

## 5. Test through the real toolkit

- **Qt**: QtTest, registered with `ctest`, run under `xvfb-run` for anything that renders. Send synthesized mouse, keyboard and tablet events to the real widgets. `QRhiWidget::grabFramebuffer()` reads back what the GPU path drew.
- **Browser**: Playwright clicking and typing into the real page.
- **One probe binary per product** is worth having: it drives the real window through every workflow, grabs every view, and exits with the number of failed checks. It's one `ctest` entry, and when it fails it names the check.

Traps:

- **Shortcuts need an active window.** Synthesized key events on a widget won't trigger window shortcuts until the test calls `activateWindow()` and `QTest::qWaitForWindowActive()`.
- **Drive the widget, not the method behind it.** Calling the save function with hand-built arguments leaves the dialog that normally builds them untested.
- **Check the expectation first when a test fails.** It is often the test that's wrong.

---

## 6. Prove the test can fail

After writing an assertion, break the code it protects, confirm the test fails, then restore it. Two mechanical checks make this reliable:

```
cmake --build build 2>&1 | tail -30
stat -c '%y %n' build/tests/tst_editing
```

- **Read the build output, not the exit status of a pipeline.** A broken version that doesn't compile leaves the previous binary in place, and the test runs the old code and passes.
- **Check the binary's timestamp** whenever a change seems to have no effect.

---

## 7. Measure at the size you'll use

Generate synthetic data at real scale and time the operation in the product's own code:

```cpp
QElapsedTimer t;
t.start();
const easel::Selection w = easel::magicWand(store, canvas, QPoint(0, 0), 0.06, true);
printf("wand %lld ms\n", t.elapsed());
```

Report the number with the change. On a 2172×724 sprite sheet this showed Color to Alpha at 1.1 s; replacing `pow()` with lookup tables brought it to 95 ms.

---

## 8. Fake what the sandbox can't reach

### A fake network device

SSH in, get a prompt, get canned output per command. Any login is accepted. Long output stops at `--More--` and never returns a prompt until `terminal length 0` is sent, which is the hang a real collector has to avoid:

```python
"""A fake network device: python3 fake_device.py PORT HOSTNAME"""
import asyncio
import sys

import asyncssh

PAGE = 24
OUTPUT = {
    "show version": "Arista DCS-7050SX3-48YC8\nSoftware image version: 4.30.1F\n",
    "show interfaces status": "".join(f"Et{i}    connected    1    full   10G\n" for i in range(1, 49)),
}


class AnyLogin(asyncssh.SSHServer):
    def begin_auth(self, username):
        return True

    def password_auth_supported(self):
        return True

    def validate_password(self, username, password):
        return True


def make_session(hostname):
    async def session(process):
        paging = True
        prompt = f"{hostname}#"
        process.stdout.write(prompt)
        while True:
            line = await process.stdin.readline()
            if not line:
                break
            cmd = line.strip()
            if cmd in ("exit", "quit"):
                break
            if cmd == "terminal length 0":
                paging = False
            else:
                lines = OUTPUT.get(cmd, f"% Invalid input: {cmd}\n").splitlines(keepends=True)
                if paging and len(lines) > PAGE:
                    process.stdout.write("".join(lines[:PAGE]) + "--More--")
                    await asyncio.Event().wait()  # stuck at the pager, like the real thing
                process.stdout.write("".join(lines))
            process.stdout.write(prompt)
        process.exit(0)
    return session


async def main(port, hostname):
    key = asyncssh.generate_private_key("ssh-ed25519")
    await asyncssh.create_server(AnyLogin, "127.0.0.1", port, server_host_keys=[key],
                                 process_factory=make_session(hostname))
    await asyncio.Event().wait()


if __name__ == "__main__":
    asyncio.run(main(int(sys.argv[1]), sys.argv[2]))
```

Start it and run the test in one call, then clean up:

```
python3 fake_device.py 2222 leaf1 & echo $! > /tmp/fake.pid
sleep 1
python3 -m pytest tests/test_collector.py
kill $(cat /tmp/fake.pid)
```

What makes fakes earn their keep:

- **Reproduce the failure, not only the happy path.** Without the pager, nothing can prove that the collector disables paging.
- **Run several devices on one address, on different ports.** That's how port-forwarded lab devices look, and it's where identity bugs hide: devices filed under the dial address instead of their prompt name, or two devices merged into one.
- **Use real output from lab devices** as the canned responses. Vendor formats are where guessing costs the most.
- **Use a throwaway `HOME`** for the end-to-end run, so default paths are created from nothing, as on a new machine:

```
HOME=$(mktemp -d) python3 -m yourtool capture --inventory tests/fixtures/inventory.yaml
```

For SNMP, a responder that replays recorded walks (such as `snmpsim`) plays the same role.

---

## 9. Deliver changes that apply in one step

### Public repo: a verified patch

```
git fetch origin
git rebase origin/main
git format-patch origin/main -o /tmp/out
git clone https://github.com/you/project /tmp/verify
cd /tmp/verify
git am /tmp/out/0001-*.patch
```

Only a patch that applies to a fresh clone gets handed over. Applying it on your machine is then:

```
cd ~/github/project
git am ~/Downloads/0001-the-change.patch
git push
```

If a patch was applied before and failed partway, `git am` refuses with "previous rebase directory .git/rebase-apply still exists". `git am --abort` puts things back; `git am --quit` keeps the working tree as it is.

### Private or large repo: complete files in the tree layout

```
cd project
zip -r /tmp/out/changes.zip src/ui/mainwindow.cpp src/ui/mainwindow.h tests/tst_editing.cpp
```

Unzip over the tree. Whole files only, never fragments.

---

## 10. Cross-platform: what the sandbox can and can't build

Pure Go tools cross-compile with no C toolchain, and `file` confirms what came out:

```
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -o dist/tool.exe .
CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 go build -o dist/tool-mac .
file dist/tool.exe dist/tool-mac
```

```
dist/tool.exe: PE32+ executable (console) x86-64, for MS Windows
dist/tool-mac: Mach-O 64-bit arm64 executable
```

Anything that links Qt or uses cgo has to be built on the target, by you or by CI. Packaging fails in platform-specific ways the sandbox can't see, for example:

- An ad-hoc-signed macOS bundle with the hardened runtime passes `codesign --verify`, then dies at launch on library validation.
- A Qt WebEngine helper process needs its own path back to the app's frameworks, or the renderer never starts.

Expect to find those on the real machine, and expect the fix to be small.

---

## 11. Keep harnesses out of the product

Harnesses, benchmarks and spikes go in a scratch folder that builds the product as a subproject, so they link the real code without being committed:

```cmake
cmake_minimum_required(VERSION 3.21)
project(bench LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
add_subdirectory(/path/to/project project-build)
find_package(Qt6 REQUIRED COMPONENTS Test)

add_executable(shot_grid shot_grid.cpp)
target_link_libraries(shot_grid PRIVATE easel_ui Qt6::Test)
```

The same applies to sandbox workarounds: a copied `go.mod` in `/tmp`, a patched config in a temporary directory. Nothing that has to be reverted before delivery should ever be in the tree.

---

## 12. Scrub before anything leaves

Keep a denylist of strings that must never appear in delivered files (internal hostnames, addresses, customer names) *outside* the repo, and check every deliverable against it:

```
grep -rInF -f /secure/denylist.txt --exclude-dir=.git . && echo "SCRUB FAILED" || echo "scrub clean"
```

Real data from networks you don't own shouldn't be in the session at all. The fakes in section 8 are the alternative.

---

## 13. What the sandbox can't tell you

Keep this list in every project's recipe and hand it over with each change:

- No Windows, no macOS, one CPU architecture.
- No GPU: rendering is tested on Mesa's software OpenGL. Driver-specific problems appear only on real hardware.
- No real display: offscreen and virtual-X grabs don't prove how the desktop's own style or scaling looks.
- No input hardware: synthesized pen pressure proves the code path, not how the pen feels.
- No real network devices: fakes prove the protocol handling, not the vendor's quirks you didn't record.
- No OS keyring, no code signing identity, no installers run.