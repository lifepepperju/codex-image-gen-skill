---
name: codex-image-gen
description: Codex CLIの組み込みimage_genツール（ChatGPT Images 2.5世代）で画像生成・編集する。ChatGPTサブスク内で追加課金ゼロ・APIキー不要。プロンプトは12ui型の構造化、Flare/Sunburst向きの判定、レビュー担当（Fable/Opus）を分けた上限3回のレビューループを含む。「画像生成して」「Codexで画像」「バナー作って」「アイコン生成」「サムネ作って」「この画像を直して」等のキーワードで起動。
---

# Codex Image Gen — ChatGPTサブスクで画像生成

Codex CLI の組み込み画像生成ツール（`image_gen`）を呼び出し、追加課金なしで画像を作る・直す。

**最終検証: 2026-09-09 / codex-cli 0.153.4 / ChatGPT Images 2.5 世代**（コマンドは実機で通したもの）
**出典**: OpenAI 公式 [Image generation guide](https://developers.openai.com/api/docs/guides/image-generation) / [Image prompting guide](https://developers.openai.com/api/docs/guides/image-prompting) /
[Zenn: GPT Image 2.5 Flare と Sunburst の使い分け](https://zenn.dev/neotechpark/articles/514d034e19f370) / [12ui: GPT Image 2 vs 2.5 比較（12本のUIプロンプト）](https://12ui.com/gpt-image-2.5-vs-2)

> **プレースホルダの規約**: `{…}` と `{{…}}` はどちらも**実行前に実値へ置換する**。1つでも残っていたら実行しない
> （`{{OUTPUT_PATH}}` という名前のファイルが作られる）。
> `$WORK` = ログ・PID置き場（スクラッチパッド配下）／`$GEN` = Codex の作業ディレクトリ／`$OUT` = 納品先（`Output/…`）。
>
> **このファイルは2箇所に同一内容で置かれている**（プロジェクト `.claude/skills/` とグローバル `~/.claude/skills/`）。
> 正本はプロジェクト側。修正したら必ず両方に反映する。**レビュー担当エージェント（セクション7-0）の定義ファイルも同じく2箇所**。

---

## 最初に判断: 作りたいものによって手段が変わる

**`image_gen`（AI生成）は写真・イラストの「絵」を作る道具。文字・図・アイコンには使わない。**
ここを間違えると、いくらプロンプトを工夫しても質が上がらない。

| 作りたいもの | 手段 | SVG併記 | 参照 |
|---|---|---|---|
| 写真風・イラストの**背景ビジュアル**（バナー、LPのヒーロー、SNS投稿の絵） | `image_gen`（AI生成） | **絵単体では出さない**（合成すれば下段の扱いになる） | セクション0〜7 |
| **既存画像の一部だけ直す**（色・1要素・背景） | `image_gen` の**編集**（参照画像を渡す） | 元の扱いに従う | **セクション4** |
| **ロゴ・見出し・コピー** | 実ファイル＋実フォントで**後から合成**（HTML＋Chrome） | **既定で出す** | セクション8・**8.6** |
| **インフォグラフィック・図解・比較表・手順図** | **HTML＋Chrome のみ。AI生成は使わない** | 要求時のみ（HTMLの組み直しになる） | **セクション11**・8.6 |
| **アイコン** | AI生成→ベクター化（質が高い）／既製セット（速い）。**AIのラスターをそのまま貼らない** | **SVGが主**（PNGは出さない） | **セクション11** |
| データのグラフ・チャート | HTML/SVG＋`dataviz` スキル＋**実データ**（数字は作らない） | **既定で出す** | セクション11・8.6 |
| 実際の地図・地形 | 地理データSVG（パブリックドメイン等）を調達。アイコンセットには無い | 元がSVG | — |
| 人物・既存素材・スタイルに**似せる** | `image_gen` に**参照画像を渡す**（0.153.4 で可能になった） | 絵単体なので出さない | **セクション10** |
| **UI・画面モック**（雰囲気の確認だけ） | `image_gen`（例外扱い）。**画面構成の詰めと実装への受け渡しは Claude Design が本筋** | **出さない**（8.6の実測） | **セクション12** |

---

## 0. モデルの前提と使い分け判定（Images 2.5 / Flare / Sunburst）

### 0-1. 事実（2026-09-09 時点）

| 項目 | 内容 |
|---|---|
| リリース | 2026-09-08。ChatGPT・ChatGPT Work・**Codex** に「ChatGPT Images 2.5」として展開。API には `gpt-image-2.5-flare` と `gpt-image-2.5-sunburst` の2モデル |
| Flare | 小型・高速。GPT Image 2 より高品質で**レイテンシ最大50%減**。OpenAI が「迷ったらこれ」と位置づける既定 |
| Sunburst | 基礎モデル・精度重視。**編集の制御が効き、複数ターン編集でも崩れにくい**。生成時間は長い |
| 料金 | **両モデル同一**（テキスト入力 $5 / 画像入力 $8 / 画像出力 $30 per 1M tokens）。差は時間だけ |
| **Codex 経由で選べるか** | **選べない。** 0.153.4 の実機バイナリで `image_gen` の引数は `prompt` / `referenced_image_paths` / `num_last_images_to_include` の3つだけ。公式の設定リファレンス・変更履歴にもモデル指定は無い。どちらで出ているかは**非公開**（生成PNGの C2PA 生成元表記は 9/5 も 9/9 も「gpt-image 2.0」で判別不能）。利用者報告では「速い方が既定で切替不可」 |
| API 直叩き | Sunburst を使う唯一の手段。OPENAI_API_KEY・組織認証・従量課金が必要。**本スキルでは使わない**（2026-09-09 決定。将来 Codex にモデル選択が来たら見直す） |

### 0-2. 判定表（生成前に必ず1行で宣言する）

依頼を受けたら、下の表で **Flare 向き / Sunburst 向き** を判定し、ユーザーへの最初の返答に1行で書く。
判定はユーザーへの期待値合わせと、レビューの厳しさを決めるためのもので、**生成は常に Codex**。

| 判定 | 該当する依頼 | 対応 |
|---|---|---|
| **Flare 向き**（通常） | SNS投稿・バナー背景・下書き・複数案の当たり出し・アイコン・UIの雰囲気確認・枚数の多い生成 | 通常フロー。Codex の速さがそのまま利点 |
| **Sunburst 向き** | 本番キャンペーンの主ビジュアル／商品写真の仕上げ／**複数の参照画像を合成**／**同じ画像を何度も直す反復編集**／細部・質感の精度が成果を左右する | Codex で生成するが、**「本来は Sunburst 向きの案件。Codex では選べないので Codex で生成し、レビュー3回上限で詰める」とユーザーに伝える**。レビュー担当は Fable を推奨（7-0） |

**Sunburst 向きと判定したときに変えること**: 判定文を伝える／レビューの点数閾値を 3 → **4** に上げる／編集で直せる範囲は編集で詰める（作り直しより崩れにくい）。

### 0-3. 参考データ（12ui 実測 156枚・中央値・API直叩きの値）

| 品質 | 1枚の料金 | Flare 秒 | Sunburst 秒 | GPT Image 2 秒 |
|---|---|---|---|---|
| low | $0.019 | 16 | 21 | 25 |
| medium | $0.024 | 19 | 22 | 46 |
| high | $0.056 | 23 | 33 | 109 |
| xhigh | $0.090 | 29 | 44 | — |
| max | $0.182 | 44 | 75 | — |

12ui の評者の所見（UI画面12本）: 「Flare は一見きれいでリズム（反復の規則性）の遵守が良い。Sunburst は UX の実用性とリアリズムで優位。12本中 Flare 勝ちは1本」「medium で見た目は決まる。max は4倍の料金で方向性は変わらない」。
Codex 経由（今回の実測）: 新規生成 1枚 **約50秒**、既存画像の編集 **約77秒**（いずれも 1672×941）。

---

## 1. プロンプトの書き方（12ui 型・全種共通）

**2026-09-09 改訂**: 旧版の「英語1〜3文」ルールを、12ui の12本のプロンプトに共通する**構造化された型**に置き換えた（適用範囲は全種。ユーザー決定）。
長さは伸びるが、**「何を」「どんな方向で」「何を守って」が段落単位で分かれている**ので、モデルが構図指示と除外条件を混同しにくい。
今回の実測（セクション0-3の1枚目）は、この型で書いた1回目の生成で 16:9・被写体位置・色・光の指示がすべて通った。

### 1-1. 型（4ブロック。順番を変えない）

```
[1] 宣言（1文）
{成果物の種類} for {誰のための何} focused on {この画像が果たす役割}.

[2] 方向づけ（6軸。該当しない軸は削る。軸名は英語のまま書く）
Let the image embody this visual-system direction:
spatial organization: {構図・比率・被写体の位置・余白の場所};
amount and packing of visible information: {写るものの数・密度};
how color carries identity or meaning: {主色・アクセント・光の色};
typographic voice and hierarchy: {UIモックのみ。絵では削る};
the physical surface the image behaves like: {写真の種類・画材・質感};
decoration alongside functional marks: {装飾の有無・何が装飾を担うか}.
Interpret the direction through the whole image rather than rendering the words as literal objects.

[3] 参照画像（渡すときだけ。役割を1枚ずつ割り当てる）
Treat Image 1 as {役割: the subject to keep / a functional specification / …}.
Use Image 2 for {composition and rhythm}, Image 3 for {color and surface behavior}.
Translate those qualities rather than collaging or reproducing any reference.

[4] 締め（固定文。除外条件はここ＝最後に置く）
Create entirely new visual assets; do not reuse logos, brands, proper nouns, or readable phrases from any reference.
Use the full native-ratio canvas without padding, letterboxing, a device mockup, or a frame.
No text, no logos{, no license plate, no people 等を必要に応じて}.
Return only the finished image.
```

### 1-2. 旧版から引き継ぐルール（型に載せても守る）

1. **[1] の1語目は成果物の種類**（`Ad banner background` / `Product photo` / `App icon` / `UI mockup`）。これだけで仕上がりの水準が合う
2. **除外条件は必ず [4]（最後）**。前に置くとモデルが構図指示と誤解して逆に出る（実測）
3. **飾り言葉を足さない**。`4k, ultra detailed, professional, trending on dribbble, masterpiece, cinematic` は主題をぼかす。12ui の12本にも一切出てこない
4. **文字（コピー・社名・数字）は画像に描かせない**。`No text` を既定にして余白だけ空けさせ、文字は HTML 合成（セクション8）で載せる。**例外は UI・画面モックだけ**（セクション12）
5. **比率は [2] の spatial organization に言葉で書く**（`a wide 16:9 composition` / `tall 9:16 composition` / `square composition`）。ピクセル数は書かない（無視される。セクション6）
6. **1回で完成させない**。土台を1枚出し、**1回に1点だけ**変える（セクション4）。変更は既存の行を差し替える形にし、文を足していかない
7. **人物が入るなら崩れにくい構図を [2] に書く**（`seen from behind with their hands down at their sides` / 手に物を持たせない / 小さく配置 / 顔のクローズアップを避ける / 群衆を入れない）。実証済みの言い換えはセクション7末尾

### 1-3. 例（実測で通ったもの・種類別）

**バナー背景（2026-09-09 実測。1回目で 1672×941 の 16:9、車は右下、空と道が左上の余白、朝の暖色＋ティール1点、35mm風の粒状感まで通った）**
```
Ad banner background for a car-rental service aimed at overseas visitors driving through rural Japan, focused on making the trip feel easy and open.
Let the image embody this visual-system direction: spatial organization: a wide 16:9 composition with a compact car small at the lower right and open sky and road filling the upper left as empty space for copy; amount and packing of visible information: one car, one road, one horizon, nothing else; how color carries identity or meaning: warm early-morning light with a single teal accent from the car; the physical surface the image behaves like: a candid 35mm travel photograph with natural grain; decoration alongside functional marks: none, the landscape itself is the only ornament.
Photorealistic. Create entirely new visual assets; do not reuse logos, brands, or readable phrases. Use the full native-ratio canvas without padding, letterboxing, or a frame.
No text, no logos, no license plate, no people.
```
> この1枚はレビューで**車が中央線の右側を走っていた**（日本は左側通行）ことが指摘された。直すときは spatial organization に
> `the car driving in the left-hand lane, Japanese left-hand traffic` を**1点だけ足して**編集または再生成する（セクション4）。

**商品カット（型の当て方）**
```
Product photo for a matte black shampoo bottle on an e-commerce listing, focused on making the label area read clean at thumbnail size.
Let the image embody this visual-system direction: spatial organization: a square composition, single bottle centered with generous padding; amount and packing of visible information: one product, one soft reflection, nothing else; how color carries identity or meaning: neutral light-gray-to-white studio gradient so the black bottle carries all the contrast; the physical surface the image behaves like: premium studio product photography with softbox highlights; decoration alongside functional marks: none.
Photorealistic. Create entirely new visual assets; do not reuse logos or brands. Use the full native-ratio canvas without padding, letterboxing, or a frame.
No text, no logos, no watermark.
```

**アイコン（セット全体で [1] 以外を1文字も変えない。概念だけ差し替える → セクション11-2）**
```
App icon for {概念} in a travel-service icon set, focused on instant recognition at 24px.
Let the image embody this visual-system direction: spatial organization: a square composition, one object centered with generous margin; amount and packing of visible information: one object only, simple geometric shapes, thick even strokes; how color carries identity or meaning: solid black silhouette on plain white; the physical surface the image behaves like: a printed pictogram; decoration alongside functional marks: none.
Create entirely new visual assets; do not reuse logos or brands. Use the full native-ratio canvas without padding or a frame.
No text.
```

**UI・画面モック** → 12ui の型をそのまま使う（セクション12）。

### 1-4. 複雑な依頼が来たときの分解方針

1枚に複数の要素を詰め込ませない。**要素ごとに分けて生成し、レイアウトは後段（HTML / 資料 / 画像編集）で合成する。**

| ユーザーの依頼 | やること |
|---|---|
| 「キャッチコピー入りのバナー」 | 背景ビジュアルだけ生成 → 文字は HTML / スライド側で載せる |
| 「3つのサービスを並べた図」 | アイコンを3枚別々に生成 → 並べるのは資料側 |
| 「人物＋商品＋店内＋ロゴ入り」 | 主役を1つに絞る。残りは別カットか、後段合成 |
| 「前回の画像のここだけ直して」 | **編集**（参照画像に前回の画像を渡し、`Change only X` で1点だけ直す）→ セクション4 |
| 「この写真の人に似せて」「この素材の雰囲気で」 | **参照画像**を渡して役割を割り当てる → セクション10 |

---

## 2. 前提チェック

```bash
grep -o '"auth_mode"[^,}]*' ~/.codex/auth.json
```

- `"chatgpt"` → そのまま続行（ChatGPTサブスク内・追加課金なし）
- `"api_key"` → API従量課金が発生する。**ユーザーに確認を取ってから**続行
- ファイルなし → `codex login` を案内して中断

機能フラグは 0.153.4 でも既定ONなので通常は何も足さなくてよい（確認したい場合のみ）:

```bash
codex features list | grep -E "image_generation"   # stable / true が既定
```

---

## 3. 実行コマンド（既定テンプレート・新規生成）

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

Prompt: {{セクション1の型で書いた英語プロンプト}}

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
> 納品先への配置は**回収時に Claude 側の `cp` で行う**（セクション5）。
>
> 上の「RIGHT NOW / Do not read any files」等は **Codex への運転指示**であって画像プロンプトではない。
> ここは固定文なので毎回同じにする。**`Prompt:` の中身だけがセクション1の型**。
> `{{…}}` が1つでも残っている状態で実行しないこと。

### フラグの役割と、間違えやすい点

| 項目 | 内容 |
|---|---|
| `--sandbox workspace-write` ＋ `network_access=true` | **これで画像生成は通る（実測）**。`--dangerously-bypass-approvals-and-sandbox` は不要。安全側のこちらを既定にする |
| `</dev/null`（末尾） | **必須**。`codex exec` は位置引数でプロンプトを渡しても stdin を読みに行き、閉じていないと無言で固まる（実測） |
| `--skip-git-repo-check` | git 外・trusted 未登録のディレクトリでも即終了しないようにする |
| `-C "{絶対パス}"` | 作業ディレクトリを明示。`$PWD` をクォート内で展開させると意図しない場所で動くことがある |
| `-c model_reasoning_effort="low"` | 画像生成に高い推論量は不要。既定（`high`）のままだと遅い |
| プロンプトは英語 | 精度が高い。日本語＋曖昧な指示だと生成せずに終わることがある |
| `events.jsonl` 冒頭の `error` 行2つ | `--dangerously-bypass-hook-trust is enabled` は **cmux のラッパーが付けるフラグ由来とみられ、生成には無関係**（2026-09-09 実測。2回とも exit 0 で正常生成） |

---

## 4. 反復ワークフロー: 1点だけ直す（編集 or 作り直し）

**2026-09-09 実測で「既存画像の編集」ができるようになった**（codex-cli 0.153.4。旧版の「編集はできない」は失効）。
`image_gen` に `referenced_image_paths` で元画像を渡すと、**同じサイズ・同じ構図のまま指定箇所だけ**変わる。

実測: 1672×941 の青い車を「車の色だけ赤に」→ 出力も 1672×941、画像全体の平均差 5.5/255（車の部分 21/255、空 1.7/255）。所要 77 秒。

### 4-1. 編集か作り直しかの判断

| 状況 | 手段 |
|---|---|
| 構図・被写体は良い。**色・1要素・背景・光**だけ変えたい | **編集**（4-2）。崩れにくく、レビューで合格した部分を守れる |
| 被写体の位置・比率・構図から違う | **作り直し**（セクション3。プロンプトの該当行を1点だけ差し替える） |
| 手・指・反射などの**局所破綻** | まず編集で「その部位を変える」を試す。直らなければ**その部位を写さない構図**に1点変えて作り直す |

### 4-2. 編集コマンド（元画像を `$GEN` に置いて渡す）

```bash
cp "{{元画像の絶対パス}}" "$GEN/base.png"          # 参照は -C 配下に置く（sandbox の読み取り範囲）

codex exec \
  --sandbox workspace-write \
  -c sandbox_workspace_write.network_access=true \
  -c model_reasoning_effort="low" \
  --skip-git-repo-check \
  -C "$GEN" \
  --json \
  "Use the built-in image_gen tool RIGHT NOW to edit exactly one existing image. Do not write any script. Do not use any API key.
The image to edit is ./base.png in the current working directory. If you need to see it first, use view_image on that path. Then call image_gen with referenced_image_paths set to that file.

Edit instruction: {{Change only X. Keep A, B, C, and everything else unchanged.}}

After image_gen returns, copy the generated PNG to ./edit.png in the current working directory using cp.
Then report: (1) the absolute path image_gen returned, (2) whether you passed referenced_image_paths and its value, (3) any error message from the tool." \
  </dev/null > "$WORK/events.jsonl" 2> "$WORK/stderr.log" &
echo $! > "$WORK/codex.pid"
wait $!
echo "EXIT=$?"
```

**編集指示の書き方（OpenAI 2.5 公式ガイド準拠）**
- `Change only X.` で変える点を**1つ**だけ書く
- `Keep … unchanged.` で**守るもの**を列挙する（構図・光・カメラ角度・背景・被写体の同一性）。**毎回書き直す**（省略するとドリフトする）
- 一度に2点以上変えない。どの変更が効いたか分からなくなる

**注意（実測）**: 編集で**実在メーカーのロゴが出る**ことがある。青→赤の色替え編集で、リアの一般的な丸いエンブレムが
**トヨタのマークそのもの**に変わった。**編集後も必ずセクション7のレビューを通す**（レビュー回数にカウントする）。
ロゴが出たら「エンブレムを無地にする」編集を1点入れるか、車の前後を写さない角度に作り直す。

**未検証**: `num_last_images_to_include`（同一スレッドの直前画像を参照する方式。`codex exec … resume --last` と併用する想定）。
ファイルパスで渡す 4-2 の方式で足りるので、必要になるまで使わない。

---

## 5. 生成画像の回収（ここが最大のハマりどころ）

`image_gen` の原本は **`~/.codex/generated_images/<thread_id>/exec-<uuid>.png`** に出る（0.153.4。旧版の `call_*.png` から名前が変わった）。
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
- `--json` の events に **`image_gen` の呼び出し自体は出ない**（agent_message と command_execution だけ）。編集で参照を渡したかは、Codex に報告させた文面（4-2 の (2)）で確認する

---

## 6. サイズ（Codex 経由では指定できない）

`image_gen` の引数に size / quality / background は無い（0.153.4 バイナリで確認）。**サイズ・品質・透過は指定できない**。
プロンプトにピクセル数を書くのは無意味なので**書かない**。**寸法は生成後に必ず自分で合わせる。**

ただし**縦横比は文章で効く**（実測）: `tall 9:16 composition` → **941×1672**、`a wide 16:9 composition` → **1672×941**（2026-09-09）。
狙いの比率を [2] の spatial organization で作り、最後に `sips` で目的サイズへ合わせるのが確実。

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

### 参考: API を直接叩く場合の仕様（公式ドキュメント・2026-09-09 時点）

Codex 経由では使えず、本スキルでも使わない（セクション0-1）。将来切り替える場合の前提として:

- モデル: `gpt-image-2.5-flare` / `gpt-image-2.5-sunburst`（`gpt-image-2` 系は公式ガイドから記述が消えた）
- サイズ: `1024x1024`（推奨）`1536x1024` `1024x1536`、または任意の `WIDTHxHEIGHT`（両辺16の倍数 / 比率 1:3〜3:1 / 最大辺 3840px / 総画素 655,360〜8,294,400）
- 品質: `low` / `medium` / `high` / `xhigh` / `max` / `auto`（2.5 で `xhigh` `max` が追加）
- 背景: `transparent` / `opaque` / `auto`（2.5 は透過に対応）
- 編集: マスク（アルファ付き・同寸）／参照画像複数／`previous_response_id` での複数ターン
- Codex 同梱の CLI `~/.codex/skills/.system/imagegen/scripts/image_gen.py` は `--model gpt-image-2.5-*` を通すが、品質は low/medium/high/auto のみ・サイズは 1024 系のみに制限される

---

## 7. 納品前チェック（レビュー担当を分ける・上限3回で必ず結論を出す）

**縮小表示では気づかない破綻が必ず残っている。** 実例（2026-07-29 GoWithGuideバナー）: 等倍では自然に見えたが、
手を2倍に拡大したら**指が溶けて本数も判別できない状態**だった。**拡大確認を飛ばして納品しない。**
実例（2026-09-09 レンタカー背景）: 等倍で違和感なし。レビューで**車が中央線の右側**（日本は左側通行）と、編集後に**実在メーカーのロゴ**が出たことを検出。

判定方式は OpenAI 公式の画像評価ガイド（[Image Evals](https://developers.openai.com/cookbook/examples/multimodal/image_evals)）に合わせる。

### 7-0. 誰が見るか（レビュー担当と最終判定者を分ける）

**レビューは「プロンプトを書いた本人」ではなく、別のサブエージェントに見せる。** 書いた側は自分の狙いに引っ張られて、
「そう見えるはず」で通してしまう。見る側を分けると、指示追従ゲートを**書かれたプロンプトの文面だけ**で判定できる。

| 役割 | 担当 | 何をするか |
|---|---|---|
| レビュー担当 | サブエージェント **`image-reviewer-fable`** または **`image-reviewer-opus`**（`.claude/agents/`。プロジェクトとグローバルの両方に同一内容） | 全体を1回 Read → 部位を相対座標で選ぶ → 切り出しスクリプトで200%拡大 → 各切り出しを Read → ゲート＋点数＋弱点＋「次の1点変更」を定型で返す |
| **最終判定者** | **Claude Code 本体**（このセッションを動かしているモデル。2026-09-09 時点では Fable 5.1） | レビュー担当の報告を鵜呑みにせず、**指摘箇所を自分でも Read で確認**してから合否を決める。ループの上限管理と最終評価文（7-7）も本体の責任 |

**生成を始める前に、ユーザーへ「レビュー担当は Fable か Opus か」を1回だけ確認する**（セクション9の手順2）。
指定が無ければ既定を使う。既定と、選ぶときの判断材料は下表。

**既定: `image-reviewer-opus`（Opus・エフォート high）。Sunburst 向き案件（0-2）では `image-reviewer-fable`（Fable・high）を提案する。**

判断材料（2026-09-09 実測。同じ1枚＝右車線走行の欠陥がある 1672×941 のレンタカー背景を、4体に同時レビューさせた結果。n=1 なので傾向として読む）

| 選択 | 所要 | メリット | デメリット | 向く場面 |
|---|---|---|---|---|
| **Opus / high（既定）** | 3分32秒 | 検出力は Fable と同等（右車線・汎用エンブレム・車のサイズ超過・情報量超過をすべて検出）。ゲート＋点数＋「次の1点変更」の定型を最も忠実に守り、**修正案を1つに絞れた**（左右反転を「コピー余白が右へ動くので不可」と正しく退けた）。Fable より安い | 「なぜそう見えるか」の切り分けは Fable より浅い | 通常案件（Flare 向き） |
| **Fable / high** | 3分38秒 | 拡大時の網目模様を「再サンプリングのモアレで原画の欠陥ではない」と切り分け、光源の位置と影の向きの整合まで確認。**誤検出が最も少ない** | 時間・コストが最も高い。今回は定型を離れて修正案を複数並べた | Sunburst 向き案件。人物・反射・ガラスが多い絵 |
| medium（どちらのモデルも） | 約2分30秒 | high より約1分速い。致命欠陥（右車線）は4体全員が検出。Opus/medium は「谷の風景が欧州寄りで日本らしくない」という用途上の弱点を唯一指摘した | 修正案が3〜4点並び、「1回1点」の反復ルールと衝突する。拡大時の網目を欠陥寄りに解釈した（Fable/medium） | 下書きの当たり出しで枚数を見るとき。納品前レビューには使わない |

エフォートを high にした理由: 致命欠陥自体は medium でも全員が見つけたが、**「直す1点」を1つに絞れたのは high だけ**。
ループ（7-7）では担当の提案をそのまま採用するので、high の方が手戻りが少ない。1回あたりの差は約1分（3回回しても3分）。
Fable を既定にしなかった理由: 今回の1枚では検出力に差が出ず、Opus の方が安い。差が出るのは反射・モアレ・光の整合のような「見えているものの解釈」で、
それが成果を左右する Sunburst 向き案件だけ Fable に切り替える。

**レビュー担当への渡し方**（Agent ツール。`subagent_type` に上のエージェント名を指定。プロンプトには下の7項目を全部入れる）

```
レビュー対象の画像: {絶対パス}
生成に使ったプロンプト（指示追従ゲートの基準）: {全文}
用途の補足: {配信面・国・現地条件。例: 日本国内の道路＝左側通行・右ハンドルが正。ナンバーは空白/ぼかし/画角外が正。実在メーカーのエンブレムが読めたら不合格}
判定: {Flare 向き / Sunburst 向き（Sunburst 向きなら点数の合格ラインは 4）}
切り出しスクリプト: {qa_crop.py の絶対パス（7-3）}
切り出し画像の出力先: {スクラッチパッド配下の絶対パス}
出力形式（この順で）: 画像サイズ / 見た部位と座標 / ゲート判定（指示追従・文字・致命的破綻、各1行の根拠） / 点数（レイアウト・適合・視覚品質） / 総合（合格・不合格） / 弱い箇所（最大3、場所を明示） / 次の1点変更の提案（不合格時のみ・1つだけ）
```

> **出力形式はプロンプト側にも必ず書く。** 2026-09-09 の実測では、4体中3体が「定義された出力形式が届いていない」と答え、
> エージェント定義本文だけに書いた形式は伝わらなかった。定義ファイルは model / effort / tools と手順書を持ち、形式は呼び出し側で毎回渡す。
> **用途の補足を省かない。** 右車線走行を4体全員が検出できたのは「左側通行が正しい」を補足で渡していたから。プロンプトに無い期待は補足に書く。

レビュー担当はファイルを作らない（切り出し画像の保存だけ）。**画像を直すのは本体側**（セクション4）。

Codex（GPT）にレビューさせない: 生成した側と同じモデル系で見ると、同じ盲点を共有する。
AI判定ツールも使わない（7-5）。

### 7-1. 判定は「ゲート」＋「点数」。平均で救わない

**ゲート（1つでも落ちたら不合格 → 1点変更して再生成 or 編集）**

| ゲート | 合格条件 |
|---|---|
| 指示追従 | 依頼した主題・構図・スタイル・比率になっている |
| 画像内の文字 | **文字が写り込んでいない**（`No text` 指定のため）。写っていたら不合格。文字を入れる場合は**綴り・記号・大文字小文字まで完全一致**が条件。**UI・画面モックのみ例外**（セクション12のゲートで判定する） |
| 致命的破綻ゼロ | 7-2 の重点チェックで、人物の解剖学・反射・接点に明確な破綻がない。**実在メーカーのロゴ・エンブレムが読めたら不合格** |

**点数（0〜5 / 3以上で合格。Sunburst 向き案件は 4 以上）**

| 項目 | 見るところ |
|---|---|
| レイアウト・余白 | ロゴとコピーを置ける余白が意図した位置にあるか |
| ブランド適合 | **既存クリエイティブと並べて**違和感がないか |
| 視覚品質 | 光・色・被写界深度・質感が破綻していないか |

**判定ルール（重要）**: どれか1つでも閾値割れなら**総合は不合格**。
高い点で平均して合格にしない（例: 視覚品質5でも文字ゲートが落ちたら不合格）。

> 我々のパイプラインは**文字をChromeで後乗せする**（セクション8）ため、文字ゲートは構造的に満たしやすい。
> AIに文字を描かせる運用は、この一点だけで不合格リスクが跳ね上がる。

### 7-2. 見る順番（2026年時点の優先度）

生成モデルの改善で「手を見れば分かる」時代は終わりつつある。**現在いちばん破綻が残るのは文字と反射**。
ただし手が壊れた実例もあるので、手も引き続き見る。

| 優先 | 見るもの | 具体的に何を疑うか |
|---|---|---|
| 1 | **画像内の文字** | 看板・のれん・値札・服のロゴ。意味不明な字形が最も残りやすい |
| 2 | **反射・映り込み** | ガラス・水面・車体・眼鏡。**映っているものが現場と一致しているか**（現行モデルが最も苦手） |
| 3 | **実在ロゴ・現地整合** | 車のエンブレム（**編集で出た実例あり**）、車線の左右、ナンバー、標識。8.5 の現地整合表 |
| 4 | **ディテール密度の不均一** | 一部だけ異様にシャープ／のっぺり。境目が不自然 |
| 5 | **解剖学の複合破綻** | 手・指（本数/融合/関節なし）、腕の本数、肩と腕の接続、脚と靴の左右、足首のねじれ。**単独より複数箇所の違和感が重なる方が危険信号** |
| 6 | 手と物の接点 | カメラ・カバン・傘の持ち方、ストラップが途切れる／体を貫通する |
| 7 | 反復パターン | 群衆・木・提灯・石畳の不自然なコピー |
| 8 | 背景の意味的不整合 | あり得ない構造・接続、消失点の破綻、遠景の人物の顔崩れ |

### 7-3. 2段階で見る（片方だけでは不十分）

**① 実際に表示されるサイズで全体を見る**
配信面のサイズ（Storyならスマホ全画面相当）で見て、目線の流れ・可読性・第一印象を判断する。
拡大だけ見て「粗が無いからOK」と判断しない。**見られるサイズで良く見えるかが本題**。

**② 200%に拡大して部位ごとに見る**（切り出しスクリプトで切り出して Read で目視）

**座標は絶対値でハードコードしない。** 画像サイズは毎回違う（Codex経由ではサイズを指定できないため）。
PIL の `crop` は**範囲外を黒で埋めて例外を出さない**ので、座標がずれると
「大部分が黒い画像」を見て「問題なし」と判定してしまう（実測: 941×1672 の画像に 1080×1920 前提の座標を当てると 68% が黒）。

**必ず 0〜1 の相対座標で指定し、範囲外を検出させる。** レビュー担当と本体が同じスクリプトを使う:

```bash
# qa_crop.py（スクラッチパッドに置く。引数: 画像 出力dir name=l,t,r,b …。範囲外は止まる・黒画素率も出す）
cat > "$WORK/qa_crop.py" <<'PY'
import sys, os
from PIL import Image
src, out = sys.argv[1], sys.argv[2]
os.makedirs(out, exist_ok=True)
im = Image.open(src).convert("RGB"); W, H = im.size
print(f"画像サイズ: {W}x{H}")
for spec in sys.argv[3:]:
    name, box = spec.split("=", 1)
    l, t, r, b = (float(v) for v in box.split(","))
    L, T, R, B = int(l*W), int(t*H), int(r*W), int(b*H)
    assert 0 <= L < R <= W and 0 <= T < B <= H, f"{name}: 範囲外 {(L,T,R,B)} / 画像 {(W,H)}"
    crop = im.crop((L, T, R, B))
    crop.resize(((R-L)*2, (B-T)*2), Image.LANCZOS).save(os.path.join(out, f"qa-{name}.png"))
    px = list(crop.resize((32, 32)).getdata())
    black = sum(1 for c in px if max(c) < 20) / len(px)
    print(f"  {name}: {(L,T,R,B)} → qa-{name}.png  黒画素 {black:.0%}")
PY
python3 "$WORK/qa_crop.py" "{{PATH}}" "$WORK/qa" hands=0.33,0.61,0.70,0.76 car=0.72,0.66,0.97,0.90
```

- **切り出し前に必ず画像全体を1度 Read して被写体位置を確認する**。座標は毎回そこから決める。前回の数値を流用しない
- 切り出し後は「黒が大半でないか」も併せて確認する（黒画素率が高ければ座標ミス）

破綻が見つかったら、**編集で1点直す**か、**その部位を写さない構図に1点だけ変えて再生成**する（セクション4）。

### 7-4. 記録を残す（後で必ず必要になる）

採用した画像ごとに、以下をユーザーへの報告に含める（成果物と一緒に残す）:

- 使ったプロンプト（全文）と、編集した場合は編集指示（全文）
- 生成が **AI** であること、どのツールか（Codex `image_gen` / ChatGPT Images 2.5 世代。Flare/Sunburst のどちらかは非公開）
- **Flare 向き / Sunburst 向きの判定**（セクション0-2）
- **レビュー担当（Fable / Opus）と、何回目のレビューで合格したか**（または上限到達）
- チェック結果（合格したゲート／点数／弱い箇所）

理由: 広告の**AI開示**（7-6）や、ブランド側の承認・監査で「これはAI生成か」を必ず聞かれる。
後から思い出せないと答えられない。

### 7-5. AI判定ツールには頼らない

Hive / Illuminarty のような「AI生成判定」サービスや、C2PA等のメタデータ判定は**当てにしない**。

- SNS（X・Instagram等）の圧縮で、検出器が見ている低次のアーティファクトが壊れる
- Chrome合成やリサイズを通すと、由来メタデータは基本的に残らない
- 判定は**レビュー担当の目 + 本体の目 + 7-2 のチェックリスト**が主。ツールは補助にもならない前提で組む

### 7-6. 「AI生成」表記を入れるか（開示）

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

### 7-7. レビューループ（上限3回。無限に回さない）

```
生成①（新規） → レビュー① ─合格→ 納品
                    │不合格
                    ▼ 1点変更（編集 or 作り直し。レビュー担当の「次の1点変更」を採用）
生成② → レビュー② ─合格→ 納品
                    │不合格
                    ▼ 1点変更
生成③ → レビュー③（最終） → 本体が最終評価 → 納品（結論を必ず明記）
```

**ルール**
- **レビューは最大3回**。編集も作り直しも1回に数える。3回目のレビューで終わり、4回目は回さない
- 1回のループで変えるのは**1点だけ**（セクション4）。レビュー担当の「次の1点変更の提案」をそのまま採用する。採用しない場合は理由を1行残す
- 合格が出ても、**本体が指摘箇所を自分で Read して納得できなければ不合格扱いにしてよい**（最終責任は本体）
- 既定は**3回まで自動で回し、結果と経過をまとめてユーザーに見せる**。ユーザーが「毎回見たい」と言えば各レビュー後に止めて1点を確認する
- ループを回している間、途中の画像は捨てずに `$WORK` に `gen-1.png` `gen-2.png` … と残す（最終評価で見比べる）

**最終評価文（3回目まで行ったら、どちらかを必ず書く。曖昧に納品しない）**

| 結論 | 書き方（そのまま報告に含める） |
|---|---|
| **A. 納得できる内容** | 「レビュー担当（Fable/Opus）の判定は合格（ゲート3つ通過・点数 n/n/n）。指摘箇所を本体でも確認し、**納得できる内容として納品**する」 |
| **B. 質は担保できなかったが上限到達** | 「レビュー3回の上限に達した。**質は担保できていないが、一旦の最終アウトプットとして納品**する。残っている弱点: {場所と内容}。次に試すなら: {1点}」 |

1〜2回目で合格した場合も、「n回目のレビューで合格。担当は Fable/Opus」と回数と担当を書く（7-4）。

### 直すより避けるほうが安い（プロンプト側の予防）

- **手を写さない／目立たせない構図にする**（後ろ姿・ポケットに手・遠景・腰から下を切る）
- **物を持たせない**（カメラ・スマホ・傘を持つ手は最も崩れる）
- 人物を小さく配置する（大写しほど崩れが目立つ）
- 顔のクローズアップを避ける
- 群衆を入れない（遠景の顔が崩れる）
- **車は前後を写さない角度にする**（エンブレム・ナンバーが出ない。編集で実在ロゴが出た実例あり）

**実証済みの言い換え（2026-07-29 GoWithGuideバナー）**

| 崩れた書き方 | 直った書き方 |
|---|---|
| `... seen from behind, warm afternoon light.`（身振りする手が生成され指が溶けた） | `... all seen from behind with their hands down at their sides, warm afternoon light.` |

`hands down at their sides` を入れるだけで、手が下がって小さく写り、指の融合・本数エラーが消えた。
**変更したのはこの1点だけ**（他の文はそのまま）。副産物として人物が下に寄り、上部のコピー余白が広がった。

> このチェックは**目視が最終判断**。ユーザーに渡すときは「拡大して確認済み／ここが弱い」まで伝える。
> 見つけた破綻を黙って納品しない。

---

## 8. ロゴ・文言を載せる（AIに描かせず、後から重ねる）

**ロゴと文字は絶対に `image_gen` に描かせない。** ロゴは再現されず、社名・数字は誤字る。
**背景ビジュアルだけAIに作らせ、ロゴの実ファイルと本物のフォントを後から重ねる。**

### 手順

1. **背景だけ生成**（セクション1の型）。[4] に `No text, no logos.` を入れ、[2] の spatial organization で**コピーを置く余白を空けさせる**
   （例: `open sky filling the upper left as empty space for copy`）
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
   **PNGロゴは表示幅を元の横幅以下に収める**（拡大すると甘くなる）。透過が無い場合はセクション8末尾の白抜き手順を使う。
3. **HTMLを書いてローカルChromeで指定サイズに書き出す**（下のテンプレ）
4. **セクション7の納品前チェックを実施する**

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
文言差し替え・色調整が先方側でできるようになり、追加コストは数十秒。ただし**何にでもSVGを付けるのではない** — 判定はセクション8.6。

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

## 8.5. マーケレビュー・ゲート（媒体適合はプロンプトに入れず、ここで担保する）

**媒体のベストプラクティスをプロンプトに詰め込まない。** セーフゾーン・テキスト量・交通ルール・コントラスト比などを
プロンプトに書き足すと [2] の方向づけがぼけて画像の質が落ちる。
代わりに **合成後にマーケレビューを通し、指摘があれば1点だけ変えて編集または再生成する**（セクション7-7 のループ回数に含める）。

### 何をどこで担保するか（この切り分けを守る）

| 担保する場所 | 内容 |
|---|---|
| **プロンプト**（セクション1の型） | 用途・主題・6軸の方向づけ・比率の語・除外（`No text, no logos, no license plate`）・破綻回避の構図（`hands down at their sides` 等） |
| **合成段階**（Chrome側で確実に実現できる） | 媒体別サイズ、セーフゾーン、テキスト量・字数、ロゴ位置とサイズ、CTAの有無、コントラスト比 |
| **レビュー**（判断が必要） | 交通ルールなどの現地整合、媒体適合、ブランド適合、AI破綻、審査リスク |

### レビューの実施方法（エージェントは前提にしない）

**このチェックリストを自分で適用すれば成立する。** 7-0 のレビュー担当は「AI破綻」を見る役で、媒体・法規・権利の判断は本体が行う。
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
| 既存素材の有無 | 流用できるなら生成しない方が速く、ブランド適合も確実。**似せたい素材があるなら参照画像として渡す**（セクション10） |
| 配信面（媒体 / Paid or オーガニック） | サイズと審査要件が決まる。未定なら未定のまま進め、**確定後に必ず再レビューする** |

#### B. 合成後にレビューする（納品前）

| 観点 | 見るポイント |
|---|---|
| **媒体ごとの作法** | 1枚を全媒体に使い回していないか。Instagram の世界観をそのまま X や LinkedIn に載せると浮く。最低でも**比率と余白は媒体ごとに作り分ける**（合成側で対応できる） |
| **法規制** | 薬機法（効能・効果の断定的表現）／景表法（優良誤認・有利誤認、「No.1」「最安」等の根拠の有無）／ステマ規制（インフルエンサー起用なら投稿側に `#PR` / `#ad` が必要）。**疑わしければ止めて確認する。自分で「たぶん大丈夫」と判断しない** |
| **権利** | 肖像権（実在の人物に似ていないか。**参照画像に人物写真を使った場合は本人の同意を確認**）／商標（ロゴ・エンブレム・キャラクターの写り込み。実測でメーカーロゴ酷似・**編集でトヨタのマークそのもの**が生成された事例あり）／素材の出所（AI生成 / ストック / 自社制作）を記録しているか（7-4） |
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
| 被写体の色・1要素・背景・光 | **編集で1点直す**（セクション4-2） |
| 構図・現地整合（車線・左ハンドル等）の問題 | **1点だけ変えて再生成**（セクション4） |
| 訴求・コピーの方向性 | ユーザーに確認する（勝手に変えない） |

### 交通・乗り物の現地整合チェック（レビュー観点。プロンプトには書かない）

日本向けの車・道路が写る場合は、**生成後に拡大して**確認する。

| 確認項目 | 正しい状態 |
|---|---|
| 走行車線 | 車は**中央線より左**（日本は左側通行）。右車線を走っていたら不合格（**2026-09-09 の1枚目で発生**） |
| ハンドル位置 | **右ハンドル**。運転席が写る構図なら要確認。写らない構図にするのが安全 |
| ナンバープレート | 日本の書式は再現されない。**空白・ぼかし・画角外**のいずれかにする |
| 車のエンブレム・車名バッジ | 実在メーカーのロゴに酷似したものが出る（実測。**編集後にトヨタのマークが出た実例**）。**車の後部・前部を写さない角度**にするのが確実 |
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

## 8.6. SVG を併記するかの判定（既定ルール）

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

`image_gen` はラスターしか返さず（11-3）、フルカラー画像をベクター化する手段も無い（`potrace` は白黒シルエット専用）。
**「一貫性のためにSVGも付ける」は、編集不能な1.33倍のファイルを毎回増やすだけ。やらない。**
UIモックで本当に編集可能なファイルが要るなら、SVGではなく **Claude Design のハンドオフ**（セクション12）。

### 納品時の伝え方

- SVGを出したとき: 「文言・色はSVG側のテキストレイヤーで直せる」と一言添える
- 出さなかったとき: 「これはAIが描いた絵なのでSVG化しても編集できない」と**理由を言う**（黙って省略しない）

---

## 9. 実行手順（まとめ）

0. **作りたいものを判定する**（冒頭の使い分け表）。インフォグラフィック・図解・アイコンなら**セクション11へ**（以下の手順は使わない）
1. 認証モードを確認する（セクション2）
2. ユーザーに確認する（1往復で済ませる）:
   - **用途・主題・スタイルの3点**。細部は聞き出しすぎない
   - 保存先と寸法。用途で言ってもらえれば寸法はこちらで決める
   - **ロゴ・文言が入るか**（入るなら背景に余白を空けさせる／セクション8）
   - **参照画像があるか**（人物・既存素材・スタイル → セクション10）
   - **レビュー担当は Fable か Opus か**（未指定なら既定の Opus。Sunburst 向き案件なら Fable を提案する。セクション7-0）
   - 配信面が決まっているなら、セクション8.5-A の確認項目も先に潰す（ブランドGL・NG表現・既存素材）
3. **Flare 向き / Sunburst 向きを判定して1行で伝える**（セクション0-2）。Sunburst 向きなら「Codex では選べないので Codex で生成し、レビュー3回上限で詰める」と添える
4. セクション1の型で**プロンプトを組む**（4ブロック。除外は最後。飾り言葉なし。比率は言葉で）
5. **バックグラウンドで実行**する（Bash ツールの `run_in_background: true`）。終了時に自動通知が来る
6. 回収する（セクション5）。`$WORK/gen-1.png` のように回数付きで残す
7. 実寸を確認し、必要ならリサイズする（セクション6）
8. **レビュー担当に渡す**（セクション7-0）→ 不合格なら「次の1点変更」を採用して**編集（4-2）か作り直し（3）**→ 再レビュー。**上限3回**（セクション7-7）
9. **本体が最終評価する**。指摘箇所を自分でも Read し、**A（納得できる）/ B（上限到達・質は担保できず一旦の最終）** のどちらかを明記する
10. ロゴ・文言が必要なら合成する（セクション8）。**合成もの・グラフ・アイコンは SVG も既定で出す**（判定はセクション8.6）。AI生成の絵そのまま・UIモックは**PNGのみ**
11. **配信面が決まっているなら、セクション8.5-B のレビューを通す**（媒体作法・法規制・権利・媒体仕様・コントラスト・現地整合）
12. 画像をユーザーに見せ、報告に**プロンプト全文・判定（Flare/Sunburst 向き）・レビュー担当と回数・最終評価文・弱い箇所**を含める（セクション7-4）
   - 成果物の置き場所は `Output/[カテゴリ]/[企業名・案件名]/` のルールに従う

### ハングした場合

まれに `codex exec` が長時間ハングする（CPU 0%・出力0バイト・画像未生成）。
そのため**起動時に PID を残しておく**（セクション3・4-2 のコマンドは `& echo $! > "$WORK/codex.pid"; wait $!` の形にしてある）。

```bash
ps -o etime=,pid=,command= -p "$(cat "$WORK/codex.pid")"   # 経過時間を確認
kill "$(cat "$WORK/codex.pid")"                            # この生成だけ止める
```

**`pkill -f "codex exec"` は使わない。** 複数枚を並行生成しているときや、別で codex-build が走っているときに
**それらも全部殺す**。必ず PID 指定で1つだけ止める。

止めて再実行すると 1〜2分で正常終了することが多い。5分以上進捗がなければ止めてよい。

---

## 10. 参照画像を使う（人物・既存素材・スタイル）

**0.153.4 の `image_gen` は参照画像を受け取れる**（`referenced_image_paths`。2026-09-09 に「既存画像の編集」で実測）。
旧版の「参照画像は渡せない」は失効した。

**実測できている範囲**: 生成済み画像を参照にした**編集**（構図維持・1要素の変更）。
**未検証**: 人物写真を参照にして「似せる」新規生成、複数参照画像の合成。最初に使うときは1枚で試し、似ているかを 7-0 のレビューで見る。

### 手順

1. 参照画像を `$GEN` にコピーする（`ref1.png` `ref2.png` …。sandbox の読み取り範囲に入れる）
2. プロンプトの [3] で**役割を1枚ずつ割り当てる**（12ui の書き方）:
   - 被写体を保つ: `Treat Image 1 as the subject to keep: preserve identity, face, hair, and build; change only the setting.`
   - 機能・構成だけ借りる: `Treat Image 1 as a functional specification, not a design reference: preserve only its layout logic.`
   - 見た目の要素を分けて借りる: `Use Image 2 for composition and rhythm, Image 3 for color and surface behavior. Translate those qualities rather than collaging or reproducing any donor.`
3. 運転指示（セクション4-2 と同じ形）で `view_image` → `image_gen` に `referenced_image_paths` を渡すよう Codex に指示し、**渡したかを報告させる**
4. [4] の締めに `do not reuse pixels, logos, brands, proper nouns, or readable phrases from any reference` を入れる（参照の丸写しを防ぐ）
5. **人物を参照にした場合は本人の同意と肖像権を確認**してから納品する（セクション8.5-B）

### 参照が効かない・似ないときの旧方式（ビジョン言語化リレー）

`-i` で写真を読ませて特徴を英語で言語化し、その描写文をプロンプトの [2] に焼き込む2段構え。参照が使えなかった時代の手法だが、
**「顔の描写」という単一目的に絞る**ので今も破綻しにくい。

```bash
printf '%s' "Describe the facial features of the person in the attached photo in detailed English
for an illustration prompt: face shape, hairstyle and length, eyes, eyebrows, glasses, facial hair,
skin tone, and build. Do NOT beautify or heroize. Avoid: angular jaw, long flowing hair, muscular hero build.
Output only the description." | codex exec \
  --sandbox workspace-write \
  -c sandbox_workspace_write.network_access=true \
  --skip-git-repo-check \
  -C "$GEN" \
  -o "$WORK/face-desc.txt" \
  -i "{{写真の絶対パス}}" -
```

> `-i` は複数ファイルを取れる形（variadic）なので、`-i 画像 "プロンプト"` と書くとプロンプトが画像引数に飲まれて
> `No prompt provided` になる。**プロンプトは stdin で渡し、最後に `-` を置く**。

- 人数が増えるほど破綻する。**1人ずつ生成して後段で並べる**方が確実

---

## 11. インフォグラフィック・図解とアイコン

**インフォグラフィックは `image_gen` を使わない。** 文字と図が主役なので HTML＋Chrome で完結する（セクション8と同じ書き出し方法）。
AI生成が必要になるのは**アイコンを自作する場合だけ**。

### 11-1. アイコンの調達（2つの選択肢）

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

### 11-2. AI生成アイコンの4工程パイプライン（実証済み）

**プロンプトは [1] の概念だけ差し替え、他は1文字も変えない**（画風を揃えるため。型はセクション1-3のアイコン例）。

```bash
# ① 生成（複数個は並行実行してよい。thread_id で回収するので混ざらない）
codex exec --sandbox workspace-write -c sandbox_workspace_write.network_access=true \
  -c model_reasoning_effort="low" --skip-git-repo-check -C "$GEN" --json \
  "Use the built-in image_gen tool RIGHT NOW to generate exactly one image.
Do not read any files. Do not write any script. Do not use any API key.

Prompt: App icon for {{概念}} in a travel-service icon set, focused on instant recognition at 24px.
Let the image embody this visual-system direction: spatial organization: a square composition, one object centered with generous margin; amount and packing of visible information: one object only, simple geometric shapes, thick even strokes; how color carries identity or meaning: solid black silhouette on plain white; the physical surface the image behaves like: a printed pictogram; decoration alongside functional marks: none.
Create entirely new visual assets; do not reuse logos or brands. Use the full native-ratio canvas without padding or a frame.
No text.

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

> 2026-07-29 に実証したのは旧版のワンライナー（`A single flat icon of {概念}, solid black silhouette, centered, plain white background, simple geometric shapes, thick even strokes, no text.`）。
> 上の 12ui 型は同じ要素を型に載せ直したもので、**セット単位での再検証はまだ**。最初のセットは 2〜3 個作って画風が揃うかを見てから量産する。揃わなければ旧ワンライナーに戻してよい。

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

### 11-3. 検証で分かったこと（判断の根拠）

| 論点 | 結果 |
|---|---|
| `image_gen` はSVGを出力できるか | **できない**。ラスターのみ。ベクターが必要なら potrace 等でトレースする |
| トレースすると黒い四角が残るか | **残らない**。`magick` に `-negate` を付けると背景が反転して四角になる。付けない |
| データ量 | AI生成→トレースは既製の3〜10倍（例: 地球儀 13KB / 既製 1.3KB）。数十個でも実害は出にくい |
| 画風は揃うか | **正規化すれば揃う**。②を飛ばすと生成キャンバスがバラバラ（1254×1254 と 1536×1024 が混在した実例あり） |
| プロンプトの指示は守られるか | **守られないことがある**。`solid black silhouette` と書いても線画で出た（結果は良好だったので採用） |
| 個体差 | 残る。細い絵（雪の結晶）と塗り寄りの絵（チャイルドシート）が混ざると重さが不均一になる。**気になる1〜2個だけ再生成する**（編集で「線を太く」も試せる） |

### 11-4. インフォグラフィックの構成（濃い情報量を成立させる型）

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

### 11-5. 数字と事実の扱い（絶対に守る）

- **数字を作らない。** 実データをユーザーからもらう、または公開情報を出典付きで調べる
- 法規・要件（免許・保険・年齢制限など）は**一次情報で裏取りするか、未確認と明記する**
- フッターに「時点」と「ドラフトである旨」を必ず入れる
- グラフを作る場合は `dataviz` スキルを読んでから（配色・軸・凡例の規約がある）

---

## 12. UI・画面モック（例外扱い。本筋は Claude Design）

**用途は「見た目の方向性を数案ざっと見る」ことだけ。** 画面構成の詰めと実装への受け渡しは
Claude Design（claude.ai/design → Export → Hand off to Claude Code）が本筋。**絵からは実装に渡せない。**

このセクションに限り、セクション1の `No text.` 既定と 7-1 の文字ゲートを**外す**（ラベルが無いと画面として成立しないため）。

**SVGは出さない。** AIが絵として描いた画面はベクター化できず、包んでも `<image>` 1個の箱になる（実測はセクション8.6）。編集可能なものが要るなら Claude Design のハンドオフ。

### 12-1. 12ui の型（12本すべてがこの骨格。そのまま使う）

```
Treat Image 1 as a functional specification, not a design reference: preserve only its interaction archetype, information needs, and useful actions.
Create a new original, aesthetically exceptional, polished, and usable interface for {何のプロダクト} for {誰} focused on {ユーザーが達成すること}.
Recompose it freely into a distinctly different product, content domain, hierarchy, and visual identity.
{方向づけ: 下の A / B / C のいずれか1つ}
Create fresh copy and entirely new visual assets; do not reuse pixels, logos, brands, proper nouns, readable phrases, illustrations, photos, icons, or a recognizable arrangement from any reference.
Use the full native-ratio canvas without padding, cropping, stretching, letterboxing, a device mockup, or a surrounding frame.
Return only the finished interface design.
```

**方向づけの3パターン**（12本の内訳: A が6本、B が3本、C が3本）

| | 書き方 | いつ使う |
|---|---|---|
| **A. 6軸で言葉で指定** | `Let the new design embody this visual-system direction: spatial organization: …; amount and packing of visible information: …; how color carries identity or meaning: …; typographic voice and hierarchy: …; the physical surface the interface behaves like: …; decoration alongside functional marks: …. Interpret the direction through the whole interface system rather than rendering the words as literal objects or content.` | 参照画像が無い／方向性を言葉で持っている |
| **B. 2枚の参照の交差** | `Find a coherent, aesthetically resolved intersection between the distinct visual-system logics in Images 2 and 3. Transfer their reasoning rather than combining their literal motifs, content, or layouts.` | 「この2つの雰囲気の間」を狙う |
| **C. 3枚の参照に役割分担** | `Build one coherent new visual system using Image 2 for composition and rhythm, Image 3 for typography and information density, and Image 4 for color and surface behavior. Translate those abstract qualities rather than collaging, averaging, or reproducing any donor.` | 参照素材が揃っている（既存アプリのスクショ等） |

**Image 1（機能の種）が無い場合**は1文目を削り、2文目から始める（`Create a new original … interface for …`）。
参照画像の渡し方はセクション10（`$GEN` にコピー → `referenced_image_paths`）。

**6軸の書き方の実例（12ui P01「Theatrical Explainer」より）**
```
spatial organization: a full-bleed theatrical set with UI reduced to a threshold;
amount and packing of visible information: a title, an invitation, and darkness;
how color carries identity or meaning: near-black scenes cut by one spectral light source;
typographic voice and hierarchy: carved, weathered title lettering with whispering captions;
the physical surface the interface behaves like: a lit film set or a game title screen;
decoration alongside functional marks: fog, grain, and set dressing carry the mood.
```
実務向けの控えめな例（12ui P07「Procedural Collaboration」より）:
```
spatial organization: sober settings columns and forms with a persistent side nav;
amount and packing of visible information: complete and procedural, every field earning its row;
how color carries identity or meaning: white utility surfaces with one trustworthy brand accent;
typographic voice and hierarchy: plain product sans with hierarchy by spacing, not size;
the physical surface the interface behaves like: the unglamorous control room of a real product;
decoration alongside functional marks: toggles, dividers, and helper text only.
```

### 12-2. 運用上の注意

- **比率は文中の語で寄せる**（`a wide 16:9 desktop canvas` / `a tall mobile canvas` を spatial organization に入れる）→ 生成後に `sips` で PC 1600×900 等へ（セクション6）
- **社名・固有名詞・実数値は書かせない。** 誤字る。ダミーの一般語に留める（型の「do not reuse … proper nouns」がその役）
- 日本語ラベルが要るときは `Japanese UI labels.` を**1文だけ**足す（同時に2点変えない）
- **既存アプリのトンマナに寄せたい場合**: スクショを参照画像 Image 2〜4 として渡し、C の役割分担で書く（セクション10）。参照が効かなければ旧方式（`-i` で言語化）
- 案を並べたいときは1回1枚なので複数回実行する（各 1〜2分）
- 12ui の所見: **Flare は一見きれい、Sunburst は実用性で優位**。Codex ではどちらか選べないので、レビューでは見栄えより「この画面は使えるか（情報階層・矛盾の有無）」を重く見る

### 12-3. このセクションでのゲート（7-1 の文字ゲートを差し替える）

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
| 狙った画像にならない | 型の [2] に方向づけが無い／[4] の除外が前に来ている／飾り言葉が入っている（セクション1-2）。1点ずつ直す |
| 直したいのに別物になる | 作り直しで一度に2点以上変えている。**編集**（セクション4-2）で `Change only X` にするか、作り直しでも**1点だけ**変える |
| 編集したら関係ない所も変わった | `Keep … unchanged` の列挙が足りない。守るもの（構図・光・カメラ角度・背景）を毎回書き直す |
| 編集で実在メーカーのロゴが出た | 実測あり（トヨタのマーク）。「エンブレムを無地に」の編集を1点入れる、または前後を写さない角度に作り直す。7-1 のゲートで不合格扱い |
| 文字が誤字る・崩れる | 仕様。`No text.` を入れて余白を空けさせ、文字は Chrome 合成で載せる（セクション8）。UI・画面モックは例外（セクション12） |
| ロゴが再現されない・崩れる | AIに描かせてはいけない。実SVGを後から重ねる（セクション8） |
| 手や指が不自然 | 拡大確認（セクション7）→ 編集で直すか、手を写さない構図に1点変えて再生成 |
| レビューが終わらない | 上限は**3回**（セクション7-7）。3回目で本体が最終評価し、A か B を明記して納品する。4回目は回さない |
| レビュー担当の判定に納得できない | 本体が指摘箇所を Read して自分で判断してよい（最終責任は本体）。担当を Fable/Opus で切り替えるのは次回から |
| レビュー担当のエージェントが見つからない | `.claude/agents/image-reviewer-fable.md` / `-opus.md` がプロジェクトかグローバルにあるか確認。無ければ `general-purpose` に `model` を指定して同じ手順書を渡す（エフォートは指定できない） |
| レビュー担当が「出力形式が届いていない」と言う／形式がばらばら | 定義本文の形式が伝わらないことがある（実測）。7-0 の7項目テンプレを使い、**形式をプロンプト側に書く** |
| レビュー担当の修正案が複数並ぶ | エフォートが medium になっていないか確認（high なら1点に絞れる傾向）。複数来たら本体が**最上位の1点だけ**採用する |
| Figmaで文字だけ直したい | PNGではなくSVGを渡す（セクション8後半）。フォントは Inter にしておく |
| UIモック・写真風の絵もSVGで欲しいと言われた | 包んでも `<image>` 1個で編集できず1.33倍重くなるだけ（セクション8.6の実測）。理由を伝え、必要なら Claude Design のハンドオフを案内する |
| アイコンの色がCSSで変わらない | `<img src="x.svg">` になっている。**インライン埋め込みにする**（セクション11-2） |
| アイコンの画風が揃わない | 正規化（余白トリム→正方形化）を飛ばしている（セクション11-2の②）。プロンプトも [1] の概念以外を変えない |
| ベクター化したら黒い四角が出た | `magick` の `-negate` を外す（背景が反転している） |
| インフォグラフィックの文字が崩れる | インフォグラフィックに `image_gen` を使ってはいけない。HTML＋Chromeで作る（セクション11） |
| セットに欲しいアイコンが無い | 意味をずらして代用せず、AI生成→ベクター化で作る（セクション11-2） |
| 出力0バイトのまま止まる | プロンプト末尾の `</dev/null` を忘れている（stdin 待ち） |
| `events.jsonl` の先頭に `error` が2行ある | `--dangerously-bypass-hook-trust is enabled` なら cmux ラッパー由来とみられ無害（exit 0 で正常生成を確認）。それ以外の error は本文を読む |
| `Could not resolve host` / ネットワークエラー | `-c sandbox_workspace_write.network_access=true` があるか確認。`--full-auto` は使わない（deprecated・ネットワークが落ちる） |
| `Not inside a trusted directory ...` で即終了 | `--skip-git-repo-check` を付ける |
| 画像が指定パスに無い | `~/.codex/generated_images/<thread_id>/exec-*.png` から自分で `cp`（セクション5） |
| 参照画像が読まれない | `$GEN`（`-C` の作業ディレクトリ）の外に置いている。中にコピーして相対パスで指示する。Codex に「referenced_image_paths を渡したか」を報告させて確認する（4-2） |
| `referenced image paths are unavailable in this session` | バイナリにこのエラー文がある（未遭遇）。出たら `codex update` の後に再試行し、それでも出るなら旧方式（セクション10末尾）に切り替える |
| 並行生成で別の画像を拾った | **フォルダ選択に `ls -t` を使わない**。`events.jsonl` の `thread.started.thread_id` で特定する（セクション5）。`--json` なしで回した場合のみ stderr の `session id:` 行 |
| サイズが指示と違う | Codex 経由ではサイズ指定不可。`sips -z` でリサイズ（セクション6） |
| 透過背景にならない | Codex 経由では透過を指定できない（引数が無い）。生成後に背景を抜く |
| Sunburst で作ってほしいと言われた | Codex では選べない（セクション0-1）。API 経路は本スキルでは持たない。判定を伝え、レビュー3回上限で詰める |
| APIキーを求められる | プロンプトに「Do not use any API key. Use the built-in image_gen tool」と明記。`auth_mode` が `chatgpt` か確認 |
| 生成せずに終わる（トークン消費だけ） | ① `codex features list \| grep image_generation` が `stable true` か確認（既定でtrue＝**`--enable image_generation` は no-op なので付けても無意味**） ② true なら原因はプロンプト側。運転指示（RIGHT NOW / Do not read any files / Do not write any script）が消えていないか確認 ③ `events.jsonl` の `agent_message` に「生成した」旨があるかで切り分ける |
| 人物が似ない | 参照画像を渡しているか（セクション10）。渡しても似ないなら旧方式（`-i` で言語化）を試す |
| `codex` コマンドが無い | `npm install -g @openai/codex` または `brew install codex` |
| フラグが通らない | `codex --version` を確認し、`codex update` で更新 |
