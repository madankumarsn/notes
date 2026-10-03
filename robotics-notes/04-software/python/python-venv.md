---
topic: Python virtual environments (venv)
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Python virtual environments (venv)

**One-line idea:**
A venv is a self-contained Python interpreter + `site-packages` tree for one project; create it with `python -m venv .venv` (or `uv venv`), activate with `source .venv/bin/activate` so `python`/`pip` resolve inside it, and leave it with `deactivate`.

**Why it exists:**
Python installs packages into a shared global `site-packages` by default; different projects often need different, conflicting package versions. A venv gives each project its own isolated interpreter + `site-packages` so installs don't collide or require root.

**Math / mechanics:**
Operational mechanics (general knowledge):
- `python -m venv .venv` copies/symlinks a Python interpreter into `.venv/bin/` and creates an empty `.venv/lib/pythonX.Y/site-packages/`.
- The activation script (`.venv/bin/activate`) prepends `.venv/bin` to `$PATH` and sets `$VIRTUAL_ENV`; it does **not** change `$PYTHONPATH`. That's why `python`/`pip` resolve into the venv — it's a `PATH` lookup, not interpreter magic.
- `pip install` inside an activated venv installs into that venv's `site-packages`, leaving the system/other venvs untouched.

**Code:**
```bash
# create
python3 -m venv .venv          # stdlib
# or
uv venv                        # uv, faster resolver/creator

# activate (bash/zsh)
source .venv/bin/activate
# fish
source .venv/bin/activate.fish
# PowerShell
.venv\Scripts\Activate.ps1

# verify
which python                   # should point into .venv/bin

# work, then
deactivate
```

**Gotchas:**
- After activation the shell prompt is prefixed with `(.venv)`; `python`/`pip` then resolve to the venv's own versions rather than the system ones.
- The `.venv/bin/activate` script bakes in an **absolute path** to the venv — moving/renaming the project directory breaks activation; delete and recreate the venv instead of copying it.
- `.venv/` is normally git-ignored; dependencies should be pinned in `requirements.txt` / `pyproject.toml` instead, not recreated from the venv's contents.
- ROS2 interaction (unverified, flagged below): sourcing a ROS2 `setup.bash` adjusts `PYTHONPATH` to inject ROS2's own Python packages. Activating a venv *after* sourcing ROS2 can shadow or conflict with those packages depending on order and on whether the venv was created with `--system-site-packages`. Verify the actual behavior against this project's setup before relying on it.

**Related:** [[]]

**Still unclear:**
- Only the one-line idea / activation command / prompt gotcha trace back to an actual sourced chat. Everything else in this note (creation commands, mechanics, git-ignore convention, `uv venv`, ROS2 interaction) is general knowledge added at your request, not verified against a specific doc or transcript — flagging per vault rules so you know what to double-check before marking this `reviewed`.
- The ROS2 `setup.bash` + venv interaction in particular is a known general gotcha class but not confirmed against any specific project's actual workspace layout.
