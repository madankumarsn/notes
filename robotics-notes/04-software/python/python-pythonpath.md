---
topic: Python import resolution — sys.path, PYTHONPATH, python environment
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Python import resolution — sys.path, PYTHONPATH, python environment

**One-line idea:**
Every `import` is resolved by searching `sys.path`, an ordered list of directories; `PYTHONPATH` is an environment variable whose entries get folded into `sys.path` at interpreter startup, letting you add import locations without installing anything.

**Why it exists:**
There needs to be a way to make code importable from somewhere other than the installed `site-packages` — workspace-local packages, an in-development checkout, or (in ROS2) an overlay of built packages — without actually packaging/installing it first.

**Math / mechanics:**
Resolution order (general knowledge):
- `sys.path` is built, roughly in this order, at interpreter startup: (1) the running script's directory (or `''`/cwd for `-c`, `-m`, and the REPL), (2) each directory listed in `PYTHONPATH`, (3) the standard library's own directories, (4) `site-packages`, populated by the `site` module — including extra paths injected by `.pth` files (e.g. from `pip install -e`).
- `PYTHONPATH` is a list of directories separated by the OS path separator (`:` on Linux/macOS, `;` on Windows), same format as `PATH`.
- Name collisions resolve to whichever matching entry comes **first** in `sys.path` — an earlier `PYTHONPATH` directory silently shadows a same-named package installed later in the search order.
- A venv does **not** use `PYTHONPATH` to find its own packages — it works because the `python` binary inside `.venv/bin/` has its own `sys.prefix`, and the `site` module adds that venv's `site-packages` based on *which interpreter is running*, independent of `PYTHONPATH`.

**Code:**
```bash
# inspect current resolution
python3 -c "import sys; print('\n'.join(sys.path))"
echo "$PYTHONPATH"

# add a directory for one shell/session
export PYTHONPATH="/path/to/some/package:$PYTHONPATH"

# add for one invocation only
PYTHONPATH=/path/to/some/package python3 script.py
```

**Gotchas:**
- `PYTHONPATH` persists for every `python` invocation in that shell session — easy to end up with a stale directory still exported (e.g. from an old workspace or branch) silently shadowing the package you actually meant to use.
- A relative path put into `PYTHONPATH` is resolved relative to the **current working directory at import time**, not at the time it was set — fragile if you `cd` around.
- `.pth` files dropped into `site-packages` (commonly by `pip install -e`) add directories to `sys.path` invisibly, outside of `PYTHONPATH` entirely — another source of "where is this importing from?" confusion.
- In ROS2 specifically, sourcing `install/setup.bash` works by prepending each package's `site-packages`-equivalent directory onto `PYTHONPATH` (and `AMENT_PREFIX_PATH`) — same mechanism as above, just automated. See [[ros2-colcon-build-install-layout]] for the ROS2-specific details.

**Related:** [[python-venv]], [[ros2-colcon-build-install-layout]]

**Still unclear:**
This note is general Python-mechanism knowledge, not drawn from a specific chat or doc — flagging per vault rules so it gets verified before ever being marked `reviewed`. Edge cases not covered: zip-imports, frozen/packaged apps (PyInstaller etc.), and namespace packages, none of which were covered in any source material.
