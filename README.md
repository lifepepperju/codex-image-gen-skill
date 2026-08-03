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

### 用途別の使い分け(条件分岐)

「何を作りたいか」によって、内部で使う手段が自動的に切り替わる。この判断を誤ると
プロンプトをどれだけ工夫しても品質が上がらないため、スキルの中核ロジックになっている。

| 作りたいもの | 使う手段 |
|---|---|
| 写真/イラスト調の背景ビジュアル(バナー、LPヒーロー、SNS投稿の絵など) | `image_gen`(AI生成) |
| ロゴ・見出し・コピーなどの文字要素 | AIには描かせない。実ロゴファイル＋実フォントを HTML+Chrome で後から合成する |
| インフォグラフィック・図解・比較表・手順図 | HTML+Chrome のみで作成する(AI生成は使わない) |
| アイコン | ①AI生成→ベクター化(質重視・自由度が高い) または ②既製アイコンセット(速度重視・画風が揃う) |
| データのグラフ・チャート | HTML/SVG ＋ `dataviz` スキル ＋ 実データ(数値は作らない) |
| 実際の地図・地形 | 地理データSVG(パブリックドメイン等)を別途調達する。アイコンセットには無い |
| 人物の顔を似せたい | `image_gen` は参照画像を受け取れない。写真の特徴をまず英語で言語化し、その描写文をプロンプトに焼き込む2段階方式を使う |
| UI・画面モック(見た目の方向性確認のみ) | `image_gen` を例外的に使用。画面構成の詰め・実装への受け渡しは Claude Design が本筋(この画像は実装仕様として渡さない) |

### 対応する主な出力サイズ(媒体別)

`image_gen` はプロンプトでピクセル数を直接指定できない(縦横比の言葉でしか寄せられない)ため、
生成後に `sips` 等で目的サイズへリサイズする。よく使うサイズ:

| 用途 | サイズ |
|---|---|
| Instagram 正方形 | 1080×1080 |
| Meta広告 横長 | 1200×628 |
| Instagramストーリー | 1080×1920 |
| Xカード | 1200×675 |
| YouTubeサムネイル | 1280×720 |
| OGP画像 | 1200×630 |
| PC画面モック | 1600×900 / 1440×900 |
| スマホ画面モック | 390×844 |

### SVGへの出力について

- `image_gen`(AIモデル)自体は**ラスター(PNG)のみ**を出力する。SVGを直接生成することはできない。
- ただしスキルの後工程で、以下の2箇所は SVG 出力に対応している。
  - **アイコン**: 生成した PNG を `potrace` でベクター化し、`currentColor` に対応した SVG として出力する(CSSで色を変えられる)
  - **バナー等の合成物**: 背景はラスターのまま、ロゴ(ベクター)・見出し・サブコピー・CTA をテキストレイヤーとして持つ
    SVG を別途生成できる。PNG とほぼ同一の見た目になり、Figma に読み込むと「背景=画像レイヤー / ロゴ=ベクター / 文字=テキストレイヤー」として編集できる
- 透過背景は非対応(既定モデル `gpt-image-2` は透過を出力できない)。必要な場合は生成後に背景を抜く。

### 同梱の関連エージェント

SKILL.md セクション7.5(マーケレビュー・ゲート)から呼び出される、マーケティング事業部の4エージェントを
`agents/` フォルダに同梱した。エージェント未導入の環境でも、このリポジトリだけでレビュー呼び出し先が揃う。

| エージェント | 役割 | 呼ばれる場面 |
|---|---|---|
| `mkt-visual-creative` | バナー・動画等ビジュアル制作の専任 | ビジュアル仕様・媒体別の作り分けを相談するとき |
| `mkt-paid` | 広告運用・メディアプラン | Paid配信の審査・入稿要件が絡むとき |
| `mkt-social` | SNSオーガニック運用 | オーガニック投稿の設計を相談するとき |
| `mkt-cmo` | マーケ領域の複合タスクの入口 | 複数媒体・複数区分にまたがるとき |

**注意**: これらのエージェント自身は、社内の `ads-*` スキル群や他の `mkt-*` エージェント(`mkt-strategist` / `mkt-text-content` 等)、
`workflows.md` などをさらに参照している箇所がある。それらは本リポジトリには含まれていないため、参照先が無い場合は
その部分の指示が使えないか、エージェントが「該当スキルが見当たらない」と応答することがある。最低限、上記4体を導入すれば
`codex-image-gen` 本体からのマーケレビュー呼び出しは機能する。

```bash
# プロジェクト単位で使う場合
cp agents/*.md /path/to/project/.claude/agents/

# 全プロジェクト共通で使う場合
cp agents/*.md ~/.claude/agents/
```

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

### Decision logic: which tool for which asset

What you're trying to create determines which method the skill uses internally. Getting this
branch wrong is the most common failure mode — no amount of prompt tweaking fixes it — so it's
the core logic of the skill.

| What you want | Method used |
|---|---|
| Photo/illustration-style background visual (banner, LP hero, social post art) | `image_gen` (AI generation) |
| Logo, headline, or copy | Never AI-drawn — composited afterward onto the background using the real logo file and real fonts via HTML+Chrome |
| Infographics, diagrams, comparison tables, step-by-step illustrations | HTML+Chrome only (no AI generation) |
| Icons | ① AI-generate then vectorize (higher quality, more freedom) or ② use a ready-made icon set (faster, consistent style) |
| Data charts/graphs | HTML/SVG + the `dataviz` skill + real data (never fabricate numbers) |
| Real maps/terrain | Sourced separately as geographic SVG data (e.g. public domain) — not available in icon sets |
| Matching a real person's face | `image_gen` can't take a reference image. Instead, the photo's features are described in English first, then that description is baked into the generation prompt (two-step relay) |
| UI/screen mockups (rough look-and-feel only) | `image_gen` used as an exception; actual screen design and implementation handoff goes through Claude Design instead (this image is never handed off as a spec) |

### Common output sizes (by platform)

`image_gen` can't be told an exact pixel size directly (only aspect ratio, via wording like
"tall 9:16 composition"), so the skill resizes to the target dimensions afterward with `sips`.
Frequently used sizes:

| Use case | Size |
|---|---|
| Instagram square | 1080×1080 |
| Meta Ads landscape | 1200×628 |
| Instagram Story | 1080×1920 |
| X (Twitter) card | 1200×675 |
| YouTube thumbnail | 1280×720 |
| OGP image | 1200×630 |
| Desktop UI mockup | 1600×900 / 1440×900 |
| Mobile UI mockup | 390×844 |

### Does it output SVG?

- The AI model (`image_gen`) itself only outputs **raster PNGs** — it cannot generate SVG directly.
- The skill's post-processing pipeline does add SVG output in two places:
  - **Icons**: the generated PNG is vectorized with `potrace` into an SVG that supports `currentColor` (so its color can be controlled via CSS)
  - **Composited banners**: alongside the PNG, the skill can emit an SVG with the background as a raster image layer, the logo as a vector, and the headline/subcopy/CTA as editable text layers — nearly identical in appearance to the PNG, and editable in Figma as "background = image layer / logo = vector / text = text layers"
- Transparent backgrounds are not supported (the default model, `gpt-image-2`, cannot output transparency). If needed, the background must be removed afterward.

### Bundled related agents

The 4 marketing-department agents called from the marketing-review gate (SKILL.md section 7.5) are bundled
in the `agents/` folder, so the review call targets exist even in an environment where the org's agents
aren't installed yet.

| Agent | Role | When it's called |
|---|---|---|
| `mkt-visual-creative` | Dedicated visual production (banners, video, etc.) | Consulting on visual specs / per-platform variants |
| `mkt-paid` | Paid ad ops & media planning | When paid-distribution review/submission requirements are involved |
| `mkt-social` | Organic social ops | Designing organic post placement |
| `mkt-cmo` | Entry point for cross-cutting marketing tasks | When the task spans multiple platforms/categories |

**Note**: these agents themselves reference further internal resources in places — the `ads-*` skill suite,
other `mkt-*` agents (`mkt-strategist`, `mkt-text-content`, etc.), and a `workflows.md` file — none of which
are included in this repo. If those aren't present, that part of the instructions simply won't be usable, or
the agent may report it can't find the referenced skill. Installing just these 4 agents is enough for the
marketing-review gate inside `codex-image-gen` itself to work.

```bash
# Per-project
cp agents/*.md /path/to/project/.claude/agents/

# Global (all projects)
cp agents/*.md ~/.claude/agents/
```

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
