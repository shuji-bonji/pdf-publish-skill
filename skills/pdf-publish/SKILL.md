---
name: pdf-publish
description: PDF の生成・編集から納品までを品質ゲート付きで編成する出力パイプライン。pdf-writer-mcp で書き、pdf-reader-mcp で読み戻し、pdf-verify-mcp で機械採点（veraPDF）する write → read-back → verify ループを回し、Publish Report 付きで納品する。ユーザーが「PDF にして納品」「PDF/UA で作って」「アクセシブルな PDF」「タグ付き PDF の生成」「電帳法対応の PDF（PDF/A-3 添付）」「品質保証付きで PDF を作って」「PDF を作って検証まで」「フォームを PDF/UA 準拠に」などに言及したら、単発の writer ツール呼び出しで済ませず必ずこの Skill を使う。pdf-trust（受入監査）の対になる送り出し側。
---

# pdf-publish — 品質ゲート付き PDF 出力パイプライン

PDF family の出力（納品）パイプラインを担う Skill。pdf-trust が「受け取った PDF を監査する」
のに対し、こちらは「**送り出す PDF を保証する**」。自前の判定ロジックは持たず、
**合否は必ず pdf-verify-mcp の結果を根拠にする** — writer の warnings は事実の報告であって
合否ではなく、reader の観測は観測であって判定ではない。

中核原則（family 共通 + writer 固有）:

1. 合否判定は verify のみ。verify 未接続のまま「準拠 PDF です」と納品しない
2. 根拠（どのツールの何の結果か）をレポートに必ず明示する
3. **パスは常に絶対パスで、writer には必ず `outputPath` を渡す** — 省略すると PDF が
   base64 で返り、数 MB の PDF で応答上限を溢れさせる（3.9MB → 530 万文字の実測あり）
4. writer のエラーは構造化されている（`code` / `next_actions` / `retryable`）。
   文字列を読んで推測せず、`code` で分岐し `next_actions` の指示に従う →
   [references/error-codes.md](references/error-codes.md)
5. **「宣言」と「適合」を混同しない。** `ensure_pdfa` / `ensure_tagged` は XMP に
   `pdfaid` / `pdfuaid` を書く = **その文書に「規格に沿っています」と名乗らせる**道具であって、
   適合させる道具ではない。**宣言を書いたら必ず対応する flavour を verify で測る**
   （測らないなら宣言も書かない）。「宣言 / 適合 / 検証は別物」

## 前提 MCP

| MCP | 必須/任意 | 役割 |
|---|---|---|
| pdf-writer-mcp (v0.8.0+ / **v0.15.0+ 推奨**) | **必須** | 生成（Tier 0）・編集（Tier A/B）・PDF/UA 修復のすべて。**PDF/A-3b の器付け（`ensure_pdfa`）は v0.15.0 から** |
| pdf-verify-mcp (**v0.7.0+ 推奨**) | 品質ゲート案件では**必須** | identify_conformance / validate_conformance（veraPDF 委譲）/ verify_integrity |
| pdf-reader-mcp (**v0.9.1+ 推奨**) | 推奨 | 読み戻し（テキスト抽出・論理順抽出・フォント・タグ・メタデータの観測） |
| pdf-spec-mcp | 任意 | 違反時の ISO 32000 / 14289 条項の根拠引用。**ISO 19005（PDF/A）は収録外**なので PDF/A の条文は引けない（T2） |

pdf-writer-mcp が未接続なら成立しない。`npx @shuji-bonji/pdf-writer-mcp@latest` の接続を
案内して停止する。品質ゲート水準 `conformance`（後述）の案件で pdf-verify-mcp が無い場合も
中止する — ゲート無しで「準拠」を名乗るのは虚偽になる。reader が無い場合は縮退動作し、
レポートに「読み戻し: 未実施（ツール未接続）」と明記する。

**`ensure_pdfa` / `ensure_tagged` を使う案件は、水準にかかわらず pdf-verify-mcp が必須。**
これらは文書に「規格に沿っています」と名乗らせるため、検査できないまま使うと
**嘘の刻印を押したファイルを納品する**ことになる。verify が無いなら宣言も書かない。

## 手順

### Phase 0 — 要件確認

利用者と次を合意する。曖昧なまま進めない（後半の手戻りがループ上限を浪費する）:

1. **成果物の種別** — 新規生成（text / markdown / table）か、既存 PDF の編集か
2. **品質ゲート水準** — 明示がなければ内容から推定して提案する:

   | 水準 | 内容 | 既定で選ぶ場面 |
   |---|---|---|
   | `none` | 生成のみ（ゲート無し） | 下書き・使い捨て |
   | `readback` | reader で読み戻し観測まで | 一般の納品物 |
   | `conformance(flavour)` | verify で COMPLIANT まで | `tagged` 指定時・アクセシビリティ/長期保存要件・行政/医療向け |

   **`conformance` は「どの flavour で測るか」を伴う**（段階を増やすのではなく引数に持つ）:

   | 要件 | flavour |
   |---|---|
   | `tagged`（アクセシビリティ） | `pdfua-1` |
   | PDF/A-3 添付（電帳法・長期保存） | `pdfa-3b` |
   | 両方 | **`pdfua-1` と `pdfa-3b` の両方**を測る |

   PDF/A-3 と PDF/UA-1 はどちらも ISO 32000-1 基盤なので同時に宣言でき、実測で両立を確認済み
   （同一ファイルで 146/146 と 106/106）。ただし**「片方が通ったから他方も大丈夫」とは言えない**
   ので、**宣言した flavour はすべて測る**。

   **`pdfa-4` は現状取れない**（verify が受け付けない）。要求されたらここで
   「PDF/A-4 は不可。-3b で足りるか」を確認する。

3. **日本語の有無** → 含むなら `fontPath` / 環境変数 `PDF_WRITER_FONT` の確認。
   **`tagged: true` なら日本語が無くても埋め込みフォントが必須**（標準 Helvetica は
   PDF/UA-1 7.21.4.1 で必ず違反 — veraPDF 実測）。`title` も必須（7.1）
4. **電帳法・PDF/A-3 文脈か** → attach_file の `relationship`（Data / Source）を確認し、
   水準は項目 2 の **`pdfa-3b`** を選ぶ。
   **`ensure_pdfa` は writer v0.15.0 から**。それ以前を掴んでいる環境では PDF/A-3b の
   器付けができないので、Phase 1 に進む前に利用者へ伝え、(a) 添付付き PDF
   （PDF/A は名乗らない）で合意する / (b) writer を更新する のどちらかを選ばせる。
   **「無いツールを呼んで失敗してから気づく」を避ける**
5. **入力が署名済みか** → 編集すると署名は必ず無効化される。利用者の明示了解を得てから
   `allowBreakingSignatures: true` を付ける（勝手に付けない）
6. **納品先**（絶対パス）と、**実行ログ**（JSONL）を残すか →
   [references/report-and-log.md](references/report-and-log.md)
7. 再現性が要る（差分比較・キャッシュ）なら `SOURCE_DATE_EPOCH` の利用を提案する

### Phase 1 — 生成・編集（writer）

要件に対応する writer ツールを呼ぶ。複数操作は次の順で直列化する
（順序の根拠は [references/conformance-notes.md](references/conformance-notes.md)）:

```
create_*（tagged はここで決める）
  → tag_form_fields（タグ付き文書にフォームがある場合。labels 必須級）
  → add_bookmarks / add_annotation（alt を渡す）
  → stamp_page_numbers / add_watermark（タグ付きでは自動で Artifact 化される）
  → attach_file（relationship を明示）
  → fill_form（flatten はタグ付きでは使わない）
  → ensure_pdfa（PDF/A-3b 要件のときだけ。**必ず最後**）
```

**`ensure_pdfa` は必ず最後に置く。** とくに **`attach_file` → `ensure_pdfa`** の順を守る:

- この順は実測済み（添付が生き残り veraPDF **146/146 COMPLIANT**）
- **逆順は未検証**。後から `/AF` が増えると `AFRelationship` が `Unspecified` のままで
  PDF/A-3 の要件を外れうる。やむなく逆順になったら**そのつど測り直す**（順序を信用しない）
- `ensure_pdfa` の後に本文・構造を変える操作を足さない。足したら PDF/A の判定はやり直し

**既存 PDF の編集案件では、writer を呼ぶ前に入力を採点しておく**（`identify_conformance` +
`validate_conformance`）。Phase 3 の差分採点の基準値になる。これが無いと
「元から壊れていた」と「自分が壊した」を区別できない（根拠は Phase 3 の差分採点を参照）。

- 各ツールの `warnings` をすべて収集する（lang 推定・グリフ置換・/TU 代用など。
  Phase 5 のレポートに全件載せる）
- エラーが返ったら [references/error-codes.md](references/error-codes.md) の分岐に従う。
  ガード系（SIGNED_PDF / TAGGED_PDF）は**利用者に確認してから**解除フラグで再試行する
- **前段が失敗したら、後段は「失敗」ではなく「スキップ」と記録する。** 直列化された操作は
  前段の出力を入力に取るため、前段の失敗は後段を巻き添えにする。両者を混ぜると
  レポートを読む人が原因を誤る（実測: 生成の失敗が署名の `ENOENT` に化け、
  「署名も失敗」と読めるログになった）

### Phase 2 — 読み戻し（reader・水準 readback 以上）

生成物に対して観測する。**合否は言わない**（それは Phase 3 の仕事）:

1. `read_text` — 意図した本文が抽出できるか（writer の既知リスク: 描画と抽出は独立に壊れる）
2. **タグ付き出力では `extract_structured_text` も呼ぶ** — 論理順（ISO 32000-2 §14.8.2.5 の
   構造木深さ優先走査）で役割付きの本文が返る。`read_text` は座標順なので、
   「H1 は何と書いてあるか」「読み順は意図どおりか」の照合はこちらが向く
3. `inspect_fonts` — フォントが埋め込まれているか（conformance 水準では必須の観測）
4. `inspect_tags` — タグ付き指定時、構造木が意図どおりか（見出し階層・表・リスト）
5. `get_metadata` — title / lang / producer

#### 入力との照合（writer の戻り値を成功の証拠にしない）

**writer が正常終了しても、要求した内容が出力に入っているとは限らない。**
読み戻したテキストを**入力と突き合わせる**こと。目視ではなく、少なくとも次を機械的に確認する:

- 入力に含めた識別子・固有名詞・数値が読み戻しに残っているか
  （実測: `create_markdown_pdf` は Markdown のインライン装飾記号を除去するため、
  `snake_case` の `_` が消えて `identify_conformance` → `identifyconformance` になる。
  **exit 0・warnings なし**で発生する）
- 段落・見出しの件数が入力と一致するか（欠落・重複の検出。実測: `title` と本文の
  先頭見出しが両方 H1 になり `H1` が 2 つになる）

#### 要求した機能の実在確認

「やったつもり」になれる操作は、**出力側でその痕跡を確認する**。writer の終了コードは証拠にならない:

| Phase 1 で呼んだ操作 | 出力側で確認するもの |
|---|---|
| `attach_file` | `inspect_structure` の catalog に `Names`（EmbeddedFiles）と、PDF/A-3 なら `AF` が現れるか（実測確認済み） |
| `add_bookmarks` | `inspect_structure` の catalog に `Outlines` が現れるか |
| `tag_form_fields` | Phase 3 の 7.18.* 違反が消えているか |
| `set_metadata` | `get_metadata` に反映されているか |

### Phase 3 — 品質ゲート（verify・水準 conformance）

1. **`identify_conformance` と `validate_conformance` を必ずペアで呼ぶ。**
   前者は XMP の**自己申告**、後者は**第三者採点**であって、別物である。
   両者が食い違ったら（「PDF/A-2b を宣言しているのに veraPDF で非適合」）、
   それは**適合の刻印を押した非適合ファイルを納品しかけている**ということ。
   差分を必ずレポートに明示する — 採点だけでは「宣言が嘘」を見逃し、
   宣言だけでは実体を見逃す。
   **ただし自分で `ensure_pdfa` / `ensure_tagged` を掛けた場合、この検査は働かない**
   （読む自称が自分で書いたものになる）→ 下の ⚠️ を参照
2. タグ付き出力 → `validate_conformance`（`flavour: "pdfua-1"`、engine は auto）
3. PDF/A 宣言のある入出力 → `validate_conformance`（該当 flavour）
4. **採点結果の `engine` を先に読む。** `verapdf` なら権威的結果、`native` なら
   検査サブセットに過ぎない（`compliant: null` は「適合」ではない）。
   native fallback だった場合、conformance 水準の判定は**保留**として扱う
5. 署名済み入力を（了解の上で）編集した場合 → `verify_integrity` で「署名が無効化された」
   ことを**意図どおり**と確認し、レポートに明記する

#### 🔴 「宣言を作るツール」を使ったら、検証は省略できない

**`ensure_pdfa` と `ensure_tagged` は「宣言」を書く道具で、「適合」は作らない。**

- **`ensure_pdfa` を呼んだら `validate_conformance(flavour: "pdfa-3b")` は必須。**
  この 2 つは**不可分**。`ensure_pdfa` 単発で Phase 5 に進んではいけない。
  **水準が `none` / `readback` でも、`ensure_pdfa` を使ったなら PDF/A の検証だけは通す** —
  さもなくば「PDF/A-3b と名乗るが誰も検査していないファイル」を納品することになる
- 同じ理由で **`ensure_tagged` を呼んだら `pdfua-1` を測る**
- **`ensure_pdfa` は必ず warnings を返す**（「CLAIMS PDF/A-3b … conformance was NOT checked」）。
  これは異常ではなく設計。**レポートに転記する**。warnings が無ければ writer 側の欠陥を疑う

> **⚠️ 上の 1.（`identify_conformance` とのペア呼び出し）は、自分で `ensure_pdfa` を掛けた
> ファイルには「宣言が嘘か」の検査として働かない** — 読み取る自称は**自分が書いたもの**だから。
> `identify_conformance` を合否の根拠にしてはいけない。合否は必ず `validate_conformance`（veraPDF）で採る。
> ペア呼び出しが意味を持つのは、**受け取った PDF を調べるとき**（Phase 0 / UC-6）である。

#### 差分採点（既存 PDF を編集した場合は必須）

**単発の合否では、その違反を誰が作ったか特定できない。** Phase 1 で取った入力の基準値と
出力の採点を突き合わせ、`入力の違反 → 出力の違反` の形でレポートする:

| 差分 | 意味 |
|---|---|
| 違反数が増えた | **その操作が壊した。** 増えた clause を名指しで報告する |
| 違反数が同じ | 元からの違反。編集の責任ではない（が納品可否には効く） |
| 違反数が減った | 修復が効いている |

実測で確認された増加パターン（他実装での観測だが、同型の事故は自分の writer でも起こりうる）:
署名の増分更新が PDF/A の字句規則（`obj` / `endobj` 前後の EOL）を破る、
ページ結合でカタログの XMP と OutputIntent が失われ DeviceRGB 違反が噴出する。
どちらも**出力を単独で見ると「元から壊れていた」と区別がつかない**。

結果の扱い:

| verify の結果 | 行動 |
|---|---|
| COMPLIANT（veraPDF） | 納品へ。規則数（例 106/106）をレポートに記載 |
| 違反あり・writer で修正可能 | Phase 4 の修正ループへ |
| 違反あり・writer の能力外 | 人手レビュー要請。Tier C 未実装（本文編集・タグ木保守）であることを明記 |
| `compliant: null`（native エンジン） | 「検査サブセットで違反なし＝必要条件のみ」と明記。veraPDF 導入を提案 |
| `ensure_pdfa` / `ensure_tagged` を使ったが対応する flavour を測っていない | **納品しない。** Phase 3 に戻る（宣言と検証は不可分） |
| verify 未接続 | conformance 水準の案件は**中止**（水準を readback に下げる合意が取れれば続行） |

### Phase 4 — 修正ループ（上限 3 回）

違反 clause を writer の操作に対応付けて修正・再検証する。対応表は
[references/conformance-notes.md](references/conformance-notes.md) にある（例:
7.18.4-1 / 7.18.3-1 / 7.18.1-3 → `tag_form_fields`、7.21.4.1 → 埋め込みフォントで再生成）。

- **上限 3 回**。超えたら停止し、残違反リスト + spec 根拠（pdf-spec-mcp があれば
  `get_section` で条項本文）を添えて人手レビューに引き渡す
- 同じ違反が 2 回続いたら、修正が効いていない — 別の対応を探すか即座に人手へ

### Phase 5 — 納品と記録

1. **Publish Report** を出力する（テンプレートは
   [references/report-and-log.md](references/report-and-log.md)）。最低限:
   成果物パス・実行した操作列（**スキップした段は「スキップ」と明記**）・
   読み戻しの観測（**入力との照合結果・要求機能の実在確認を含む**）・
   宣言（identify）と採点（validate）の**両方と、その差分**・
   verify の判定（**エンジン名**と規則数）・**編集案件は入力採点との差分**・
   warnings 全件・ループ回数
2. Phase 0 で合意していれば**実行ログ（JSONL 1 行）**を追記する。
   これが read-write-verify ループの学習データ（verify の verdict がラベル）になる

### PDF/A の判定を書くときの言い方（T2）

**PDF/A は T2** = ISO 19005 が family のコーパスに無いので**条文を引けない**。
**PDF/UA-1（ISO 14289-1）は T1** で条文を引ける。**同じレポートの中で書き方が変わる**ことに注意。

| 書いてよい | 書いてはいけない |
|---|---|
| 「**veraPDF が** PDF/A-3b **COMPLIANT と判定**（146/146）」 | 「ISO 19005-3 **準拠**」 |
| 「veraPDF の指摘: `6.1.3-1` /ID 欠落」 | 「ISO 19005-3 §6.1.3 **に違反**」（条文を確認していない） |
| 「PDF/UA-1 7.1 が文書タイトルを要求する（条文）」 | — （PDF/UA は T1 なので断定してよい） |

**是正指示だけは条文まで降ろせる**（T2 → T1 の昇格ルート）。
例: `/ID` 欠落 → 直し方は **ISO 32000-1 §14.4** を引いて示せる。
ただし §14.4 は `/ID` を「optional but should」としており、**「無ければならない」義務は PDF/A 側にしかない**
— 条文で言えるのは「値の作り方」までである。ここを混ぜると T2 の義務を T1 の断定に見せてしまう。

## やらないこと

- 合否の自前判定（verify の結果以外を根拠に「準拠」と言うこと）
- verify 未接続での conformance 水準の納品
- **`ensure_pdfa` / `ensure_tagged` を使ったのに、対応する flavour を測らずに納品すること**
  （宣言だけ書いて検査しない = 嘘の刻印を押した納品）
- **PDF/A の判定を「ISO 19005 準拠」と書くこと**（T2。判定者は veraPDF）
- **writer の正常終了を「要求どおり出力された」の証拠にすること**（読み戻して確認する）
- **`identify_conformance` を省いて `validate_conformance` だけで済ませること**
  （宣言と実体の乖離を見逃す）— **呼ぶこと自体は必須**
- **ただし `identify_conformance` の結果を合否の根拠にすること**（自称を読むだけ。とくに自分で
  `ensure_pdfa` を掛けた後は、自分が書いた宣言を読み返しているに過ぎない）
- **編集案件で入力を採点せずに出力だけ採点すること**（壊した責任の所在が消える）
- 利用者の了解なしの `allowBreakingSignatures` / `allowBreakingTags`
- `outputPath` 省略での大きな PDF 生成（base64 溢れ）
- 内容（文章そのもの）の品質保証 — このパイプラインが保証するのは構造・準拠性・
  抽出可能性であって、文章の正しさではない
