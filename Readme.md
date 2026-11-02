# OpenVINO Realtime Speech Recognition on AI PC

Windows PC でマイク音声をリアルタイム文字起こしするためのアプリです。
OpenVINO 2026.2 と `openvino-genai` の `WhisperPipeline` を使い、CLI と PyQt6 GUI の両方から同じ音声認識エンジンを利用します。

標準では日本語認識向けに `OpenVINO/whisper-large-v3-turbo-int8-ov` を使います。

## 主な機能

- マイク入力のリアルタイム文字起こし
- CLI / GUI の2つの起動モード
- OpenVINO IR 形式のローカルモデル指定
- Hugging Face モデルIDからのモデル取得
- モデルに応じた言語トークン解決
- WebRTC VAD による発話区間検出
- CPU / GPU / NPU / AUTO デバイス指定

## 動作環境

- Windows
- Python 3.12 以上、または `uv` 管理の Python
- PowerShell または `cmd`
- `uv`
- OpenVINO 2026.2 対応環境

## セットアップ

依存関係は `pyproject.toml` に集約しています。

```powershell
uv sync
```

`uv sync` により `.venv` が作成され、実行に必要なパッケージがインストールされます。

## 起動方法

CLI で起動:

```powershell
uv run python app.py
```

GUI で起動:

```powershell
uv run python app.py --gui
```

マイク一覧を表示:

```powershell
uv run python app.py --list-mics
```

`--list-mics` はモデルの読み込みやエクスポートを行わず、音声デバイスの確認だけを実行します。

## よく使うオプション

```powershell
uv run python app.py --device GPU --language ja
uv run python app.py --gui --device NPU --mic 1
uv run python app.py --model .\models\whisper-large-v3-turbo-int8-ov
uv run python app.py --model-id OpenVINO/whisper-large-v3-turbo-int8-ov
```

主なオプション:

- `--gui`: GUI モードで起動
- `--cli`: CLI モードで起動
- `--list-mics`: 利用可能なマイクを表示
- `--model`: ローカルの OpenVINO IR モデルディレクトリ
- `--model-id`: Hugging Face のモデルID
- `--model-cache-dir`: モデルキャッシュ先。既定値は `.cache_whisper`
- `--device`: 推論デバイス。例: `AUTO`, `CPU`, `GPU`, `NPU`
- `--mic`: 使用するマイクのインデックス
- `--language`: 認識言語。既定値は `"<|ja|>"`
- `--task`: `transcribe` または `translate`
- `--weight-format`: Optimum export 時の重み形式。例: `int8`, `fp16`, `fp32`

`--gui` と `--cli` は同時に指定できません。

## モデル

既定モデル:

```text
OpenVINO/whisper-large-v3-turbo-int8-ov
```

モデル指定がない場合、このモデルを `.cache_whisper` に取得して使用します。

ローカルモデルを使う場合は、OpenVINO IR ディレクトリを `--model` に指定します。

```powershell
uv run python app.py --model .\models\my-whisper-ov
```

Hugging Face モデルIDを指定する場合:

```powershell
uv run python app.py --model-id OpenVINO/whisper-large-v3-turbo-int8-ov
```

必要なモデルファイル:

- `openvino_encoder_model.xml`
- `openvino_encoder_model.bin`
- `openvino_decoder_model.xml`
- `openvino_decoder_model.bin`
- `openvino_tokenizer.xml`
- `openvino_detokenizer.xml`
- `generation_config.json`

任意のモデルファイル:

- `openvino_decoder_with_past_model.xml`
- `openvino_decoder_with_past_model.bin`

ローカルモデルは、ディレクトリが存在するだけでは有効とはみなしません。必要ファイルが不足している場合は、起動時に明確なエラーとして扱います。

Hugging Face から取得したモデル、または Optimum でエクスポートしたモデルで必要ファイルが不足している場合は、再取得または再エクスポートを試みます。

## 言語指定

既定の言語は日本語です。

```text
<|ja|>
```

`--language ja` や `--language en` のような短縮指定も受け付けます。実行時には、選択されたモデルの `generation_config.json` に含まれる `lang_to_id` を使って、`WhisperPipeline.generate()` に渡せる正式な言語トークンへ変換します。

モデル側に指定言語が存在しない場合は、最初の推論まで待たずに起動時エラーとして扱います。

## GUI

GUI は PyQt6 で実装します。

表示・操作できる主な項目:

- モデル
- 推論デバイス
- マイク
- 実行状態
- ログ
- 文字起こし結果
- 開始 / 停止 / クリア操作

GUI ワーカーが音声認識エンジンを所有し、状態・ログ・文字起こし結果を Qt signal 経由で UI に通知します。

## 実装方針

音声入力の callback は軽量に保ちます。callback 内では音声フレームをキューに渡すだけにし、VAD、発話区間の分割、ASR 推論は callback の外で実行します。

想定している主要ファイル:

- `app.py`: エントリーポイント、引数解析、CLI / GUI モード選択
- `model_manager.py`: モデル検証、ダウンロード、Optimum export
- `asr_engine.py`: マイク入力、VAD、セグメント処理、WhisperPipeline 推論
- `realtime_asr.py`: CLI 実行ループ
- `asr_gui.py`: PyQt6 GUI
- `pyproject.toml`: 依存関係とプロジェクト設定
- `DESIGN.md`: GUI とアプリ全体の設計方針

## 動作確認

変更内容に応じて、以下を確認します。

```powershell
uv sync
uv run python app.py --help
uv run python app.py --list-mics
uv run python app.py
uv run python app.py --gui
```

モデルのダウンロードや推論を含む確認は時間がかかる場合があります。

## キャッシュと生成物

以下はリポジトリに含めません。

- `.venv`
- `.cache_whisper`
- モデルキャッシュ
- 生成された大きなモデルファイル

## 関連ドキュメント

- `DESIGN.md`: アプリの設計意図、GUI 方針、配色、コンポーネント構成
- `AGENTS.md`: 開発エージェント向けの作業ルール
