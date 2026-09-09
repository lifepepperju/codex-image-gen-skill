# codex-image-gen

*[日本語](#日本語) / [English](#english)*

---

## 日本語

Claude Code 用スキル。Codex CLI の組み込み画像生成ツール（`image_gen`）を呼び出し、
ChatGPT サブスクリプション内（追加課金ゼロ・APIキー不要）で画像を生成する。

「画像生成して」「Codexで画像」「バナー作って」「アイコン生成」「サムネ作って」等のキーワードで起動する。

### できること

- バナー・LPヒーロー画像などの写真/イラスト調の背景ビジュアル生成
- **既存画像の編集**(色・1要素・背景だけを変える。作り直しより崩れにくい)
- 人物・既存素材・スタイルを**参照画像として渡す**生成
- アイコンの AI 生成 → ベクター化(既製アイコンセットとの使い分けも解説)
- ロゴ・文言は AI に描かせず、生成した背景に HTML+Chrome で後から合成
- **納品前レビュー**: 専任サブエージェントが 200% 拡大でゲート判定 → 依頼側が最終判定、を上限3回まで回す

含まないもの: インフォグラフィック・図解・グラフ(別途 HTML+Chrome / `dataviz` スキル側で対応)。

> **2026-09-09 更新**: OpenAI が Codex にも配布した「ChatGPT Images 2.5」世代(内部モデルは `gpt-image-2.5-flare` /
> `gpt-image-2.5-sunburst` のいずれか)を前提に改訂。**Codex 経由ではどちらのモデルが使われるかを選べない**(公式にも非公開)ため、
> スキルは「この依頼は本来どちらに向くか」を判定してユーザーに伝えるだけで、モデル自体の切り替えはしない。

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
| 人物・既存素材・スタイルに似せたい | `image_gen` に参照画像を渡す(`referenced_image_paths`)。渡しても似ない場合のみ、写真の特徴を英語で言語化してプロンプトに焼き込む旧方式を使う |
| 前回の画像のここだけ直したい | 作り直さず**編集**する(同じ参照画像の仕組みで「変える点」だけを指示) |
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
- 透過背景は非対応(Codexの`image_gen`にサイズ・品質・透過を指定する引数が無い)。必要な場合は生成後に背景を抜く。

### 納品前レビューは専任サブエージェントが行う(2026-09-09〜。以前は自己完結だった)

**この設計は一度、逆方向で試して戻した経緯がある。** 当初はマーケエージェント(`mkt-visual-creative`等)を呼ぶ設計 →
「画像1枚のレビューに使えるのはごく一部」と判断してエージェント依存を無くし SKILL.md 単体で完結する形にした →
しかし**プロンプトを書いた本人がレビューすると、自分の狙いに引っ張られて「そう見えるはず」で通してしまう**ことが
実測で分かったため、**レビュー専任のサブエージェントを新設し、書いた側と見る側を分離した**。

- `agents/image-reviewer-opus.md` / `agents/image-reviewer-fable.md` の2体。**どちらも同じ手順書**(画像を1回全体で見る→
  破綻が出やすい部位を相対座標で選ぶ→200%拡大→ゲート＋点数＋次の1点変更を定型で返す)で、モデルだけが違う
- **既定は Opus。** 反射・モアレ・光の整合など「見えているものの解釈」が成果を左右する案件(前述の Sunburst 向き判定)は
  Fable に切り替える。実測(同一画像を4体で比較)は SKILL.md セクション7-0 参照
- **画像を直すのはレビュー担当ではなく呼び出し側。** レビュー担当はファイルを作らず、判定材料だけを返す
- **最終判定もレビュー担当ではなく呼び出し側。** レビューは上限3回までで、3回目で「合格」か「質は担保できないが打ち切り」を必ず言い切る

導入すると、内蔵マーケエージェントの一部観点(法規制・媒体作法など)は今も SKILL.md 側に残っている。両者は役割が違う:
レビューエージェント=AIの破綻を見つける、SKILL.md 7.5=法規制・媒体適合を見る(勝手に判断せず止めるべき所を明示)。

### インストール

このリポジトリの `codex-image-gen/` フォルダと **`agents/` フォルダの2体**を、Claude Code のディレクトリにコピーする。

```bash
# プロジェクト単位で使う場合
cp -r codex-image-gen /path/to/project/.claude/skills/
cp -r agents/image-reviewer-*.md /path/to/project/.claude/agents/

# 全プロジェクト共通で使う場合
cp -r codex-image-gen ~/.claude/skills/
cp agents/image-reviewer-*.md ~/.claude/agents/
```

`agents/` を入れずにスキルだけ導入した場合、SKILL.md のレビュー手順(セクション7)がエージェントを見つけられず動かない。
その場合は `general-purpose` に `model` だけ指定して同じ手順書を渡す代替手段が SKILL.md のトラブルシューティングにある
(エフォートは指定できない)。

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
- **Edits an existing image** (change only a color, one element, or the background — more stable than regenerating)
- Generates **using reference images** (a person, an existing asset, a style to match)
- AI-generates icons and vectorizes them (also covers when to use a ready-made icon set instead)
- Never lets the AI draw logos or copy — those are composited afterward onto the generated background via HTML+Chrome
- **Pre-delivery review**: a dedicated subagent renders 200% zoomed crops and applies pass/fail gates; the caller makes the final call, for up to 3 review rounds

Not covered: infographics, diagrams, or charts (handled separately via HTML+Chrome / the `dataviz` skill).

> **Updated 2026-09-09** for the "ChatGPT Images 2.5" generation OpenAI rolled out to Codex (backed by either
> `gpt-image-2.5-flare` or `gpt-image-2.5-sunburst` — which one is used per call is not exposed, even officially).
> Since Codex can't select between them, the skill only judges which one a given request would actually suit and
> tells the user — it never tries to force the model choice itself.

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
| Matching a real person / an existing asset / a style | Pass it to `image_gen` as a reference image (`referenced_image_paths`). Only if that doesn't produce a good enough match, fall back to describing the photo's features in English and baking that description into the prompt |
| "Just fix this one thing from last time" | **Edit**, don't regenerate — same reference-image mechanism, instructing only the one change |
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
- Transparent backgrounds are not supported (Codex's `image_gen` exposes no parameter for size, quality, or transparency). Remove the background afterward if you need one.

### Pre-delivery review is now a dedicated subagent (since 2026-09-09 — this skill used to be self-contained)

**This design was tried in the opposite direction once already, and reverted back.** It originally called
marketing agents (`mkt-visual-creative`, etc.) for review → that was judged overkill for reviewing a single
image, so the skill was made agent-free and self-contained → but testing then showed that **whoever wrote the
prompt tends to review it through the same lens they wrote it with**, waving through results because "that's
what I meant it to look like." So **a dedicated review subagent was introduced to separate the person who
wrote the prompt from the person who judges the result.**

- Two agents, `agents/image-reviewer-opus.md` and `agents/image-reviewer-fable.md`, **share the exact same
  instructions** (look at the whole image once, pick likely failure spots by relative coordinates, zoom to
  200%, return gates + scores + one suggested next change in a fixed format) — only the underlying model differs
- **Opus is the default.** Switch to Fable for cases where getting the *interpretation* right — reflections,
  moiré vs. real artifacts, lighting consistency — decides the outcome (the same cases judged "Sunburst-leaning"
  above). See SKILL.md section 7-0 for the side-by-side test that led to this default
- **The reviewer never edits the image** — only the caller does that. The reviewer doesn't create files either,
  it only returns judgment material
- **The reviewer doesn't make the final call either** — the caller does. Reviews are capped at 3 rounds; by the
  3rd, the caller must state either "passes" or "quality wasn't fully verified but stopping here"

Some of the criteria originally pulled from the marketing agents (regulatory checks, per-platform etiquette) still
live in SKILL.md section 7.5 — that's a different job from the reviewer subagent: the reviewer catches AI-specific
artifacts, section 7.5 covers regulatory/platform fit (and tells you when to stop and ask, rather than deciding alone).

### Install

Copy this repo's `codex-image-gen/` folder **and the two files in `agents/`** into your Claude Code directories.

```bash
# Per-project
cp -r codex-image-gen /path/to/project/.claude/skills/
cp agents/image-reviewer-*.md /path/to/project/.claude/agents/

# Global (all projects)
cp -r codex-image-gen ~/.claude/skills/
cp agents/image-reviewer-*.md ~/.claude/agents/
```

If you skip `agents/` and install only the skill, the review step in SKILL.md (section 7) won't find an agent to
call. SKILL.md's troubleshooting table has a fallback (point `general-purpose` at the same instructions with just
`model` set — effort level can't be set that way, though).

### Prerequisites

- Codex CLI installed and logged in (`codex login`)
- `auth_mode` in `~/.codex/auth.json` must be `chatgpt` (if it's `api_key`, API usage will be billed)

Full instructions and caveats (Japanese only) are in [`codex-image-gen/SKILL.md`](./codex-image-gen/SKILL.md).
