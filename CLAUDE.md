# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Google Research Football — a reinforcement-learning environment built on top of the open-source
Gameplay Football game. A Python `gym` package (`gfootball/`) wraps a C++ physics/game engine
(`third_party/gfootball_engine/`) that is compiled into a native module (`_gameplayfootball.so` /
`libgame.so`) and imported from Python as `gfootball_engine`.

## Build & install

The package cannot run until the C++ engine is compiled — `football_env_core.py` imports
`gfootball_engine as libgame`, and that fails if the native library is missing.

- Full install (compiles C++, can take several minutes): `python3 -m pip install .`
- Editable/dev install: `python3 -m pip install -e .` — on non-Windows this symlinks
  `third_party/gfootball_engine` into the repo root as `gfootball_engine`.
- Use a pre-built binary instead of compiling: set `GFOOTBALL_USE_PREBUILT_SO=1` before pip install
  (copies `third_party/gfootball_engine/lib/prebuilt_gameplayfootball.so`).
- The compile is driven by `setup.py` (`CMakeExtension` / `CustomBuild`), which shells out to the
  platform build script: `gfootball/build_game_engine.sh` (Linux/macOS, runs `cmake . && make`) or
  `gfootball/build_game_engine.bat` (Windows). CMake config lives in
  `third_party/gfootball_engine/CMakeLists.txt`.
- Windows source builds need extra tooling and an env var — see `gfootball/doc/compile_engine.md`.
- `gym` is pinned `<=0.21.0`; training scripts additionally need TensorFlow 1.15, dm-sonnet 1.x, and
  OpenAI Baselines (not in `requirements.txt`).

## Common commands

- Play interactively vs. built-in AI: `python3 -m gfootball.play_game --action_set=full`
  (full flag list: `python3 -m gfootball.play_game -helpfull`).
- Run a single test (tests are `absltest` files named `*_test.py`, runnable as modules):
  `python3 -m gfootball.env.wrappers_test` (same pattern for `football_env_test`,
  `football_action_set_test`, `observation_rotation_test`, `script_helpers_test`, `gym_test`).
- Run the full test + example suite in Docker: `./run_docker_test.sh` (builds the image, then runs
  every `gfootball/env/*_test.py` with `UNITTEST_IN_DOCKER=1 PYTHONPATH=/`, and a short PPO run).
- Example PPO training: `python3 -m gfootball.examples.run_ppo2 --level=academy_empty_goal_close`
  (add `--render=True --dump_full_episodes=True` to save replays).

## Architecture

The single public entry point is **`create_environment(...)` in `gfootball/env/__init__.py`**. It
builds a `config.Config`, instantiates the env, and then stacks gym wrappers. The layering is the
key mental model:

```
create_environment
  → config.Config / ScenarioConfig      # loads the scenario module, resolves options
  → football_env.FootballEnv (gym.Env)  # env/football_env.py — action/obs plumbing
     → football_env_core.FootballEnvCore # env/football_env_core.py — talks to C++ engine
        → gfootball_engine (libgame)     # compiled C++ from third_party/gfootball_engine
  → wrappers (env/wrappers.py)           # representation + reward + agent-count + stacking
```

`_apply_output_wrappers` in `env/__init__.py` selects wrappers from CLI/API options:
- **Representation** (`representation=`): `raw` (no wrapper), `simple115`/`simple115v2`
  (`Simple115StateWrapper`, 115-float vector), `extracted`/SMM (`SMMWrapper`, minimap planes),
  `pixels`/`pixels_gray` (`PixelsStateWrapper`).
- **Reward** (`rewards=`): always `scoring`; optional `checkpoints` adds `CheckpointRewardWrapper`.
- Single-agent reduction (`SingleAgentObservationWrapper`/`SingleAgentRewardWrapper`) when exactly
  one player is agent-controlled; optional `FrameStack` (4 frames); `GetStateWrapper` last.

At import time, `gfootball/__init__.py` registers a gym id per scenario per representation
(`GFootball-<scenario>-SMM-v0`, `-Pixels-v0`, `-simple115-v0`, `-simple115v2-v0`).

### Scenarios (`gfootball/env/scenarios/`)

Each scenario is a standalone `.py` module exposing `build_scenario(builder)`. It sets
`builder.config()` fields (e.g. `game_duration`, `deterministic`, `end_episode_on_score`), then
`SetBallPosition`, `SetTeam(Team.e_Left/e_Right)`, and `AddPlayer(x, y, role)`. The builder API is
`env/scenario_builder.py`; `all_scenarios()` there is what drives gym registration. To add a level,
drop a new module here — no other registration is needed.

### Players (`gfootball/env/players/`)

Pluggable controllers selected via the `players=`/`--players` string
(`"<type>:left_players=?,right_players=?,<param>=?"`). Types: `agent` (external RL agent),
`bot` (scripted), `keyboard`, `gamepad`, `ppo2_cnn` (loads a TF checkpoint), `replay`, `lazy`.

### Other directories

- `gfootball/examples/` — training entry points (`run_ppo2.py`, `run_multiagent_rllib.py`,
  `models.py`, repro shell scripts).
- `gfootball/eval_server/` — remote/competition evaluation server; pairs with
  `env/remote_football_env.py` and `create_remote_environment` in `env/__init__.py`.
- `gfootball/doc/` — authoritative reference for API, observations/actions, scenarios, multi-agent,
  docker, replays, and engine compilation. Consult these before changing env semantics.
- `gfootball/{replay,dump_to_txt,dump_to_video}.py` — tools for saved episode traces/dumps.

## Conventions

- Python 2/3-compatible style (`from __future__ import ...`, `six`) is used throughout — match it in
  existing files rather than modernizing.
- Every source file carries the Apache 2.0 license header; keep it on new files.
- If you change dependencies, update **both** `requirements.txt` and `setup.py` (noted explicitly in
  `requirements.txt`).
