# codex-image-gen

*[日本語](#日本語) / [English](#english)*

---

## 日本語

Claude Code 用スキル。Codex CLI の組み込み画像生成ツール（`image_gen`）を呼び出し、
ChatGPT サブスクリプション内（追加課金ゼロ・APIキー不要）で画像を生成する。

「画像生成して」「Codexで画像」「バナー作って」「アイコン生成」「サムネ作って」等のキーワードで起動する。

### できること

- バナー・LPヒーロー画像などの写真/イラスト調の背景ビジュアル生成
- アイコンの AI 生成 → ベクター化(既製アイコンセットとの使い分けも解説)
- ロゴ・文言は AI に描かせず、生成した背景に HTML+Chrome で後から合成
- 納品前チェック(拡大確認によるAI特有の破綻の検出)の手順

含まないもの: インフォグラフィック・図解・グラフ(別途 HTML+Chrome / `dataviz` スキル側で対応)。

### インストール

このリポジトリの `codex-image-gen/` フォルダを、Claude Code の skills ディレクトリにコピーする。

```bash
# プロジェクト単位で使う場合
cp -r codex-image-gen /path/to/project/.claude/skills/

# 全プロジェクト共通で使う場合
cp -r codex-image-gen ~/.claude/skills/
```

### 前提

- Codex CLI がインストール済み・ログイン済みであること(`codex login`)
- `~/.codex/auth.json` の `auth_mode` が `chatgpt` であること(`api_key` の場合は API 従量課金が発生する)

詳細な手順・注意点は [`codex-image-gen/SKILL.md`](./codex-image-gen/SKILL.md) を参照(日本語)。

---

## English

A Claude Code skill that drives Codex CLI's built-in image generation tool (`image_gen`)
to create images inside your existing ChatGPT subscription — no extra billing, no API key.

Triggers on prompts like "generate an image", "make a banner with Codex", "create an icon", "make a thumbnail", etc.

### What it does

- Generates photo/illustration-style background visuals (banners, LP hero images, etc.)
- AI-generates icons and vectorizes them (also covers when to use a ready-made icon set instead)
- Never lets the AI draw logos or copy — those are composited afterward onto the generated background via HTML+Chrome
- Includes a pre-delivery QA routine (zoomed-in checks for AI-specific artifacts like broken hands/text)

Not covered: infographics, diagrams, or charts (handled separately via HTML+Chrome / the `dataviz` skill).

### Install

Copy this repo's `codex-image-gen/` folder into a Claude Code skills directory.

```bash
# Per-project
cp -r codex-image-gen /path/to/project/.claude/skills/

# Global (all projects)
cp -r codex-image-gen ~/.claude/skills/
```

### Prerequisites

- Codex CLI installed and logged in (`codex login`)
- `auth_mode` in `~/.codex/auth.json` must be `chatgpt` (if it's `api_key`, API usage will be billed)

Full instructions and caveats (Japanese only) are in [`codex-image-gen/SKILL.md`](./codex-image-gen/SKILL.md).
