---
title: Reachy Mini Conversation App
emoji: 🎤
colorFrom: red
colorTo: blue
sdk: static
pinned: false
short_description: Talk with Reachy Mini !
tags:
 - reachy_mini
 - reachy_mini_python_app
---

# Reachy Mini conversation app

Conversational app for the Reachy Mini robot combining OpenAI's realtime APIs (including Azure OpenAI) and choreographed motion libraries.

![Reachy Mini Dance](docs/assets/reachy_mini_dance.gif)

## Architecture

The app follows a layered architecture connecting the user, AI services, and robot hardware:

<p align="center">
  <img src="docs/assets/conversation_app_arch.svg" alt="Architecture Diagram" width="600"/>
</p>

## Overview
- Real-time audio conversation loop powered by the OpenAI realtime API (with optional Azure OpenAI support) and `fastrtc` for low-latency streaming.
- Layered motion system queues primary moves (dances, emotions, goto poses, breathing) while blending speech-reactive wobble.
- Async tool dispatch integrates robot motion through a Gradio web UI with live transcripts.

## Installation

> [!IMPORTANT]
> Before using this app, you need to install [Reachy Mini's SDK](https://github.com/pollen-robotics/reachy_mini/).<br>
> Windows support is currently experimental and has not been extensively tested. Use with caution.

### Using uv
You can set up the project quickly using [uv](https://docs.astral.sh/uv/):

```bash
uv venv --python 3.12.1  # Create a virtual environment with Python 3.12.1
source .venv/bin/activate
uv sync
```

> [!NOTE]
> To reproduce the exact dependency set from this repo's `uv.lock`, run `uv sync` with `--locked` (or `--frozen`). This ensures `uv` installs directly from the lockfile without re-resolving or updating any versions.

To include optional dependencies:
```
uv sync --extra reachy_mini_wireless # For wireless Reachy Mini with GStreamer support
```

You can combine extras or include dev dependencies:
```
uv sync --extra reachy_mini_wireless --group dev
```

### Using pip

```bash
python -m venv .venv # Create a virtual environment
source .venv/bin/activate
pip install -e .
```

Install optional extras depending on the feature set you need:

```bash
# Wireless Reachy Mini support
pip install -e .[reachy_mini_wireless]

# Tooling for development workflows
pip install -e .[dev]
```

Some wheels (e.g. PyTorch) are large and require compatible CUDA or CPU builds—make sure your platform matches the binaries pulled in by each extra.

## Optional dependency groups

| Extra | Purpose | Notes |
|-------|---------|-------|
| `reachy_mini_wireless` | Wireless Reachy Mini with GStreamer support. | Required for wireless versions of Reachy Mini, includes GStreamer dependencies.
| `dev` | Developer tooling (`pytest`, `ruff`). | Add on top of base environment.

## Configuration

1. Copy `.env.example` to `.env`.
2. Fill in the required values, notably the OpenAI API key (or Azure OpenAI credentials).

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | Required for OpenAI. Grants access to the OpenAI realtime endpoint.
| `AZURE_OPENAI_API_KEY` | Optional. If set along with `AZURE_OPENAI_ENDPOINT`, Azure OpenAI will be used instead of OpenAI.
| `AZURE_OPENAI_ENDPOINT` | Optional. Your Azure OpenAI endpoint URL (e.g., `https://your-resource.openai.azure.com`). Required when using Azure OpenAI.
| `MODEL_NAME` | Override the realtime model (defaults to `gpt-realtime`).
| `HF_TOKEN` | Optional token for Hugging Face models (used for recorded emotions dataset).

## Running the app

Activate your virtual environment, ensure the Reachy Mini robot (or simulator) is reachable, then launch:

```bash
reachy-mini-conversation-app
```

By default, the app runs in console mode for direct audio interaction. Use the `--gradio` flag to launch a web UI served locally at http://127.0.0.1:7860/ (required when running in simulation mode).

### CLI options

| Option | Default | Description |
|--------|---------|-------------|
| `--gradio` | `False` | Launch the Gradio web UI. Without this flag, runs in console mode. Required when running in simulation mode. |
| `--debug` | `False` | Enable verbose logging for troubleshooting. |
| `--wireless-version` | `False` | Use GStreamer backend for wireless version of the robot. Requires `reachy_mini_wireless` extra to be installed.


### Examples
- Run on hardware with Gradio UI:

  ```bash
  reachy-mini-conversation-app --gradio
  ```

- Run with wireless support (requires `reachy_mini_wireless` extra and daemon started with `--wireless-version`):

  ```bash
  reachy-mini-conversation-app --wireless-version
  ```

- Run with Azure OpenAI:

  Set `AZURE_OPENAI_API_KEY` and `AZURE_OPENAI_ENDPOINT` in your `.env` file, then run normally:

  ```bash
  reachy-mini-conversation-app
  ```

### Troubleshooting

- Timeout error:
If you get an error like this:
  ```bash
  TimeoutError: Timeout while waiting for connection with the server.
  ```
It probably means that the Reachy Mini's daemon isn't running. Install [Reachy Mini's SDK](https://github.com/pollen-robotics/reachy_mini/) and start the daemon.

## LLM tools exposed to the assistant

| Tool | Action | Dependencies |
|------|--------|--------------|
| `move_head` | Queue a head pose change (left/right/up/down/front). | Core install only. |
| `dance` | Queue a dance from `reachy_mini_dances_library`. | Core install only. |
| `stop_dance` | Clear queued dances. | Core install only. |
| `play_emotion` | Play a recorded emotion clip via Hugging Face assets. | Needs `HF_TOKEN` for the recorded emotions dataset. |
| `stop_emotion` | Clear queued emotions. | Core install only. |
| `do_nothing` | Explicitly remain idle. | Core install only. |

## Using custom profiles
Create custom profiles with dedicated instructions and enabled tools! 

Set `REACHY_MINI_CUSTOM_PROFILE=<name>` to load `src/reachy_mini_conversation_app/profiles/<name>/` (see `.env.example`). If unset, the `default` profile is used.

Each profile requires two files: `instructions.txt` (prompt text) and `tools.txt` (list of allowed tools), and optionally contains custom tools implementations.

### Custom instructions
Write plain-text prompts in `instructions.txt`. To reuse shared prompt pieces, add lines like:
```
[passion_for_lobster_jokes]
[identities/witty_identity]
```
Each placeholder pulls the matching file under `src/reachy_mini_conversation_app/prompts/` (nested paths allowed). See `src/reachy_mini_conversation_app/profiles/example/` for a reference layout.

### Enabling tools
List enabled tools in `tools.txt`, one per line; prefix with `#` to comment out. For example:

```
play_emotion
# move_head

# My custom tool defined locally
sweep_look
```
Tools are resolved first from Python files in the profile folder (custom tools), then from the shared library `src/reachy_mini_conversation_app/tools/` (e.g., `dance`). 

### Custom tools
On top of built-in tools found in the shared library, you can implement custom tools specific to your profile by adding Python files in the profile folder. 
Custom tools must subclass `reachy_mini_conversation_app.tools.core_tools.Tool` (see `profiles/example/sweep_look.py`).

### Edit personalities from the UI
When running with `--gradio`, open the “Personality” accordion:
- Select among available profiles (folders under `src/reachy_mini_conversation_app/profiles/`) or the built‑in default.
- Click “Apply” to update the current session instructions live.
- Create a new personality by entering a name and instructions text; it stores files under `profiles/<name>/` and copies `tools.txt` from the `default` profile.

Note: The “Personality” panel updates the conversation instructions. Tool sets are loaded at startup from `tools.txt` and are not hot‑reloaded.




## Development workflow
- Install the dev group extras: `uv sync --group dev` or `pip install -e .[dev]`.
- Run formatting and linting: `ruff check .`.
- Execute the test suite: `pytest`.
- When iterating on robot motions, keep the control loop responsive => offload blocking work using the helpers in `tools.py`.

## License
Apache 2.0
