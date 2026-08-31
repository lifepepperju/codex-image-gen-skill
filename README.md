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
- **自分で座標を書いた成果物は SVG も既定で出す。** 線引きは「誰がレイアウトを決めたか」(SKILL.md セクション7.6)。
  - **アイコン**: 生成した PNG を `potrace` でベクター化し、`currentColor` に対応した SVG として出力する(CSSで色を変えられる)。**SVGが主**でPNGは出さない
  - **バナー等の合成物・データのグラフ**: 背景はラスターのまま、ロゴ(ベクター)・見出し・サブコピー・CTA をテキストレイヤーとして持つ
    SVG を **PNG と併せて既定で出力する**。Figma に読み込むと「背景=画像レイヤー / ロゴ=ベクター / 文字=テキストレイヤー」として編集できる
  - **インフォグラフィック・図解**: 要求時のみ(HTML で組んでいるため、SVG 版は手座標での組み直しになる)
- **AI が絵として描いたもの(写真風ビジュアル・UI画面モック)には SVG を出さない。** ベクター化する手段が無く、PNG を包んだ SVG は
  `<image>` タグ1個の箱になる。実測では 1.03MB の PNG が 1.37MB(**1.33倍**)になり、編集できる要素は**1個だけ**だった
- 透過背景は非対応(既定モデル `gpt-image-2` は透過を出力できない)。必要な場合は生成後に背景を抜く。

### 依存するエージェント・スキルは無い(自己完結)

このスキルは**単体で動く**。別途エージェントを入れる必要はない。

以前は納品前のレビュー工程で社内のマーケエージェント(`mkt-visual-creative` / `mkt-paid` / `mkt-social` / `mkt-cmo`)を
呼ぶ設計だったが、それらの中身を精査したところ、画像1枚のレビューに実際に使えるのは全体のごく一部だった
(大半は広告アカウント監査・メディアプラン・投稿カレンダー等でこのスキルとは無関係、かつ社内の別スキル群が前提)。
そのため**必要な観点だけを SKILL.md セクション7.5 に直接書き起こし、エージェント依存を無くした**。

| 元のエージェント | 取り込んだ観点 |
|---|---|
| `mkt-visual-creative` | 生成前の確認項目(ブランドGL・NG表現・避けたい色味・既存素材)、権利と出所の記録 |
| `mkt-paid` | 法規制の観点(薬機法・景表法) |
| `mkt-social` | 媒体ごとの作法(1枚を全媒体に使い回さない)、ステマ規制 |
| `mkt-cmo` | 取り込みなし(振り分け役のみで、レビューの知識を持たないため) |

社内のマーケエージェントが導入済みの環境なら、同じ材料を渡して意見を求めてもよい(任意・無くても成立する)。

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
- **Anything whose layout you authored yourself gets an SVG by default.** The dividing line is who decided the layout (SKILL.md section 7.6).
  - **Icons**: the generated PNG is vectorized with `potrace` into an SVG that supports `currentColor` (so its color can be controlled via CSS). The SVG is the primary deliverable — no PNG is shipped
  - **Composited banners and data charts**: the SVG is emitted **by default alongside the PNG**, with the background as a raster image layer, the logo as a vector, and the headline/subcopy/CTA as editable text layers — editable in Figma as "background = image layer / logo = vector / text = text layers"
  - **Infographics and diagrams**: on request only (they are built in HTML, so an SVG version means re-laying it out by hand coordinates)
- **Never wrap an AI-drawn image (photographic visuals, UI mockups) in an SVG.** There is no way to vectorize it, so the result is a box holding a single
  `<image>` tag. Measured: a 1.03 MB PNG became 1.37 MB (**1.33x**) with exactly **one** editable element.
- Transparent backgrounds are not supported (the default model, `gpt-image-2`, cannot output transparency). If needed, the background must be removed afterward.

### No agent or skill dependencies (self-contained)

This skill **works on its own**. No additional agents need to be installed.

An earlier version delegated the pre-delivery review step to internal marketing agents
(`mkt-visual-creative`, `mkt-paid`, `mkt-social`, `mkt-cmo`). On closer inspection, only a small
fraction of those agents was actually applicable to reviewing a single generated image — the bulk of
them covers ad-account auditing, media planning, and posting calendars, none of which this skill touches,
and they in turn depend on other internal skill suites. So **the applicable criteria were written directly
into SKILL.md section 7.5 instead, removing the agent dependency**.

| Original agent | What was carried over |
|---|---|
| `mkt-visual-creative` | Pre-generation checks (brand guidelines, prohibited expressions, colors to avoid, existing assets), plus rights/provenance recording |
| `mkt-paid` | Regulatory angle (Japanese pharmaceutical-advertising and fair-labeling law) |
| `mkt-social` | Per-platform etiquette (don't reuse one image everywhere), influencer-disclosure rules |
| `mkt-cmo` | Nothing — it's purely a dispatcher and holds no review knowledge |

If those marketing agents happen to be installed in your environment, you can still hand them the same
material for a second opinion — but it's optional, and the skill is complete without them.

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
