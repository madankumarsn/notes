---
topic: ROS2/colcon build vs install layout, PYTHONPATH/AMENT_PREFIX_PATH, symlink-install (pikaros2)
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# ROS2 colcon build/install layout and PYTHONPATH/AMENT_PREFIX_PATH (pikaros2)

**One-line idea:**
Colcon builds packages from `src/` into `build/` (scratch) and `install/<pkg>/` (runtime), where `share/` holds non-code data discovered by package name and `lib/` holds executables/importable Python; in pikaros2's Docker images, every new bash shell auto-sources both the ROS underlay and the workspace overlay via `.bashrc`, and `--symlink-install` is turned on by default via `COLCON_DEFAULTS_FILE`.

**Why it exists:**
Colcon separates scratch build artifacts (`build/`) from the deployable runtime tree (`install/<pkg>/`) to support reproducible, per-package isolated installs, and splits `share/` (data looked up by package name, e.g. launch files, configs, URDFs) from `lib/` (executables and importable Python) so `ros2 run`/imports can find things by convention. Docker-based dev environments commonly auto-source the ROS underlay + workspace overlay on every shell so engineers don't have to remember to do it manually each session.

**Math / mechanics:**
Sourcing `install/setup.bash` chains prefixes in order: the ROS underlay (`/opt/ros/<distro>`) first, then this workspace's per-package isolated installs on top. Because colcon's default "isolated" install layout gives each package its own `install/<pkg>/` prefix (rather than merging everything into one shared tree), each package's env hook (generated via `_local_setup_util_sh.py`) *prepends* its own `share/`/`lib/` paths onto `AMENT_PREFIX_PATH`/`PYTHONPATH`/`PATH` — so after sourcing, `AMENT_PREFIX_PATH` lists every `install/<pkg>` individually, workspace packages first and the ROS underlay last. `ros2 run` resolves an executable by walking `AMENT_PREFIX_PATH` and looking under `<prefix>/lib/<package>/` — it does not consult `PATH` for this lookup.

**Code:**
```
install/pika_perception/
  lib/          # executables and Python
  share/        # data ROS looks up by package name
  include/      # C++ headers (C++ packages only)
```

```cmake
install(DIRECTORY cfg DESTINATION share/${PROJECT_NAME})
install(DIRECTORY config DESTINATION share/${PROJECT_NAME})
install(DIRECTORY rviz DESTINATION share/${PROJECT_NAME})
install(DIRECTORY launch DESTINATION share/${PROJECT_NAME})
install(DIRECTORY urdf DESTINATION share/${PROJECT_NAME})
```

```dockerfile
# docker/base/Dockerfile — auto-sourcing every new shell
RUN echo "source /opt/ros/${ROS_DISTRO}/setup.bash" >> /home/${username}/.bashrc
RUN echo "source /home/${username}/project/install/setup.bash" >> /home/${username}/.bashrc
RUN echo "export ROS_WORKSPACE=/home/${username}/project" >> /home/${username}/.bashrc
```

```json
// docker/base/colcon-defaults.yaml (referenced via ENV COLCON_DEFAULTS_FILE)
{
    "build": {
        "symlink-install": true,
        "packages-skip": ["arena_camera_node", "..."]
    }
}
```

```bash
python3 src/pika_perception/scripts/extract_images_from_bag.py /path/to/bag /path/to/out
echo "$AMENT_PREFIX_PATH"
echo "$COLCON_DEFAULTS_FILE"
```

**Gotchas:**
- `src/pika_perception/scripts/*.py` (offline tools like `extract_images_from_bag.py`, `calibrate_camera_to_arm.py`, `pub_tags.py`, `product_mapping/infer.py`) are **not** installed by colcon at all — no `install(PROGRAMS scripts/...)` or `install(DIRECTORY scripts ...)` in `CMakeLists.txt`. They're run directly as `python3 path/to/script.py`, and they still import the installed package (e.g. `from pika_perception.utils_ros import read_messages`) via `PYTHONPATH`, not via anything in `scripts/`. See [[python-pythonpath]] for how that resolution mechanism works in general.
- `ros2 run` finds executables by walking `AMENT_PREFIX_PATH` then looking in `<prefix>/lib/<package>/` — it does **not** use `PATH` for this.
- Why imports "just work" without manually sourcing: the base Docker image appends `source /opt/ros/$ROS_DISTRO/setup.bash` and `source .../install/setup.bash` to `.bashrc`, so every new interactive shell (Docker, `attach.sh`, Cursor terminal) is pre-sourced. Manual `source install/setup.bash` is still needed when: a new package just appeared in `install/` (bashrc sourced the old file), a non-interactive shell skips `.bashrc` (`bash -c`, some CI, some tmux panes), or you're outside this Docker image.
- Editor (Pylance) not showing red underlines is a separate, weaker signal than the runtime env being correct — `pyrightconfig.json` sets `"reportMissingImports": "none"`, so it won't flag missing ROS imports even in a never-sourced shell.
- `install/setup.bash` chains prefixes: underlay `/opt/ros/jazzy` first, then this workspace's per-package isolated installs, each contributing env hooks (`prepend AMENT_PREFIX_PATH/PYTHONPATH/PATH`) via `_local_setup_util_sh.py`. Because install is "isolated" (one prefix per package, not merged), `AMENT_PREFIX_PATH` after sourcing lists every `install/<pkg>` individually, workspace prefixes first, `/opt/ros/jazzy` last.
- `PATH` barely changes from workspace packages since ROS executables live under `lib/<pkg>/`, not a `bin/`, so isolated installs usually add nothing to `PATH` beyond `/opt/ros/jazzy/bin` (where `ros2` itself lives).
- `--symlink-install` isn't passed on the CLI because `docker/base/Dockerfile` sets `ENV COLCON_DEFAULTS_FILE=.../docker/base/colcon-defaults.yaml`, which sets `"symlink-install": true` (and a `packages-skip` list) as defaults for any `colcon build` run in that container. `docker/simulation` and `docker/robot` have their own defaults files doing the same. To build without symlinks, unset that env var or point it elsewhere (e.g. `src/ignore_packages_for_sim.yaml` sets `symlink-install: false`).
- `--symlink-install` only affects files CMake actually installs — it does not change how `scripts/` files behave, since those were never installed in the first place; edits there are live immediately because you execute the source file directly.

**Related:** [[python-pythonpath]]

**Still unclear:**
Raw extraction from a chat, unreviewed — verify before relying on it. The explanation of `PATH`/`PYTHONPATH`/`AMENT_PREFIX_PATH` population was cut off mid-explanation in the original extract ("...So in this scenario `PATH` is 'system + `ros2` from Jazzy'..." then the extract ends), so the full final wording of the `PYTHONPATH`/`AMENT_PREFIX_PATH` sections isn't fully captured. Exact contents of the `packages-skip` list beyond `arena_camera_node` weren't captured either.
