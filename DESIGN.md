# DESIGN.md

This file describes the design intent for the Windows realtime speech recognition app. See `docs/requirements.md` for detailed requirements and `docs/architecture.md` for runtime flow.

## Purpose

The application provides local-first realtime speech recognition on Windows using OpenVINO 2026.2 and the `openvino-genai` `WhisperPipeline`.

The app supports both CLI and PyQt6 GUI modes while sharing the same ASR engine. Users can transcribe microphone input with low latency.

## Design Principles

- Local-first: keep speech recognition processing on the local machine.
- Easy to run: use `uv sync` and `uv run` as the standard workflow.
- Visible failures: report missing models, language mismatches, and audio device problems as readable errors.
- Shared CLI / GUI core: keep audio processing and inference in a shared engine, and separate only the presentation layer.
- Lightweight callback: avoid heavy work inside the audio input callback to preserve realtime behavior.

## Component Structure

The expected main files and responsibilities are:

- `app.py`: entry point, CLI argument parsing, and CLI / GUI mode selection.
- `model_manager.py`: OpenVINO IR model validation, Hugging Face download, and Optimum export.
- `asr_engine.py`: shared engine for microphone input, WebRTC VAD, segment queues, and WhisperPipeline inference.
- `realtime_asr.py`: CLI run loop and transcript output to stdout.
- `asr_gui.py`: PyQt6 GUI, worker object, status display, logs, and transcript display.

## Data Flow

1. `app.py` parses arguments.
2. For `--list-mics`, the app lists microphones without loading a model.
3. If a model is requested, `model_manager.py` resolves either a local IR directory or a Hugging Face model ID.
4. `asr_engine.py` initializes `WhisperPipeline`.
5. `sounddevice.InputStream` captures 16 kHz mono audio in 30 ms frames.
6. Audio frames are passed into a queue.
7. WebRTC VAD extracts speech regions.
8. Completed speech segments are passed to the inference queue.
9. `WhisperPipeline.generate()` transcribes the audio.
10. The CLI or GUI displays results, status, and logs.

## Model Design

The default model is `OpenVINO/whisper-large-v3-turbo-int8-ov`.

Required IR files:

- `openvino_encoder_model.xml`
- `openvino_encoder_model.bin`
- `openvino_decoder_model.xml`
- `openvino_decoder_model.bin`
- `openvino_tokenizer.xml`
- `openvino_detokenizer.xml`

Optional IR files:

- `openvino_decoder_with_past_model.xml`
- `openvino_decoder_with_past_model.bin`

A model directory is not valid merely because it exists. The app must validate required files and `generation_config.json` `lang_to_id`, then clearly report any missing pieces.

## Language Design

The default language is `"<|ja|>"`.

When a user specifies shorthand such as `ja` or `en`, the app should resolve it through the selected model's `generation_config.json` `lang_to_id` map and convert it to the actual token that can be passed to `WhisperPipeline.generate()`.

Language resolution failures should be treated as startup configuration errors instead of being delayed until the first inference call.

## GUI Design

The GUI is a work-focused screen where users can understand speech recognition state at a glance. It should include:

- Model and inference device display.
- Microphone selection.
- Start / Stop / Clear actions.
- Current status.
- Transcript display.
- Log display.

The GUI worker owns the ASR engine and notifies the UI thread through Qt signals. This prevents inference and audio processing from freezing the UI.

## Color Balance

The GUI color design should use a pear-inspired, calm color balance. As a speech recognition app, it should remain comfortable during long viewing sessions while making state changes gently noticeable.

Recommended palette:

- Background: pale yellow-green pear flesh `#F3F4D5`
- Main surface: cream-white `#FFFDF2`
- Primary: pear-skin green `#9CB35B`
- Primary emphasis: deeper leaf green `#5F7334`
- Accent: ripe pear yellow `#D8B84E`
- Border: soft seed-and-stem brown `#B7A77A`
- Text: readable green-tinted charcoal `#263024`
- Error: muted rose brown `#A6534B`

Suggested color ratio:

- 60%: pale pear flesh and cream tones for backgrounds, panels, and inputs.
- 25%: pear green for primary buttons, selected states, and status indicators.
- 10%: yellow accent for attention, recording state, and highlights.
- 5%: dark text, borders, and error color.

For recording and transcription states, combine pear green with the yellow accent instead of strong reds or blues. Reserve rose brown for errors so error states remain clearly distinct from normal operation.

## Error Handling Policy

- Treat missing required model files as configuration errors.
- Treat missing optional model files as warnings.
- Fail at startup when `generation_config.json` or `lang_to_id` is missing.
- Set UTF-8 related environment variables for model export subprocesses to avoid failures in Windows `cp932` terminals.
- User-facing errors should clearly explain the cause and the next thing to check.

## Future Extensions

- TXT / SRT transcript export.
- GUI settings persistence.
- Multiple microphone profiles.
- Device-specific inference status display.
- Segment-level timestamp display.

