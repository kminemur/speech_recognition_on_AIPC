# AGENTS.md

This file provides working instructions for AI agents and development-assist tools operating in this repository.

## Core Policy

- Treat the existing documentation as the primary source of truth.
  - Use `docs/requirements.md` for requirements.
  - Use `docs/architecture.md` for architecture and runtime flow.
  - Treat the README as the user-facing entry point.
- When a change makes implementation and documentation diverge, update the related documentation along with the implementation.
- Assume local Windows execution.
- Use PowerShell / `cmd` and `uv` as the standard operation path.

## Target Application

This repository contains a Windows realtime speech recognition app built with OpenVINO 2026.2 and the `openvino-genai` `WhisperPipeline`.

Main execution modes:

- CLI: `uv run python app.py`
- GUI: `uv run python app.py --gui`
- List microphones: `uv run python app.py --list-mics`

## Implementation Rules

- Target Python 3.12+.
- Keep dependencies centralized in `pyproject.toml`; use `uv sync` for environment setup.
- Treat model download, validation, and export as application startup responsibilities.
- `--list-mics` must not require model loading or model export.
- `--gui` and `--cli` must be mutually exclusive.
- The default language must be `"<|ja|>"`.
- Normalize shorthand values such as `ja` and `en` using the selected model's `generation_config.json` `lang_to_id` map.
- Pass only the resolved language token to `WhisperPipeline.generate()`, not raw user input.
- Keep the audio callback lightweight.
  - The callback should only hand frames to a queue.
  - VAD, segmentation, and ASR inference must run outside the callback.

## Model Handling

- Accept local OpenVINO IR directories through `--model`.
- Accept Hugging Face model IDs through `--model-id`.
- If neither is provided, use `OpenVINO/whisper-large-v3-turbo-int8-ov` as the default.
- Use `.cache_whisper` as the default cache directory.
- Do not treat a directory as a valid model only because it exists.
- If required files are missing from a local model, fail immediately with a clear error.
- If required files are missing from a downloaded or exported model, try to refresh it by downloading again or exporting again.
- Treat `openvino_decoder_with_past_model.xml/.bin` as optional files; warn when they are missing and continue.
- When running Optimum export on Windows, set UTF-8 related environment variables for the subprocess.
  - `PYTHONIOENCODING=utf-8`
  - `PYTHONUTF8=1`
  - `NO_COLOR=1`

## UI / GUI Rules

- Use PyQt6 for the GUI.
- The GUI should make the model, device, microphone, status, logs, and transcripts easy to inspect.
- The GUI worker should own the engine and forward status, logs, and transcript events to the UI through Qt signals.
- When changing UI colors or visual styling, follow the color balance defined in `DESIGN.md`.

## Testing And Verification

Depending on the change, consider at least the following checks.

```powershell
uv sync
uv run python app.py --list-mics
uv run python app.py --help
uv run python app.py
uv run python app.py --gui
```

Checks that require model download or inference can take time. In work summaries, explicitly state whether those checks were run.

## Files To Avoid And Cautions

- Do not commit `.venv`, `.cache_whisper`, model caches, or generated large model files.
- Do not revert user work or unrelated uncommitted changes.
- Do not mix unrelated refactors or formatting churn into focused changes.
- If implementation files do not exist yet, create them according to the file names and responsibilities defined in the documentation.

