---
name: codex-image-gen
description: Codex CLIの組み込みimage_genツールで画像生成する。ChatGPTサブスク内で追加課金ゼロ・APIキー不要。「画像生成して」「Codexで画像」「バナー作って」「アイコン生成」「サムネ作って」等のキーワードで起動。
---

# Codex Image Gen — ChatGPTサブスクで画像生成

Codex CLI の組み込み画像生成ツール（`image_gen`）を呼び出し、追加課金なしで画像を作る。

**最終検証: 2026-07-29 / codex-cli 0.145.0**（コマンドは実機で通したもの）
**プロンプト方針の出典**: OpenAI 公式 [GPT Image Models Prompting Guide](https://developers.openai.com/cookbook/examples/multimodal/image-gen-models-prompting-guide) / [Image generation API guide](https://developers.openai.com/api/docs/guides/image-generation)

> **プレースホルダの規約**: `{…}` と `{{…}}` はどちらも**実行前に実値へ置換する**。1つでも残っていたら実行しない
> （`{{OUTPUT_PATH}}` という名前のファイルが作られる）。
> `$WORK` = ログ・PID置き場（スクラッチパッド配下）／`$GEN` = Codex の作業ディレクトリ／`$OUT` = 納品先（`Output/…`）。
>
> **このファイルは2箇所に同一内容で置かれている**（プロジェクト `.claude/skills/` とグローバル `~/.claude/skills/`）。
> 正本はプロジェクト側。修正したら必ず両方に反映する。

---

## 最初に判断: 作りたいものによって手段が変わる

**`image_gen`（AI生成）は写真・イラストの「絵」を作る道具。文字・図・アイコンには使わない。**
ここを間違えると、いくらプロンプトを工夫しても質が上がらない。

| 作りたいもの | 手段 | SVG併記 | 参照 |
|---|---|---|---|
| 写真風・イラストの**背景ビジュアル**（バナー、LPのヒーロー、SNS投稿の絵） | `image_gen`（AI生成） | **絵単体では出さない**（合成すれば下段の扱いになる） | セクション0〜6 |
| **ロゴ・見出し・コピー** | 実ファイル＋実フォントで**後から合成**（HTML＋Chrome） | **既定で出す** | セクション7・**7.6** |
| **インフォグラフィック・図解・比較表・手順図** | **HTML＋Chrome のみ。AI生成は使わない** | 要求時のみ（HTMLの組み直しになる） | **セクション10**・7.6 |
| **アイコン** | AI生成→ベクター化（質が高い）／既製セット（速い）。**AIのラスターをそのまま貼らない** | **SVGが主**（PNGは出さない） | **セクション10** |
| データのグラフ・チャート | HTML/SVG＋`dataviz` スキル＋**実データ**（数字は作らない） | **既定で出す** | セクション10・7.6 |
| 実際の地図・地形 | 地理データSVG（パブリックドメイン等）を調達。アイコンセットには無い | 元がSVG | — |
| 人物の顔を似せる | `image_gen` は参照画像を取れない。ビジョン言語化リレー | 絵単体なので出さない | セクション9 |
| **UI・画面モック**（雰囲気の確認だけ） | `image_gen`（例外扱い）。**画面構成の詰めと実装への受け渡しは Claude Design が本筋** | **出さない**（7.6の実測） | **セクション11** |

---

## 0. 最重要ルール: プロンプトは短く保つ

**盛れば盛るほど狙いから外れる。** 公式ガイドも「長ければ良いという前提を置くな。きれいな土台から始めて、
小さな1点変更で詰めていく方がデバッグしやすい」としている。ユーザーの経験とも一致する。

### 守ること（この5つだけ）

1. **英語で 1〜3文**。4文を超えたら盛りすぎ。切る
2. **順番は固定**: 用途 → 主題（何が写るか）→ 必要な細部 → **除外条件は必ず最後**
   - 除外（`No text` 等）を前に置くと、モデルが「構図の指示」と誤解して逆に出てくる
3. **用途を1語目に書く**（`Ad banner for...` / `App icon for...` / `Hero image for...`）。
   これだけで仕上がりの水準が合う
4. **1回で完成させようとしない**。まず土台を1枚出し、**1回に1点だけ**変えて作り直す
5. **飾り言葉を足さない**。`4k, ultra detailed, professional, trending on dribbble, masterpiece, cinematic`
   のような呪文は効果が薄く、主題をぼかす。使わない

### 使う穴埋めテンプレ（これ以上は書かない）

```
{用途} for {対象}. {主題を1文で: 何が、どこに、どんな色/形で}. {スタイルを2〜3語}.
No {除外したいもの}.
```

### 良い例 / 悪い例

**悪い（盛りすぎ・実際に外れる書き方）**
```
A modern minimalist app icon for a car maintenance tracker app, flat design,
blue and white color scheme, clean geometric shapes, transparent-like white background,
subtle gradient, soft shadows, rounded corners, iOS style, high detail, 4k, professional,
centered composition, no text, no watermark, trending on dribbble
```

**良い（同じ狙いを3要素に絞る）**
```
App icon for a car maintenance tracker.
A flat geometric wrench and speedometer, centered on a white background, blue and white palette.
No text.
```

**良い（広告バナー）**
```
Ad banner for a fuel cost tracking app.
A smartphone showing a simple dashboard with a line chart, on a soft blue gradient background.
Flat, clean, plenty of empty space on the left for copy.
No text, no logos.
```

> 文字（コピー・社名・数字）は**画像に描かせない**。短い一般的な日本語なら描けるが、
> 固有名詞・数字は誤字る。文字は HTML / 資料側のテキストで後乗せするのが確実。
> だからテンプレの除外は `No text.` を既定にし、**文字を置く余白だけ空けさせる**。
> **例外は UI・画面モックだけ**（ラベルが無いと画面として成立しない）→ セクション11。

### 複雑な依頼が来たときの分解方針

1枚に複数の要素を詰め込ませない。**要素ごとに分けて生成し、レイアウトは後段（HTML / 資料 / 画像編集）で合成する。**

| ユーザーの依頼 | やること |
|---|---|
| 「キャッチコピー入りのバナー」 | 背景ビジュアルだけ生成 → 文字は HTML / スライド側で載せる |
| 「3つのサービスを並べた図」 | アイコンを3枚別々に生成 → 並べるのは資料側 |
| 「人物＋商品＋店内＋ロゴ入り」 | 主役を1つに絞る。残りは別カットか、後段合成 |
| 「前回の画像のここだけ直して」 | 「編集」はできない（後述）→ **1点だけ変えたプロンプトで作り直す** |

---

## 1. 前提チェック

```bash
grep -o '"auth_mode"[^,}]*' ~/.codex/auth.json
```

- `"chatgpt"` → そのまま続行（ChatGPTサブスク内・追加課金なし）
- `"api_key"` → API従量課金が発生する。**ユーザーに確認を取ってから**続行
- ファイルなし → `codex login` を案内して中断

機能フラグは 0.145.0 では既定ONなので通常は何も足さなくてよい（確認したい場合のみ）:

```bash
codex features list | grep -E "image_generation|tool_suggest"   # どちらも stable / true が既定
```

---

## 2. 実行コマンド（既定テンプレート）

**作業ファイルは納品フォルダに置かない。** `Output/` は git 管理下なので、実行ログを置くと git を汚す。
Codex の作業場所（`-C`）と、ログ・PIDの置き場所（`$WORK`）と、最終納品先（`$OUT`）を分ける。

```bash
WORK="{スクラッチパッドの絶対パス}/codex-image-gen"   # ログ・PID・中間ファイル
GEN="$WORK/gen"                                       # Codex が書き込む作業ディレクトリ
OUT="{納品先ディレクトリの絶対パス}"                   # 完成物だけを置く（Output/[カテゴリ]/[企業名]/）
mkdir -p "$WORK" "$GEN" "$OUT"

codex exec \
  --sandbox workspace-write \
  -c sandbox_workspace_write.network_access=true \
  -c model_reasoning_effort="low" \
  --skip-git-repo-check \
  -C "$GEN" \
  --json \
  "Use the built-in image_gen tool RIGHT NOW to generate exactly one image.
Do not read any files. Do not write any script. Do not use any API key.

Prompt: {{短い英語プロンプト（1〜3文・セクション0のテンプレ）}}

After image_gen returns, copy the generated PNG to ./gen.png in the current working directory using cp.
Then report the absolute path image_gen originally returned." \
  </dev/null > "$WORK/events.jsonl" 2> "$WORK/stderr.log" &
echo $! > "$WORK/codex.pid"
wait $!
echo "EXIT=$?"
```

> **Codex には `-C` 配下の相対パス（`./gen.png`）へコピーさせる。**
> `--sandbox workspace-write` の書き込み許可は「`-C` の作業ディレクトリ＋`/tmp`＋`$TMPDIR`」に限られるため、
> `Output/` などの外部パスを直接指示すると sandbox に弾かれてコピーされない（「やってくれない場合がある」の主因）。
> 納品先への配置は**回収時に Claude 側の `cp` で行う**（セクション4）。
>
> 上の「RIGHT NOW / Do not read any files」等は **Codex への運転指示**であって画像プロンプトではない。
> ここは固定文なので毎回同じにする。**短く保つべきなのは `Prompt:` の中身だけ**。
> `{{…}}` が1つでも残っている状態で実行しないこと（`{{OUTPUT_PATH}}` という名前のファイルが作られる）。

### フラグの役割と、間違えやすい点

| 項目 | 内容 |
|---|---|
| `--sandbox workspace-write` ＋ `network_access=true` | **これで画像生成は通る（実測）**。`--dangerously-bypass-approvals-and-sandbox` は不要。安全側のこちらを既定にする |
| `</dev/null`（末尾） | **必須**。`codex exec` は位置引数でプロンプトを渡しても stdin を読みに行き、閉じていないと無言で固まる（実測） |
| `--skip-git-repo-check` | git 外・trusted 未登録のディレクトリでも即終了しないようにする |
| `-C "{絶対パス}"` | 作業ディレクトリを明示。`$PWD` をクォート内で展開させると意図しない場所で動くことがある |
| `-c model_reasoning_effort="low"` | 画像生成に高い推論量は不要。既定（`max`）のままだと遅い |
| プロンプトは英語 | 精度が高い。日本語＋曖昧な指示だと生成せずに終わることがある |

---

## 3. 1点ずつ直す（反復ワークフロー）

**Codex 経由では「さっきの画像を編集」はできない**（`image_gen` は参照画像を受け取れない＝後述）。
反復とは「**プロンプトを1点だけ変えて作り直す**」こと。

```
1枚目: テンプレ通りの土台（主題＋用途＋スタイル2〜3語）
  ↓ ユーザーに見せて、直したい点を1つ聞く
2枚目: その1点だけ変える（例: 背景を白→薄いグレー / 構図を寄りに / 色を暖色に）
  ↓
3枚目: さらに1点だけ
```

- **一度に2点以上変えない**。どの変更が効いたか分からなくなる
- 変更は既存の文を差し替える形にする（**文を足していくと元の狙いが薄まる**）
- 同じ狙いを保ちたい要素は毎回そのまま書き直す（省略するとドリフトする）
- `codex exec ... resume --last` で同スレッド継続もできるが、画像は毎回新規生成になる。
  プロンプトを短く保つ方が結果は安定する

---

## 4. 生成画像の回収（ここが最大のハマりどころ）

`image_gen` の原本は **`~/.codex/generated_images/<session id>/call_*.png`** に出る。
プロンプトで `cp` を指示すれば Codex がコピーしてくれるが、**やってくれない場合がある**ので必ず確認する。

フォルダ名は **`thread_id`**（＝セッションID）。**`ls -t` で最新フォルダを拾う方法は使わない**（並行生成で取り違える）。

**取得元は `--json` を付けたかどうかで変わる。**（実測で確認。間違えると空振りする）

| 実行方法 | ID の取り方 |
|---|---|
| **`--json` あり（このスキルの既定）** | `events.jsonl` の `thread.started` の `thread_id`。**stderr には出ない** |
| `--json` なし | stderr の `session id: <uuid>` 行 |

```bash
# 既定（--json あり）の場合。パスは argv で渡す（パスに ' が入るとコード内埋め込みは壊れる）
SID=$(python3 -c 'import json,sys
for l in open(sys.argv[1]):
    try: e=json.loads(l)
    except Exception: continue
    if e.get("type")=="thread.started": print(e.get("thread_id","")); break' "$WORK/events.jsonl")
[ -n "$SID" ] || { echo "thread_id が取れない。$WORK/events.jsonl の先頭行を確認する"; exit 1; }

DIR="$HOME/.codex/generated_images/$SID"
[ -d "$DIR" ] || { echo "生成画像フォルダが無い: $DIR"; exit 1; }
ls -la "$DIR"

# 同一セッション内に複数PNGがある場合があるので、最新1枚だけを取る
SRC=$(ls -t "$DIR"/*.png 2>/dev/null | head -1)
[ -n "$SRC" ] || { echo "PNGが無い: $DIR"; exit 1; }
cp "$SRC" "{{OUTPUT_PATH}}"
# 複数枚まとめて欲しい場合は宛先を「既存のディレクトリ」にする: cp "$DIR"/*.png "{{OUTPUT_DIR}}/"
```

**注意点（すべて実測で壊れることを確認済み）**

- **`SID` が空でも次の行に進まないようガードを入れる**。空だと `generated_images//*.png` になり、zsh は `no matches found` で止まる
- **`cp A B C ... 宛先ファイル` は失敗する**（`cp: …: Not a directory`）。宛先が単一ファイルなら入力も1つに絞る
- **`ls -t` を禁止しているのは「フォルダの選択」の話**。同一 `thread_id` フォルダ内で最新の1枚を選ぶ用途には使ってよい
- パスに `'`（アポストロフィ）が入ると、Pythonコードに文字列として埋め込む書き方は SyntaxError になる → **必ず `sys.argv` で渡す**

---

## 5. サイズ（Codex 経由では指定できない）

実測: プロンプトに `Size: 1024x1024` と書いても **1254×1254** が生成された。
`image_gen` は prompt しか受け取らないため、**サイズ・品質・透過は指定できない**。
プロンプトにサイズを書くのは無意味なので**書かない**。**寸法は生成後に必ず自分で合わせる。**

ただし**縦横比は文章で効く**（実測）: `tall 9:16 composition` と書いた縦型バナーは **941×1672（比率0.56 ≒ 9:16）**
で出た。**ピクセル数は指定できないが、比率は「vertical 9:16 composition」「square composition」等の
構図の言葉で寄せられる**。狙いの比率を先に文章で作り、最後に `sips` で目的サイズへ拡大するのが確実。

```bash
# 実寸を確認
python3 -c "
import struct;d=open('{{OUTPUT_PATH}}','rb').read(33)
w,h=struct.unpack('>II',d[16:24]);print(f'{w}x{h}')"

# 目的サイズへ（縦横比が違う場合は切り抜きを検討。引き伸ばすと不自然になる）
# 原本を保全してから変換する（同じパスに --out すると原本が消え、比率違いに気づいても戻せない）
cp "{{OUTPUT_PATH}}" "{{OUTPUT_PATH}}.orig.png"
sips -z {{HEIGHT}} {{WIDTH}} "{{OUTPUT_PATH}}.orig.png" --out "{{OUTPUT_PATH}}"
```

### よく使うサイズ

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

### 参考: API を直接叩く場合の仕様（公式ドキュメント）

Codex 経由では使えないが、将来 API 直叩きに切り替える場合の前提として:

- 現行の既定モデルは **`gpt-image-2`**（他に `gpt-image-1.5` / `gpt-image-1` / `gpt-image-1-mini`）
- サイズ: `1024x1024` `2048x2048` `1536x1024` `2048x1152` `3840x2160` `1024x1536` `2160x3840` `auto`
- 制約: 最大辺 3840px 以下 / 両辺が16の倍数 / アスペクト比 3:1 以内
- 品質: `low` / `medium` / `high` / `auto`
- **`gpt-image-2` は透過背景に非対応**（透過が必要なら別モデル、または生成後に背景を抜く）

---

## 6. 納品前チェック（AI生成の破綻を潰す・省略しない）

**縮小表示では気づかない破綻が必ず残っている。** 実例（2026-07-29 某クライアント案件のバナー）: 等倍では自然に見えたが、
手を2倍に拡大したら**指が溶けて本数も判別できない状態**だった。カメラを持つ手も指の付き方が解剖学的に破綻していた。
**拡大確認を飛ばして納品しない。**

判定方式は OpenAI 公式の画像評価ガイド（[Image Evals](https://developers.openai.com/cookbook/examples/multimodal/image_evals)）に合わせる。

### 6-1. 判定は「ゲート」＋「点数」。平均で救わない

**ゲート（1つでも落ちたら不合格 → 作り直し）**

| ゲート | 合格条件 |
|---|---|
| 指示追従 | 依頼した主題・構図・スタイル・比率になっている |
| 画像内の文字 | **文字が写り込んでいない**（`No text` 指定のため）。写っていたら不合格。文字を入れる場合は**綴り・記号・大文字小文字まで完全一致**が条件。**UI・画面モックのみ例外**（セクション11のゲートで判定する） |
| 致命的破綻ゼロ | 6-2 の重点チェックで、人物の解剖学・反射・接点に明確な破綻がない |

**点数（0〜5 / 3以上で合格）**

| 項目 | 見るところ |
|---|---|
| レイアウト・余白 | ロゴとコピーを置ける余白が意図した位置にあるか |
| ブランド適合 | **既存クリエイティブと並べて**違和感がないか |
| 視覚品質 | 光・色・被写界深度・質感が破綻していないか |

**判定ルール（重要）**: どれか1つでも閾値割れなら**総合は不合格**。
高い点で平均して合格にしない（例: 視覚品質5でも文字ゲートが落ちたら不合格）。

> 我々のパイプラインは**文字をChromeで後乗せする**（セクション7）ため、文字ゲートは構造的に満たしやすい。
> AIに文字を描かせる運用は、この一点だけで不合格リスクが跳ね上がる。

### 6-2. 見る順番（2026年時点の優先度）

生成モデルの改善で「手を見れば分かる」時代は終わりつつある。**現在いちばん破綻が残るのは文字と反射**。
ただし今回のバナーでは実際に手が壊れたので、手も引き続き見る。

| 優先 | 見るもの | 具体的に何を疑うか |
|---|---|---|
| 1 | **画像内の文字** | 看板・のれん・値札・服のロゴ。意味不明な字形が最も残りやすい |
| 2 | **反射・映り込み** | ガラス・水面・車体・眼鏡。**映っているものが現場と一致しているか**（現行モデルが最も苦手） |
| 3 | **ディテール密度の不均一** | 一部だけ異様にシャープ／のっぺり。境目が不自然 |
| 4 | **解剖学の複合破綻** | 手・指（本数/融合/関節なし）、腕の本数、肩と腕の接続、脚と靴の左右、足首のねじれ。**単独より複数箇所の違和感が重なる方が危険信号** |
| 5 | 手と物の接点 | カメラ・カバン・傘の持ち方、ストラップが途切れる／体を貫通する |
| 6 | 反復パターン | 群衆・木・提灯・石畳の不自然なコピー |
| 7 | 背景の意味的不整合 | あり得ない構造・接続、消失点の破綻、遠景の人物の顔崩れ |

### 6-3. 2段階で見る（片方だけでは不十分）

**① 実際に表示されるサイズで全体を見る**
配信面のサイズ（Storyならスマホ全画面相当）で見て、目線の流れ・可読性・第一印象を判断する。
拡大だけ見て「粗が無いからOK」と判断しない。**見られるサイズで良く見えるかが本題**。

**② 200%に拡大して部位ごとに見る**（コードで切り出して Read で目視）

**座標は絶対値でハードコードしない。** 画像サイズは毎回違う（Codex経由ではサイズを指定できないため）。
PIL の `crop` は**範囲外を黒で埋めて例外を出さない**ので、座標がずれると
「大部分が黒い画像」を見て「問題なし」と判定してしまう（実測: 941×1672 の画像に 1080×1920 前提の座標を当てると 68% が黒）。

**必ず 0〜1 の相対座標で指定し、範囲外を検出させる。**

```bash
python3 - "{{PATH}}" "{{TMP}}" <<'PY'
import sys
from PIL import Image
src, tmp = sys.argv[1], sys.argv[2]
im = Image.open(src); W, H = im.size
print(f"画像サイズ: {W}x{H}")

def qa(name, box, zoom=2):
    l, t, r, b = (int(v*s) for v, s in zip(box, (W, H, W, H)))
    assert 0 <= l < r <= W and 0 <= t < b <= H, f"{name}: 範囲外 {(l,t,r,b)} / 画像 {(W,H)}"
    im.crop((l,t,r,b)).resize(((r-l)*zoom, (b-t)*zoom), Image.LANCZOS).save(f"{tmp}/qa-{name}.png")
    print(f"  {name}: {(l,t,r,b)} → {tmp}/qa-{name}.png")

# 相対座標は「一度画像全体を見てから」決める。前回の数値を流用しない
qa("hands", (0.33, 0.61, 0.70, 0.76))
qa("legs",  (0.17, 0.81, 0.91, 0.99))
PY
```

- **切り出し前に必ず画像全体を1度 Read して被写体位置を確認する**。座標は毎回そこから決める
- 切り出し後は「黒が大半でないか」も併せて確認する（`assert` を抜けても構図がずれている可能性がある）

破綻が見つかったら、**その部位を写さない構図に1点だけ変えて再生成**する（ピクセル修正はできない）。

### 6-4. 記録を残す（後で必ず必要になる）

採用した画像ごとに、以下をユーザーへの報告に含める（成果物と一緒に残す）:

- 使ったプロンプト（全文）
- 生成が **AI** であること、どのツールか
- チェック結果（合格したゲート／点数／弱い箇所）
- 何周目の出力を採用したか

理由: 広告の**AI開示**（6-6）や、ブランド側の承認・監査で「これはAI生成か」を必ず聞かれる。
後から思い出せないと答えられない。

### 6-5. AI判定ツールには頼らない

Hive / Illuminarty のような「AI生成判定」サービスや、C2PA等のメタデータ判定は**当てにしない**。

- SNS（X・Instagram等）の圧縮で、検出器が見ている低次のアーティファクトが壊れる
- Chrome合成やリサイズを通すと、由来メタデータは基本的に残らない
- 判定は**自分の目 + 6-2 のチェックリスト**が主。ツールは補助にもならない前提で組む

### 6-6. 「AI生成」表記を入れるか（開示）

**「AI表記を入れるとパフォーマンスが上がる」は一般則として成立しない。** 研究は真っ二つに割れている。

| 出典 | 結果 |
|---|---|
| MediaScience（2026-05・動画広告実験） | ラベル表示で**どの指標も低下しなかった**。AI生成の認知は上昇（冒頭3秒表示で+28%、常時表示で+36%） |
| Shi & Jiang（2026・SAGE Open） | **両刃**。新規性↑を通じて広告態度・購買意向を**押し上げ**、同時に真正性↓を通じて**押し下げる** |
| 学術研究の多数 | 開示は説得知識を発動させ、信頼・購買意向を**下げる**。**高関与商材で顕著** |
| 別の実験 | ラベルでCTRが約1.17pt（相対約31%）**低下** |
| IAB（2026-01） | 業界は若年層のAI広告受容を過大評価（幹部82%が好意的と予想 / 実際45%） |

**判断の指針**
- 「上がる」説は**新規性効果**を捉えたもの。逆に**真正性が価値の中心の商材（人・信頼・高額）では下がりやすい**
- 隠すリスクも別にある（後で気づかれると不公正と受け取られる）
- 結論: **自社でA/Bテストして決める**（ラベル有無の2本で配信し、CTRとCVRの両方で見る）。他社の結論を輸入しない

**プラットフォーム・法規（2026-07-29 時点でMeta公式ページで確認した内容）**
- Meta は**自社の生成AI機能**で作成・大幅編集した広告に「AI情報」ラベルを**自動で付ける**。
  リアルな人物が含まれる場合は、より目立つ位置（広告の上）に表示される
- **サードパーティのAIツールで作った広告には Meta のラベルは付かない**
- リサイズ・色補正などの軽微な編集はラベル対象外
- **社会問題・選挙・政治**の広告は、サードパーティAIの使用を含めて**開示が義務**
- 一般商材の広告について「全広告で開示が義務」とする記述は**公式ページでは確認できなかった**（一部メディアはそう報じている）
- **未確認**: EU等の地域別ルール。EU向けに配信する場合はクライアントの法務確認が必要

### 直すより避けるほうが安い（プロンプト側の予防）

- **手を写さない／目立たせない構図にする**（後ろ姿・ポケットに手・遠景・腰から下を切る）
- **物を持たせない**（カメラ・スマホ・傘を持つ手は最も崩れる）
- 人物を小さく配置する（大写しほど崩れが目立つ）
- 顔のクローズアップを避ける
- 群衆を入れない（遠景の顔が崩れる）

**実証済みの言い換え（2026-07-29 某クライアント案件のバナー）**

| 崩れた書き方 | 直った書き方 |
|---|---|
| `... seen from behind, warm afternoon light.`（身振りする手が生成され指が溶けた） | `... all seen from behind with their hands down at their sides, warm afternoon light.` |

`hands down at their sides` を入れるだけで、手が下がって小さく写り、指の融合・本数エラーが消えた。
**変更したのはこの1点だけ**（他の文はそのまま）。副産物として人物が下に寄り、上部のコピー余白が広がった。

> このチェックは**目視が最終判断**。ユーザーに渡すときは「拡大して確認済み／ここが弱い」まで伝える。
> 見つけた破綻を黙って納品しない。

---

## 7. ロゴ・文言を載せる（AIに描かせず、後から重ねる）

**ロゴと文字は絶対に `image_gen` に描かせない。** ロゴは再現されず、社名・数字は誤字る。
**背景ビジュアルだけAIに作らせ、ロゴの実ファイルと本物のフォントを後から重ねる。**

### 手順

1. **背景だけ生成**（セクション0のテンプレ）。`No text, no logos.` を入れ、**コピーを置く余白を空けさせる**
   （例: `plenty of empty space at the top for copy`）
2. **ブランド色をロゴの実ファイルから抽出する**（推測しない）
   ```bash
   grep -o 'fill="[^"]*"' logo.svg | sort -u        # 例: rgb(204,40,46) → CTAの赤に使う
   # 縮尺計算用（SVGは viewBox が内部座標系。width属性だけ見ると桁がずれる）
   python3 -c "
import re,sys
s=open(sys.argv[1]).read()
m=re.search(r'viewBox=\"([\d.\-\s]+)\"', s)
if m: vb=m.group(1).split(); print('viewBox幅:', vb[2], '高さ:', vb[3])
w=re.search(r'\bwidth=\"([\d.]+)', s); h=re.search(r'\bheight=\"([\d.]+)', s)
print('width属性:', w.group(1) if w else '無し', '/ height属性:', h.group(1) if h else '無し')" logo.svg
   ```
   PNGロゴの場合は透過と実寸を確認する:
   ```bash
   python3 -c "
from PIL import Image; import sys
im=Image.open(sys.argv[1]); print('サイズ:', im.size, 'モード:', im.mode)
print('透過:', 'あり' if im.mode in ('RGBA','LA') or 'transparency' in im.info else 'なし（白背景なら抜く必要あり）')" logo.png
   ```
   **PNGロゴは表示幅を元の横幅以下に収める**（拡大すると甘くなる）。透過が無い場合はセクション7末尾の白抜き手順を使う。
3. **HTMLを書いてローカルChromeで指定サイズに書き出す**（下のテンプレ）
4. **セクション6の納品前チェックを実施する**

```html
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  html, body { width: 1080px; height: 1920px; overflow: hidden; }
  .stage { position: relative; width: 1080px; height: 1920px;
           font-family: Inter, "Avenir Next", "Helvetica Neue", sans-serif; }
  .bg { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; }
  .logo { position: absolute; top: 96px; left: 50%; transform: translateX(-50%); width: 460px; }
  .copy { position: absolute; top: 268px; left: 0; right: 0; text-align: center; padding: 0 90px; }
  .headline { font-size: 82px; line-height: 1.14; font-weight: 700; letter-spacing: -0.02em; color: #14263d; }
  .sub { margin-top: 30px; font-size: 40px; line-height: 1.4; font-weight: 500; color: #3d4b5c; }
  .cta { position: absolute; bottom: 132px; left: 50%; transform: translateX(-50%);
         background: rgb(204,40,46); color: #fff; font-size: 44px; font-weight: 700;
         padding: 34px 76px; border-radius: 999px; box-shadow: 0 12px 32px rgba(0,0,0,.22); white-space: nowrap; }
</style>
<div class="stage">
  <img class="bg" src="bg.png">
  <img class="logo" src="logo.svg">
  <div class="copy">
    <div class="headline">{{見出し1行目}}<br>{{2行目}}</div>
    <div class="sub">{{サブコピー}}</div>
  </div>
  <div class="cta">{{CTA}}</div>
</div>
```

```bash
W="{作業ディレクトリの絶対パス}"       # bg.png / logo.svg(or .png) / banner.html を置いた場所
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
rm -f "$W/out.png"
"$CHROME" --headless --disable-gpu --hide-scrollbars --force-device-scale-factor=1 \
  --window-size=1080,1920 --screenshot="$W/out.png" "file://$W/banner.html"

# Chrome は書き込みに失敗しても exit 0 を返す。必ず実体を検証する
python3 -c "
from PIL import Image
im=Image.open('$W/out.png'); assert im.size==(1080,1920), im.size; print('OK', im.size)" \
  || echo "スクリーンショット失敗（\$W のパス・権限・HTMLの読み込みを確認）"
```

- **`$W` を必ず定義する。** 未定義のまま実行すると `--screenshot=/out.png` になり、
  `Failed to write file /out.png: Read-only file system` を出しながら **exit 0（成功扱い）** で終わる（実測）。
  納品物を作る唯一の工程なので、**存在と寸法のアサートを省略しない**
- 背景・ロゴ・HTMLは**同じフォルダに置いて相対パス参照**にする（日本語＋スペース入りパスの `file://` は実測で問題なし）
- **フォントは Inter を第一候補**にする（Figma標準搭載）。ただし **ローカルにも Inter が入っていないと意味がない**:
  入っていなければ Chrome はフォールバックで描くので、**PNGとFigma表示で字幅が変わる**（実測: 同じ文字列で 519px → Inter導入後 470px）。
  確認と導入:
  ```bash
  ls ~/Library/Fonts /Library/Fonts 2>/dev/null | grep -i "^Inter"   # 何も出なければ未導入
  brew install --cask font-inter
  ```
- **文言の差し替えはHTMLの1行を書き換えて再出力するだけ**（画像は作り直さない・数秒）
- ブランドフォントが不明な場合は「Interで仮組みした」と必ず伝える（勝手に確定させない）
- HTMLに `<meta charset="utf-8">` を入れておく（日本語コピーの文字化け保険）

### PNG＋SVG の2形式で出す（合成ものは**既定**。頼まれなくても出す）

同じ構成を **SVG** でも作れば、Figma に読み込んだとき **背景＝画像レイヤー / ロゴ＝ベクター / 見出し・サブ・CTA＝テキストレイヤー**
として編集できる（実測で書き出し確認済み。PNGとほぼ同一の見た目になる）。

**この節の成果物（背景AI＋文字・ロゴを自分で配置したもの）は、SVGも既定で納品する。**
文言差し替え・色調整が先方側でできるようになり、追加コストは数十秒。ただし**何にでもSVGを付けるのではない** — 判定はセクション7.6。

**文言は必ず Python 変数として定義する**（テンプレートに `{{…}}` を残すと、そのまま納品物に出る。実測で発生）。
**XMLエスケープを通す**（コピーに `&` や `<` が入ると SVG がパース不能になり Figma 取り込みも失敗する。実測で再現）。

```python
import base64, html, re, pathlib
import xml.etree.ElementTree as ET

# --- 1) 文言をここで確定させる（プレースホルダを残さない） ---
HEAD1, HEAD2 = "Rent a car,", "see more of Japan"
SUB, CTA_TXT = "Book your rental car online", "Book now"
BRAND = "#FF5C04"                      # ロゴから抽出した実値を使う
FONTS = 'Inter, Avenir Next, Helvetica Neue, sans-serif'   # HTML側と完全に同じスタックにする
e = lambda s: html.escape(s, quote=False)

# --- 2) 背景を埋め込む ---
bg = base64.b64encode(pathlib.Path('bg.png').read_bytes()).decode()

# --- 3) ロゴ。SVGならベクターのまま、PNGなら画像として埋め込む ---
LOGO_W = 460                            # 表示したい幅
logo_src = pathlib.Path('logo.svg')     # PNGの場合は下の else 側を使う
if logo_src.suffix == '.svg':
    logo = logo_src.read_text()
    inner = re.sub(r'</svg>\s*$', '', re.sub(r'^.*?<svg[^>]*>', '', logo, flags=re.S), flags=re.S)
    # 縮尺は「内部座標系の幅」で割る。viewBox があれば必ずそちらを使う（width属性だけ見ると桁がずれる）
    m = re.search(r'viewBox="([\d.\-\s]+)"', logo)
    base_w = float(m.group(1).split()[2]) if m else float(re.search(r'\bwidth="([\d.]+)', logo).group(1))
    logo_el = f'<g id="logo" transform="translate(310,96) scale({LOGO_W/base_w:.5f})">{inner}</g>'
else:
    lb = base64.b64encode(logo_src.read_bytes()).decode()
    from PIL import Image
    lw, lh = Image.open(logo_src).size
    logo_el = (f'<g id="logo"><image x="310" y="96" width="{LOGO_W}" height="{LOGO_W*lh/lw:.1f}" '
               f'xlink:href="data:image/png;base64,{lb}"/></g>')

svg = f'''<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"
  width="1080" height="1920" viewBox="0 0 1080 1920">
  <g id="background"><image x="0" y="0" width="1080" height="1920" preserveAspectRatio="xMidYMid slice" xlink:href="data:image/png;base64,{bg}"/></g>
  {logo_el}
  <g id="headline" font-family="{FONTS}" font-size="82" font-weight="700" fill="#14263d" text-anchor="middle">
    <text x="540" y="330">{e(HEAD1)}</text>
    <text x="540" y="424">{e(HEAD2)}</text>
  </g>
  <text id="sub" x="540" y="510" text-anchor="middle" font-family="{FONTS}" font-size="40" font-weight="500" fill="#3d4b5c">{e(SUB)}</text>
  <g id="cta">
    <rect x="295" y="1676" width="490" height="112" rx="56" fill="{BRAND}"/>
    <text x="540" y="1748" text-anchor="middle" font-family="{FONTS}" font-size="44" font-weight="700" fill="#ffffff">{e(CTA_TXT)}</text>
  </g>
</svg>'''
pathlib.Path('banner.svg').write_text(svg)

# --- 4) 生成直後に必ず検証する ---
assert '{{' not in svg, "プレースホルダが残っている"
ET.parse('banner.svg')                  # XMLとして壊れていないか
print("OK: banner.svg")
```

- **`<g id="...">` に名前を付ける**（Figma側でレイヤー名になり、編集しやすくなる）
- **フォント指定は HTML と1文字も違わないスタックにする。** SVG側だけ `font-family="Inter"` にすると
  解決結果が変わり、**文字幅が8〜14%ずれる**（実測。スタックを揃えたら1〜2px以内に収束した）
- **SVGは折り返しを自動でしない**。改行は行ごとに `<text>` を分けて書く
- **背景はラスター（PNG）のまま**。Figmaでも画像レイヤーとして扱われ、絵の中身は編集できない（差し替え・トリミングは可）
- **ロゴがPNG支給の場合はロゴもラスター**になる（Figmaで色変更・無劣化拡大は不可）。ベクターで渡したいならSVG支給を依頼する
- **PNGはこのSVGからChromeで書き出す**と2形式が一致する（`--screenshot=out.png file://.../banner.svg`）。
  HTML由来PNGとSVGを別々に作った場合は、**画素比較で一致を確認する**こと
- 埋め込みで容量が増える（背景の複雑さ次第で数百KB〜数MB）
- Figma MCP が使える環境なら、SVG受け渡しの代わりに**Figmaへ直接レイヤーを作る**選択肢もある（未検証）

---

## 7.5. マーケレビュー・ゲート（媒体適合はプロンプトに入れず、ここで担保する）

**媒体のベストプラクティスをプロンプトに詰め込まない。** セーフゾーン・テキスト量・交通ルール・コントラスト比などを
プロンプトに書き足すと文が伸び、**主題がぼけて画像の質が落ちる**（このスキルの最重要ルールと衝突する）。
代わりに **合成後にマーケレビューを通し、指摘があれば1点だけ変えて再生成する**。

### 何をどこで担保するか（この切り分けを守る）

| 担保する場所 | 内容 |
|---|---|
| **プロンプト**（1〜3文のまま） | 用途・主題・スタイル・比率の語（`tall 9:16 composition`）・除外（`No text, no logos, no license plate`）・破綻回避の構図（`hands down at their sides` 等） |
| **合成段階**（Chrome側で確実に実現できる） | 媒体別サイズ、セーフゾーン、テキスト量・字数、ロゴ位置とサイズ、CTAの有無、コントラスト比 |
| **レビュー**（判断が必要） | 交通ルールなどの現地整合、媒体適合、ブランド適合、AI破綻、審査リスク |

### レビューの実施方法（エージェントは前提にしない）

**このチェックリストを自分で適用すれば成立する。** 専用エージェントの導入は不要。
社内のマーケエージェント（`mkt-visual-creative` / `mkt-paid` / `mkt-social`）がある環境なら、
同じ材料を渡して意見を求めてもよいが、**任意**。

**配信面が決まっている場合は2回**通す: **生成前**（下のA・仕様を固める）＋**合成後**（下のB・実物を見る）。

レビュー時に手元に揃えるもの: 合成後の画像パス／配信面（媒体・Paid or オーガニック）／出力サイズ／
画像内テキストの全文／使用プロンプト／ロゴの扱い。

#### A. 生成前に確認する（プロンプトを書く前）

| 確認項目 | なぜ必要か |
|---|---|
| ブランドガイドラインの有無 | あれば色・トーン・ロゴ使用規則はそこで決まる。無ければ**「仮で組んだ」と明示して渡す**（勝手に確定させない） |
| NG表現・避けたい色味 | 後から言われると作り直しになる。競合を連想させる色・過去に不評だった表現を先に聞く |
| 既存素材の有無 | 流用できるなら生成しない方が速く、ブランド適合も確実 |
| 配信面（媒体 / Paid or オーガニック） | サイズと審査要件が決まる。未定なら未定のまま進め、**確定後に必ず再レビューする** |

#### B. 合成後にレビューする（納品前）

| 観点 | 見るポイント |
|---|---|
| **媒体ごとの作法** | 1枚を全媒体に使い回していないか。Instagram の世界観をそのまま X や LinkedIn に載せると浮く。最低でも**比率と余白は媒体ごとに作り分ける**（合成側で対応できる） |
| **法規制** | 薬機法（効能・効果の断定的表現）／景表法（優良誤認・有利誤認、「No.1」「最安」等の根拠の有無）／ステマ規制（インフルエンサー起用なら投稿側に `#PR` / `#ad` が必要）。**疑わしければ止めて確認する。自分で「たぶん大丈夫」と判断しない** |
| **権利** | 肖像権（実在の人物に似ていないか）／商標（ロゴ・エンブレム・キャラクターの写り込み。実測でメーカーロゴ酷似が生成された事例あり）／素材の出所（AI生成 / ストック / 自社制作）を記録しているか（6-4） |
| **媒体仕様** | サイズ・セーフゾーン・テキスト量・審査要件は**変わる**。配信直前に媒体の公式ヘルプで確認する |
| **アクセシビリティ** | 文字と背景のコントラスト比（本節末尾の実測スクリプト）。WCAG AA 以上を目安にする |
| **現地整合** | 交通ルール等（次項の表） |

> **このファイルに媒体仕様の固定値を書かないのは意図的。** 「Meta は画像内テキスト20%まで」のような
> かつてのルールは既に廃止されており、古い基準を書き込むと**誤った指摘を生む**。
> 数値が必要な場面では、その都度媒体の公式ソースを見る。

### レビュー結果の反映ルール

| 指摘の種類 | 対応 |
|---|---|
| サイズ・セーフゾーン・テキスト量・ロゴ位置・コントラスト | **合成をやり直す**（画像は再生成しない。HTMLの数値を直して再出力＝数秒） |
| 被写体・構図・現地整合（左ハンドル等）の問題 | **1点だけ変えて再生成**（セクション3の反復ルール） |
| 訴求・コピーの方向性 | ユーザーに確認する（勝手に変えない） |

### 交通・乗り物の現地整合チェック（レビュー観点。プロンプトには書かない）

日本向けの車・道路が写る場合は、**生成後に拡大して**確認する。

| 確認項目 | 正しい状態 |
|---|---|
| 走行車線 | 車は**中央線より左**（日本は左側通行）。右車線を走っていたら不合格 |
| ハンドル位置 | **右ハンドル**。運転席が写る構図なら要確認。写らない構図にするのが安全 |
| ナンバープレート | 日本の書式は再現されない。**空白・ぼかし・画角外**のいずれかにする |
| 車のエンブレム・車名バッジ | 実在メーカーのロゴに酷似したものが出る（実測）。**車の後部・前部を写さない角度**にするのが確実 |
| 標識・看板の文字 | 崩れるので写さない／遠景にする |

> 実例（2026-07-29）: `No logos` を指定しても車の後部にトヨタのロゴに酷似したエンブレムと崩れた文字が生成された。
> **角度を変える（真横・遠景）ことで解消**した。プロンプトに禁止語を足すより構図を変える方が効く。

### コントラスト比の実測（合成後）

文字を背景に直接乗せる場合、**AI背景は局所的に明暗が激しい**ので目視だけで判断しない。

```bash
python3 - "{{IMG}}" <<'PY'
import sys
from PIL import Image
im = Image.open(sys.argv[1]).convert("RGB"); W,H = im.size
def lum(c):
    f=[v/255 for v in c]; f=[v/12.92 if v<=.03928 else ((v+.055)/1.055)**2.4 for v in f]
    return .2126*f[0]+.7152*f[1]+.0722*f[2]
def ratio(box, fg):                      # box=文字が乗る領域(相対), fg=文字色
    l,t,r,b = (int(v*s) for v,s in zip(box,(W,H,W,H)))
    px = list(im.crop((l,t,r,b)).resize((40,40)).getdata())
    L2 = sorted(lum(p) for p in px)      # 背景の明暗の幅を見る
    L1 = lum(fg)
    def cr(a,b): a,b=max(a,b),min(a,b); return (a+.05)/(b+.05)
    print(f"  背景の最暗/最明での比: {cr(L1,L2[-1]):.2f} / {cr(L1,L2[0]):.2f}")
    print("  → 大見出し(太字48px以上)は3.0以上、本文は4.5以上が必要（WCAG AA）")
ratio((0.10,0.14,0.90,0.30), (255,255,255))   # 見出し領域と文字色を指定
PY
```

不足する場合は、**文字色を変える／文字下に半透明の帯（スクリム）を敷く／文字位置を低コントラストな面に移す**。
画像を作り直す必要はない（合成側で解決できる）。

---

## 7.6. SVG を併記するかの判定（既定ルール）

**線引きは「誰がレイアウトを決めたか」。** AIが絵として描いた部分はベクターにできない。
**自分で座標を書いた部分だけ**がSVGでレイヤーになる。

| 成果物 | SVG | SVGの中身 |
|---|---|---|
| **合成バナー・広告クリエイティブ**（背景AI＋文字/ロゴを自分で配置） | **既定で出す** | 背景＝画像 / 見出し・サブ・CTA＝テキストレイヤー / ロゴ＝ベクター。先方で文言差し替えができる |
| **アイコン** | **SVGが主**（PNGは出さない） | potraceで完全ベクター。CSSで色が変わる |
| **データのグラフ・チャート** | **既定で出す** | 最初からSVGで描く。軸ラベル・凡例がテキストのまま残る |
| インフォグラフィック・図解 | 要求時のみ | HTMLで組んでいるのでSVG版は手座標での書き直し＝二重実装。SVGは**自動折り返しをしない** |
| **AI生成の絵そのまま / UIモック / 写真風ビジュアル** | **出さない** | `<image>` 1個の箱。編集できる要素はゼロ |

### AI生成画像をSVGに包まない理由（2026-08-31 実測 / 1672×941 のUIモック）

| | サイズ | 編集できる要素 |
|---|---|---|
| PNG | 1.03 MB | — |
| PNGを包んだSVG | 1.37 MB（**1.33倍**） | **1個**（`<image>` タグ＝画像1枚） |

`image_gen` はラスターしか返さず（10-3）、フルカラー画像をベクター化する手段も無い（`potrace` は白黒シルエット専用）。
**「一貫性のためにSVGも付ける」は、編集不能な1.33倍のファイルを毎回増やすだけ。やらない。**
UIモックで本当に編集可能なファイルが要るなら、SVGではなく **Claude Design のハンドオフ**（セクション11）。

### 納品時の伝え方

- SVGを出したとき: 「文言・色はSVG側のテキストレイヤーで直せる」と一言添える
- 出さなかったとき: 「これはAIが描いた絵なのでSVG化しても編集できない」と**理由を言う**（黙って省略しない）

---

## 8. 実行手順（まとめ）

0. **作りたいものを判定する**（冒頭の使い分け表）。インフォグラフィック・図解・アイコンなら**セクション10へ**（以下の手順は使わない）
1. 認証モードを確認する（セクション1）
2. **用途・主題・スタイルの3点だけ**ユーザーに確認する。細部は聞き出しすぎない（盛ると外れる）
   - 保存先と寸法も確認する。用途で言ってもらえれば寸法はこちらで決める
   - **ロゴ・文言が入るかも確認する**（入るなら背景に余白を空けさせる／セクション7）
   - **配信面が決まっているなら、セクション7.5-A の確認項目も先に潰す**（ブランドGL・NG表現・既存素材）
3. セクション0のテンプレで**1〜3文の英語プロンプト**を作る。ユーザーの日本語をそのまま長い英語に膨らませない
4. **バックグラウンドで実行**する（Bash ツールの `run_in_background: true`）。終了時に自動通知が来る
5. 保存先を確定して回収する（セクション4）
6. 実寸を確認し、必要ならリサイズする（セクション5）
7. **納品前チェック（セクション6）を実施する** — ①配信サイズで全体を見る ②200%拡大で文字・反射・手を見る
   → ゲート（指示追従／文字／致命的破綻）を1つでも落としたら作り直し。点数は平均で救わない
8. ロゴ・文言が必要なら合成する（セクション7）。**合成もの・グラフ・アイコンは SVG も既定で出す**（判定はセクション7.6）。AI生成の絵そのまま・UIモックは**PNGのみ**
9. **配信面が決まっているなら、セクション7.5-B のレビューを通す**（媒体作法・法規制・権利・媒体仕様・
   コントラスト・現地整合）。指摘は「合成やり直し」か「1点変更で再生成」に振り分ける
10. 画像をユーザーに見せ、**直したい点を1つ聞く**（セクション3）
   - 拡大確認の結果（問題なし／ここが弱い）も一緒に伝える
   - 成果物の置き場所は `Output/[カテゴリ]/[企業名・案件名]/` のルールに従う

### ハングした場合

まれに `codex exec` が長時間ハングする（CPU 0%・出力0バイト・画像未生成）。
そのため**起動時に PID を残しておく**（セクション2のコマンドの末尾を `& echo $! > "$OUT/.codex.pid"; wait $!` にする）。

```bash
ps -o etime=,pid=,command= -p "$(cat "$OUT/.codex.pid")"   # 経過時間を確認
kill "$(cat "$OUT/.codex.pid")"                            # この生成だけ止める
```

**`pkill -f "codex exec"` は使わない。** 複数枚を並行生成しているときや、別で codex-build が走っているときに
**それらも全部殺す**。必ず PID 指定で1つだけ止める。

止めて再実行すると 1〜2分で正常終了することが多い。5分以上進捗がなければ止めてよい。

---

## 9. 写真の人物を似せたい場合（例外・上級）

**`image_gen` は参照画像を受け取れない**（プロンプトに写真のパスを書いても、ただの文字列として無視される）。
写真をそのまま反映させようとすると「それっぽい別人」になる。

代わりに **2段構え（ビジョン言語化リレー）** を使う。ここだけはプロンプトが長くなるが、
「顔の描写」という単一目的に絞っているので破綻しにくい。

**Step 1: `-i` で写真を読ませ、特徴を英語で言語化させる**（`-i` は「モデルが画像を見る」ためのフラグ）

```bash
printf '%s' "Describe the facial features of the person in the attached photo in detailed English
for an illustration prompt: face shape, hairstyle and length, eyes, eyebrows, glasses, facial hair,
skin tone, and build. Do NOT beautify or heroize. Avoid: angular jaw, long flowing hair, muscular hero build.
Output only the description." | codex exec \
  --sandbox workspace-write \
  -c sandbox_workspace_write.network_access=true \
  --skip-git-repo-check \
  -C "$OUT" \
  -o "$OUT/face-desc.txt" \
  -i "{{写真の絶対パス}}" -
```

> `-i` は複数ファイルを取れる形（variadic）なので、`-i 画像 "プロンプト"` と書くとプロンプトが画像引数に飲まれて
> `No prompt provided` になる。**プロンプトは stdin で渡し、最後に `-` を置く**。

**Step 2: Step 1 の描写文を画像プロンプトに焼き込んで生成する**（セクション2のテンプレを使う）

- 人数が増えるほど破綻する。**1人ずつ生成して後段で並べる**方が確実
- ピクセル忠実な合成が要る場合の本筋は OpenAI `images.edit`（`gpt-image-1.5` / `input_fidelity=high`）だが、
  **APIキーが必要**なので現環境では上のリレーが現実解

---

## 10. インフォグラフィック・図解とアイコン

**インフォグラフィックは `image_gen` を使わない。** 文字と図が主役なので HTML＋Chrome で完結する（セクション7と同じ書き出し方法）。
AI生成が必要になるのは**アイコンを自作する場合だけ**。

### 10-1. アイコンの調達（2つの選択肢）

| | ①AI生成→ベクター化 | ②既製アイコンセット |
|---|---|---|
| **見た目の質** | **優れる**（線が繊細・実物に忠実） | 素朴・重い |
| 概念の自由度 | **何でも作れる**（ハンドル、チャイルドシート等） | セット内のみ。無い概念は意味をずらして代用するしかない |
| 所要 | 1個あたり1〜2分（並行実行可）＋正規化 | **数秒** |
| データ量 | 3〜13KB（複雑な絵は重い） | 0.4〜1.3KB |
| 画風の統一 | 正規化すれば揃う。ただし個体差は残る | **完全に揃う** |

**判断**: 見た目を優先するなら①。急ぎ・点数が多いなら②。**代用が必要になる概念が出たら①に切り替える**。

**既製セットの導入**（MIT / ISC。商用可。ライセンス表記の要否はクライアントに確認）
```bash
npm install bootstrap-icons lucide-static     # 合計 約4,000個
# 実体: node_modules/bootstrap-icons/icons/*.svg, node_modules/lucide-static/icons/*.svg
ls node_modules/bootstrap-icons/icons | grep -i "calendar"     # 名前で探す
```
`-fill` が付く名前は塗りつぶし版。線画と混ぜると画風が崩れるので**どちらかに統一する**。

### 10-2. AI生成アイコンの4工程パイプライン（実証済み）

**プロンプト書式を1文字も変えず、概念だけ差し替える**（画風を揃えるため）。

```bash
# ① 生成（複数個は並行実行してよい。thread_id で回収するので混ざらない）
codex exec --sandbox workspace-write -c sandbox_workspace_write.network_access=true \
  -c model_reasoning_effort="low" --skip-git-repo-check -C "$GEN" --json \
  "Use the built-in image_gen tool RIGHT NOW to generate exactly one image.
Do not read any files. Do not write any script. Do not use any API key.

Prompt: A single flat icon of {{概念}}, solid black silhouette, centered, plain white background, simple geometric shapes, thick even strokes, no text.

After image_gen returns, copy the generated PNG to ./icon.png in the current working directory using cp." \
  </dev/null > "$WORK/events.jsonl" 2> "$WORK/stderr.log"

# ② 正規化（これを飛ばすとセット感が出ない）
#    二値化 → 余白トリム → 900pxに収めて 1000x1000 の中央へ配置
magick icon.png -colorspace gray -threshold 60% -trim +repage \
  -resize 900x900 -gravity center -background white -extent 1000x1000 icon.pbm

# ③ ベクター化 ＋ CSSで色を変えられるようにする
potrace icon.pbm -s -o icon.svg
python3 - icon.svg <<'PY'
import re,sys,pathlib
p=pathlib.Path(sys.argv[1]); s=p.read_text()
s=re.sub(r'fill="#0+"','fill="currentColor"',s)      # 固定色 → currentColor
s=re.sub(r'\s(width|height)="[^"]*"','',s,count=2)   # 固定サイズを外しCSSに任せる
p.write_text(s)
PY
```

**④ HTMLに「インラインで」埋め込む（最重要）**

```python
import re, pathlib
def ic(path, cls="ic"):
    s = pathlib.Path(path).read_text()
    s = re.sub(r'<\?xml.*?\?>', '', s, flags=re.S)
    s = re.sub(r'<!DOCTYPE.*?>', '', s, flags=re.S)
    s = re.sub(r'<metadata>.*?</metadata>', '', s, flags=re.S)
    return re.sub(r'<svg ', f'<svg class="{cls}" ', s, count=1).strip()

html = f'<div class="icobox">{ic("icon.svg")}</div>'   # ← 文字列としてHTMLに差し込む
```

```css
svg.ic{display:block;color:#3A4250}      /* インラインなら color が効く */
.icobox svg.ic{width:76px;height:76px}
.brand svg.ic{color:#FF5C04}             /* 場所ごとに色を変えられる */
```

> **`<img src="icon.svg">` では CSS の色が効かない（実測）。**
> 外部参照のSVGは別ドキュメントとして読み込まれるため、`currentColor` が親のCSSを継承しない。
> `color:` を指定しても既定の黒で描かれる。**既製アイコンセットでも同じ**。
> 色を制御したいアイコンは必ずインライン化する。ロゴのように色を変えないものは `<img>` でよい。

### 10-3. 検証で分かったこと（判断の根拠）

| 論点 | 結果 |
|---|---|
| `gpt-image-2` はSVGを出力できるか | **できない**。ラスターのみ。ベクターが必要なら potrace 等でトレースする |
| トレースすると黒い四角が残るか | **残らない**。`magick` に `-negate` を付けると背景が反転して四角になる。付けない |
| データ量 | AI生成→トレースは既製の3〜10倍（例: 地球儀 13KB / 既製 1.3KB）。数十個でも実害は出にくい |
| 画風は揃うか | **正規化すれば揃う**。②を飛ばすと生成キャンバスがバラバラ（1254×1254 と 1536×1024 が混在した実例あり） |
| プロンプトの指示は守られるか | **守られないことがある**。`solid black silhouette` と書いても線画で出た（結果は良好だったので採用） |
| 個体差 | 残る。細い絵（雪の結晶）と塗り寄りの絵（チャイルドシート）が混ざると重さが不均一になる。**気になる1〜2個だけ再生成する** |

### 10-4. インフォグラフィックの構成（濃い情報量を成立させる型）

参考にした構造（縦長ポスター型）。**1080×1500 前後**で4セクション・12〜15項目が読みやすい上限。

| 要素 | 実装 |
|---|---|
| ヘッダー | 濃色の帯にロゴ＋タイトル＋サブコピー |
| **セクション見出し** | ブランド色の**リボン**（中央寄せの帯）。`text-transform:uppercase` ＋ `letter-spacing:.10em` |
| **段の区切り** | 背景色を交互に（白 / `#F4F1ED`）。罫線だけだと段が読み取れない |
| 3列・4列 | `display:flex` ＋ `.col + .col{border-left:2px solid}` で縦の区切り線 |
| 2×2 | `display:grid;grid-template-columns:1fr 1fr` |
| **数字の見せ場** | 各セクションに1つ、大きい数字（100px超）＋短い説明。単調さを防ぐ |
| フッター | 出典・時点・ドラフト表記 |

**文字サイズの下限**: 見出し23px / 本文17px（1080px幅基準）。これ未満はSNSの縮小表示で読めない。

### 10-5. 数字と事実の扱い（絶対に守る）

- **数字を作らない。** 実データをユーザーからもらう、または公開情報を出典付きで調べる
- 法規・要件（免許・保険・年齢制限など）は**一次情報で裏取りするか、未確認と明記する**
- フッターに「時点」と「ドラフトである旨」を必ず入れる
- グラフを作る場合は `dataviz` スキルを読んでから（配色・軸・凡例の規約がある）

---

## 11. UI・画面モック（例外扱い。本筋は Claude Design）

**用途は「見た目の方向性を数案ざっと見る」ことだけ。** 画面構成の詰めと実装への受け渡しは
Claude Design（claude.ai/design → Export → Hand off to Claude Code）が本筋。**絵からは実装に渡せない。**

このセクションに限り、セクション0の `No text.` 既定と 6-1 の文字ゲートを**外す**（ラベルが無いと画面として成立しないため）。

**SVGは出さない。** AIが絵として描いた画面はベクター化できず、包んでも `<image>` 1個の箱になる（実測はセクション7.6）。編集可能なものが要るなら Claude Design のハンドオフ。

### プロンプト雛形

```
UI mockup for a marketing analytics dashboard.
Desktop browser window with a left sidebar, four KPI cards across the top,
one large line chart and a data table below, wide 16:9 composition.
Clean light theme, geometric sans-serif.
No company names, no long paragraphs.
```

- **比率は言葉で寄せる**（`wide 16:9 composition`）→ 生成後に `sips` で PC 1600×900 等へ（セクション5）
- **社名・固有名詞・実数値は書かせない。** 誤字る。ダミーの一般語に留める
- 日本語ラベルが要るときは `Japanese UI labels.` を**1文だけ**足す（同時に2点変えない）
- **既存アプリのトンマナに寄せたい場合**: 参照画像は渡せないので、**セクション9のビジョン言語化リレー**で
  既存画面のスクショを `-i` で読ませ、配色・余白・角丸・書体の特徴を英語化してからプロンプトに焼き込む
- 案を並べたいときは1回1枚なので複数回実行する（各1〜2分）

### このセクションでのゲート（6-1 の文字ゲートを差し替える）

| ゲート | 合格条件 |
|---|---|
| 指示追従 | 依頼した画面構成・比率になっている |
| 画像内の文字 | 文字の写り込みは**可**。ただし社名・固有名詞・実数値が含まれていたら不合格 |
| 構造の破綻 | ありえない情報階層・辻褄の合わないグラフ・矛盾した状態表示が目立たない |

**やってはいけない**: このモックを実装の仕様として渡すこと。生成画像は成立しない構造（矛盾した状態、
嘘のグラフ）を含み、Claude Code はそれを忠実に再現してしまう。実装に渡すのは Claude Design の
ハンドオフバンドル（トークン実値・コンポーネント仕様・レイアウト階層を含む）。

---

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| 狙った画像にならない | プロンプトが長すぎる。セクション0のテンプレで1〜3文に切り直す。除外条件が前に来ていないか確認 |
| 直したいのに別物になる | 一度に2点以上変えている。**1点だけ**変える。保ちたい要素は毎回書き直す |
| 文字が誤字る・崩れる | 仕様。`No text.` を入れて余白を空けさせ、文字は Chrome 合成で載せる（セクション7）。UI・画面モックは例外（セクション11） |
| ロゴが再現されない・崩れる | AIに描かせてはいけない。実SVGを後から重ねる（セクション7） |
| 手や指が不自然 | 拡大確認（セクション6）→ 手を写さない構図に1点変えて再生成。ピクセル修正はできない |
| Figmaで文字だけ直したい | PNGではなくSVGを渡す（セクション7後半）。フォントは Inter にしておく |
| UIモック・写真風の絵もSVGで欲しいと言われた | 包んでも `<image>` 1個で編集できず1.33倍重くなるだけ（セクション7.6の実測）。理由を伝え、必要なら Claude Design のハンドオフを案内する |
| アイコンの色がCSSで変わらない | `<img src="x.svg">` になっている。**インライン埋め込みにする**（セクション10-2） |
| アイコンの画風が揃わない | 正規化（余白トリム→正方形化）を飛ばしている（セクション10-2の②）。プロンプト書式も1文字も変えない |
| ベクター化したら黒い四角が出た | `magick` の `-negate` を外す（背景が反転している） |
| インフォグラフィックの文字が崩れる | インフォグラフィックに `image_gen` を使ってはいけない。HTML＋Chromeで作る（セクション10） |
| セットに欲しいアイコンが無い | 意味をずらして代用せず、AI生成→ベクター化で作る（セクション10-2） |
| 出力0バイトのまま止まる | プロンプト末尾の `</dev/null` を忘れている（stdin 待ち） |
| `Could not resolve host` / ネットワークエラー | `-c sandbox_workspace_write.network_access=true` があるか確認。`--full-auto` は使わない（deprecated・ネットワークが落ちる） |
| `Not inside a trusted directory ...` で即終了 | `--skip-git-repo-check` を付ける |
| 画像が指定パスに無い | `~/.codex/generated_images/<session id>/` から自分で `cp`（セクション4） |
| 並行生成で別の画像を拾った | **フォルダ選択に `ls -t` を使わない**。`events.jsonl` の `thread.started.thread_id` で特定する（セクション4）。`--json` なしで回した場合のみ stderr の `session id:` 行 |
| サイズが指示と違う | Codex 経由ではサイズ指定不可。`sips -z` でリサイズ（セクション5） |
| 透過背景にならない | `gpt-image-2` は透過非対応。生成後に背景を抜く |
| APIキーを求められる | プロンプトに「Do not use any API key. Use the built-in image_gen tool」と明記。`auth_mode` が `chatgpt` か確認 |
| 生成せずに終わる（トークン消費だけ） | ① `codex features list \| grep image_generation` が `stable true` か確認（既定でtrue＝**`--enable image_generation` は no-op なので付けても無意味**） ② true なら原因はプロンプト側。`Prompt:` を英語1〜3文に切り、運転指示（RIGHT NOW / Do not read any files / Do not write any script）が消えていないか確認 ③ `events.jsonl` に `image_gen` 呼び出しイベントがあるかで切り分ける |
| 人物が似ない | `image_gen` は参照画像を受け取れない。**セクション9**のリレー方式にする |
| `codex` コマンドが無い | `npm install -g @openai/codex` |
| フラグが通らない | `codex --version` を確認し、`codex update` で更新 |
