---
name: pdf-publish
description: PDF の生成・編集から納品までを品質ゲート付きで編成する出力パイプライン。pdf-writer-mcp で書き、pdf-reader-mcp で読み戻し、pdf-verify-mcp で機械採点（veraPDF）する write → read-back → verify ループを回し、Publish Report 付きで納品する。ユーザーが「PDF にして納品」「PDF/UA で作って」「アクセシブルな PDF」「タグ付き PDF の生成」「電帳法対応の PDF」「データを添付した PDF/A-3」「PDF/A-4 で作って」「PDF/UA-2 で作って」「品質保証付きで PDF を作って」「PDF を作って検証まで」「フォームを PDF/UA 準拠に」などに言及したら、単発の writer ツール呼び出しで済ませず必ずこの Skill を使う。pdf-trust（受入監査）の対になる送り出し側。
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
4. writer・reader・verify のエラーは同じ形で構造化されている（`code` / `retryable` /
   `next_actions`。family の規約 §2.3）。文字列を読んで推測せず、`code` で分岐し
   `next_actions` の指示に従う。**読み戻しや採点のエラーを「未実施」と「所見」に分ける**
   （`INVALID_PDF` は出力が壊れているという所見）→
   [references/error-codes.md](references/error-codes.md)
5. **「宣言」と「適合」を混同しない。** `ensure_pdfa` / `ensure_tagged` は XMP に
   `pdfaid` / `pdfuaid` を書く = **その文書に「規格に沿っています」と名乗らせる**道具であって、
   適合させる道具ではない。**宣言を書いたら必ず対応する flavour を verify で測る**
   （測らないなら宣言も書かない）。「宣言 / 適合 / 検証は別物」

## 前提 MCP

| MCP | 必須/任意 | 役割 |
|---|---|---|
| pdf-writer-mcp (v0.8.0+ / **v0.28.1+ 推奨**) | **必須** | 生成（Tier 0）・編集（Tier A/B）・PDF/UA 修復のすべて。**PDF/A-3b の器付け（`ensure_pdfa`）は v0.15.0 から・PDF/A-4 / -4f と PDF 2.0 出力は v0.16.0 から・`declarationRisks`（測ると落ちると分かっている宣言）は v0.17.0 から・`ensure_tagged` が成功時にも CLAIMS … NOT checked の warning を返すのは v0.21.1 から・生成 3 ツールの `drawTitle` は v0.23.0 から・PDF/UA を宣言した入力への `ensure_pdfa`（-4 / -4f）の拒否と、応答の `previousDeclaration`（`wasDeclared` を置き換えた）は v0.24.0 から・標準フォントで WinAnsiEncoding の字（`“` `—` `€` など）を受け付け、`FONT_REQUIRED`・`MISSING_GLYPH` が原因の字を `detail.characters` に返すのは v0.26.0 から・`tagged: true` と `pdfVersion: "2.0"` で PDF/UA-2 を書き、PDF/UA-2 の入力に `pdfa-4` を通すのは v0.27.0 から・全ツールの結果の `output`（出力の版と宣言）と `merge_pdfs` の `pdfVersion`、PDF 2.0 の入力への `pdfa-3b` の拒否は v0.28.0 から・PDF/UA-2 の文書に足す注釈とウィジェットの `/Contents` を条文に合わせたのは v0.28.1 から** |
| pdf-verify-mcp (**v0.35.0+ 推奨**) | 品質ゲート案件では**必須** | identify_conformance / validate_conformance（veraPDF 委譲）/ verify_integrity。v0.20.0 から報告の先頭に `scope`（判定の射程）が入る。**v0.29.0 から `violations[]` は 200 件で切られる**（`violationsTruncated: { returned, total }` が付く）。違反の**数**は配列の長さではなく `failedRules` で取る（下記「差分採点」）。**v0.35.0 から、鍵を導けない暗号化文書の PDF/A 判定はエンジンにかかわらず `ENCRYPTED_PDF`**（v0.34.x までは veraPDF の無い環境で、読めなかった XMP・OutputIntent を違反として返していた） |
| pdf-reader-mcp (**v0.20.0+ 推奨**) | 推奨 | 読み戻し（テキスト抽出・論理順抽出・フォント・タグ・メタデータの観測）。**v0.14.0 から `read_text` / `extract_structured_text` の応答に `scope`（どこまで読んだか）が入り、行われなかった読みの項目は `null` になる** —— 読み戻せたかどうかはここで決まる。**v0.20.0 から `get_metadata` が XMP も読む**（PDF 2.0 の出力の題名・日付） |
| pdf-spec-mcp | 任意 | 違反時の ISO 32000 / 14289 条項の根拠引用。**ISO 19005（PDF/A）は収録外**なので PDF/A の条文は引けない（T2） |

pdf-writer-mcp が未接続なら成立しない。`npx @shuji-bonji/pdf-writer-mcp@latest` の接続を
案内して停止する。品質ゲート水準 `conformance`（後述）の案件で pdf-verify-mcp が無い場合も
中止する — ゲート無しで「準拠」を名乗るのは虚偽になる。reader が無い場合は縮退動作し、
レポートに「読み戻し: 未実施（ツール未接続）」と明記する。

**`ensure_pdfa` / `ensure_tagged` を使う案件は、水準にかかわらず pdf-verify-mcp が必須。**
これらは文書に「規格に沿っています」と名乗らせるため、検査できないまま使うと
**嘘の刻印を押したファイルを納品する**ことになる。verify が無いなら宣言も書かない。

### 🔴 読み戻しで `scope.reconstructed` が true なら、それはこちらの欠陥

verify v0.20.0+ は全ツールの報告の先頭に `scope` を返す。**自分が今書いた
ファイルに対して `reconstructed: true` が返ったら、書いた相互参照表が読めて
いない** —— verify が本文の `N G obj` を数え上げて表を作り直したという意味で、
これは受け取った文書の性質ではなく **こちらの出力の欠陥**である。
`chainStop.kind` が `complete` でない場合も同じ。

合否の前に見ること。veraPDF が COMPLIANT を返しても、それは
**組み直した表の上での判定**でしかない。

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
   | `tagged`（アクセシビリティ）・PDF 1.7（既定） | `pdfua-1` |
   | **`tagged` と `pdfVersion: "2.0"`（PDF/UA-2。writer v0.27.0 から）** | **`pdfua-2`** |
   | PDF/A-3 添付（請求書とデータの同梱・長期保存） | `pdfa-3b` |
   | **PDF/A-4（PDF 2.0 基盤の長期保存）・添付なし** | **`pdfa-4`** |
   | **PDF/A-4 で添付あり（CSV/JSON の同梱）** | **`pdfa-4f`** |
   | 両方 | **PDF/UA 側と PDF/A 側の両方**を測る（組み合わせは下の 2 つだけ） |

   **測る flavour は、writer の結果の `output.declares` で確かめる**（writer v0.28.0 から。下の「測る flavour の決め方」）。

   PDF/A-3 と PDF/UA-1 はどちらも ISO 32000-1 基盤なので同時に宣言でき、実測で両立を確認済み
   （同一ファイルで 146/146 と 106/106）。ただし**「片方が通ったから他方も大丈夫」とは言えない**
   ので、**宣言した flavour はすべて測る**。

   **PDF/A-4 は writer v0.16.0 / verify v0.11.0 から取れる**（実測: `pdfa-4` / `pdfa-4f` とも
   veraPDF **109/109 COMPLIANT**）。選ぶときの判断は 2 つだけ:

   - **添付があるなら必ず `pdfa-4f`。** 素の `pdfa-4` は**添付ファイル自身が PDF/A であること**を
     要求する（veraPDF `ISO 19005-4:2020 6.9-3`）ので、CSV や JSON を同梱した瞬間に非適合になる。
     **添付を持つ文書で PDF/A-4 を選ぶなら、事実上つねに `-4f`**
   - **PDF/UA と PDF/A の組み合わせは版で決まる。** 両立するのは次の 2 つだけ:

     | 版 | PDF/UA | PDF/A | 作り方 | 実測（veraPDF 1.30.0） |
     |---|---|---|---|---|
     | PDF 1.7 | PDF/UA-1（`pdfua-1`） | PDF/A-3b（`pdfa-3b`） | `create_*(tagged)` → `ensure_pdfa` | 106/106・146/146 |
     | PDF 2.0 | PDF/UA-2（`pdfua-2`） | PDF/A-4（`pdfa-4` / `-4f`） | `create_*(tagged, pdfVersion: "2.0")` → `ensure_pdfa(flavour: "pdfa-4")` | 1,728/1,728・109/109 |

     PDF/UA-1 は版を 1.0〜1.7 とし（ISO 14289-1 §6.1）、PDF/UA-2 は 2.0 とする（ISO 14289-2 §6.1）。
     PDF/A-3 はヘッダーを `%PDF-1.n` に、PDF/A-4 は `%PDF-2.n` に限る。版をまたぐ組み合わせは
     writer が `INVALID_ARGUMENT` で断る（PDF/UA-1 の入力に -4 は v0.24.0 から、PDF 2.0 の入力に
     `pdfa-3b` は v0.27.0（PDF/UA-2）・v0.28.0（タグ無しも）から）。
     **アクセシビリティと PDF/A-4 の両方を求められたら、新規生成なら PDF 2.0 で作る。** 既存の
     PDF/UA-1 文書なら「-3b + PDF/UA-1」に倒すか、PDF 2.0 で作り直すかを利用者に選ばせる
     （writer は既存の文書の版を上げて PDF/UA-1 を PDF/UA-2 に書き換えることはしない）

   **-3b と -4 のどちらを勧めるか**: 実需が「請求書とデータを 1 つのファイルで渡す器」なら **-3b で足りている**
   （UC-4 は -3b で充足済み）。-4 を選ぶのは「PDF 2.0 基盤で残したい」「-4 を指定された」ときに限る。

3. **標準フォントで書けない字（日本語など）の有無** → 含むなら `fontPath` / 環境変数 `PDF_WRITER_FONT` の確認。
   標準 Helvetica で書けるのは WinAnsiEncoding の 218 字（ASCII・Latin-1 の文字と記号・`“` `”` `–` `—` `€` `…` `™` など）。
   writer v0.26.0 から、それ以外の U+0100 以上の字だけが `FONT_REQUIRED` になる（v0.25.x までは U+0100 以上の字すべて）。
   **`tagged: true` なら日本語が無くても埋め込みフォントが必須**（標準 Helvetica は
   PDF/UA-1 7.21.4.1・PDF/UA-2 8.4.5.5.1 で必ず違反 — veraPDF 実測）。`title` も必須（PDF/UA-1 7.1）
   - **本文が `title` と同じ見出し（`# 題名`）で始まるなら、`drawTitle: false` を渡す**（writer v0.23.0 から）。
     既定（`drawTitle` を渡さない）では writer が `title` を本文の先頭に描き、構造木の H1 になるので、
     本文の `#` と合わせて **H1 が 2 つ並ぶ**。`drawTitle: false` なら `title` はメタデータ（`dc:title`）にだけ入り、
     本文の最初の見出しが H1 になる。本文に題名の見出しが無いときは、既定のまま題名を描かせる
   - writer が v0.23.0 より前なら `drawTitle` が無いので、本文から先頭の `# 題名` を外して `title` だけで渡す
4. **PDF/A と添付の文脈か** → attach_file の `relationship`（Data / Source）を確認し、
   水準は項目 2 の **`pdfa-3b`**（または `pdfa-4f`）を選ぶ。
   **`ensure_pdfa` は writer v0.15.0 から・`flavour`（-4 / -4f）は v0.16.0 から**。
   それ以前を掴んでいる環境では器付けができない / -4 を選べないので、Phase 1 に進む前に
   利用者へ伝え、(a) 添付付き PDF（PDF/A は名乗らない）で合意する / (b) -3b に倒す /
   (c) writer を更新する のいずれかを選ばせる。
   **「無いツールを呼んで失敗してから気づく」を避ける**
   - **「電帳法対応」と言われても、PDF/A と添付は電帳法の要件ではない。** 電子取引のデータの保存で
     電帳法が求めるのは、タイムスタンプ・訂正削除の記録か防止ができるシステム・訂正削除の防止に関する
     事務処理規程のいずれかの措置と、同項が準用する見読性・検索機能の要件（電子帳簿保存法施行規則 4 条 1 項）で、
     ファイル形式は決めていない。この Skill が作れるのは PDF/A と添付の器で、その措置ではない
   - なので「電帳法対応の PDF」と頼まれたら、(a) PDF/A も添付も法令の要件ではないことを伝え、
     (b) 求めているのが「PDF/A で残したい」「データを同梱したい」のどちらか（両方か）を確かめてから、
     項目 2 の水準を決める。Publish Report には「電帳法に対応した」と書かない
     （書けるのは「PDF/A-3b を宣言し、veraPDF が適合と判定した」まで）。電帳法の要件の確認は
     houki 系の MCP（houki-research Skill）の仕事
5. **入力が署名済みか** → 編集すると署名は必ず無効化される。利用者の明示了解を得てから
   `allowBreakingSignatures: true` を付ける（勝手に付けない）
6. **納品先**（絶対パス）と、**実行ログ**（JSONL）を残すか →
   [references/report-and-log.md](references/report-and-log.md)
7. 再現性が要る（差分比較・キャッシュ）なら `SOURCE_DATE_EPOCH` の利用を提案する

### Phase 1 — 生成・編集（writer）

要件に対応する writer ツールを呼ぶ。複数操作は次の順で直列化する
（順序の根拠は [references/conformance-notes.md](references/conformance-notes.md)）:

```
create_*（tagged と pdfVersion はここで決める）
  （merge_pdfs / split_pdf などのページ操作は構造木と宣言を運ばない。タグ付き・PDF/A の成果物に
   挟まない。挟んだら output.declares が {} になる）
  → tag_form_fields（タグ付き文書にフォームがある場合。labels 必須級）
  → add_bookmarks / add_annotation（alt を渡す）
  → stamp_page_numbers / add_watermark（タグ付きでは自動で Artifact 化される）
  → attach_file（relationship を明示）
  → fill_form（flatten はタグ付きでは使わない）
  → ensure_pdfa（PDF/A 要件のときだけ。flavour = -3b / -4 / -4f。**必ず最後**）
```

**`ensure_pdfa` は必ず最後に置く。** とくに **`attach_file` → `ensure_pdfa`** の順を守る:

- この順は実測済み（添付が生き残り veraPDF **146/146 COMPLIANT**）
- **逆順は未検証**。後から `/AF` が増えると `AFRelationship` が `Unspecified` のままで
  PDF/A-3 の要件を外れうる。やむなく逆順になったら**そのつど測り直す**（順序を信用しない）
- `ensure_pdfa` の後に本文・構造を変える操作を足さない。足したら PDF/A の判定はやり直し
- **`ensure_pdfa` が `INVALID_ARGUMENT` を返し、`next_actions[0].action` が `retry_with_flavour` なら**、
  入力の版と求めた flavour が両立しない。理由は `example.flavour` で分かれる（エラーの文に入力の版と宣言が入る。v0.28.0〜）:
  - `example.flavour` が `pdfa-3b` — 入力が PDF/UA-1 を宣言していて、-4（PDF 2.0）と両立しない（v0.24.0〜）
  - `example.flavour` が `pdfa-4` — 入力が PDF 2.0 で、-3b（`%PDF-1.n`）と両立しない（PDF/UA-2 は v0.27.0〜、タグ無しは v0.28.0〜）

  **どちらも `example` に黙って切り替えない。** Phase 0 の組み合わせの表に戻り、利用者に選ばせる
- `ensure_pdfa` の応答の `previousDeclaration`（`{ part, conformance }` か `null`。v0.24.0 で
  `wasDeclared` を置き換えた）で、入力がもともと何を名乗っていたかを Publish Report に書く。
  `ensure_tagged` は、足したもの（`addedRequirements`）と書き直したもの（`replacedRequirements`）を
  分けて返し、何も変えなければ `requirementsChanged: false` になる（v0.24.0〜。出力は書き出される）

**既存 PDF の編集案件では、writer を呼ぶ前に入力を採点しておく**（`identify_conformance` +
`validate_conformance`）。Phase 3 の差分採点の基準値になる。これが無いと
「元から壊れていた」と「自分が壊した」を区別できない（根拠は Phase 3 の差分採点を参照）。

- 各ツールの `warnings` をすべて収集する（lang 推定・グリフ置換・/TU 代用・
  PDF/UA-2 の文書に `/NS` の無い構造要素を足したことなど。Phase 5 のレポートに全件載せる）
- **各ツールの結果の `output`（`{ pdfVersion, declares }`。writer v0.28.0〜）を手順ごとに記録する。**
  書いたファイルから読み戻した版と宣言で、引数から推測した値ではない。Phase 3 で測る flavour を
  ここから決める。`merge_pdfs` などページ操作の後で `declares` が `{}` になっていたら、構造木と
  宣言が運ばれなかったということ（タグ付けと `ensure_pdfa` をやり直す）
- **`merge_pdfs` が `INVALID_ARGUMENT`（`next_actions[0].example` が `{ pdfVersion: "2.0" }`）を返したら、**
  入力に PDF 2.0 と 1.x が混ざっている（v0.28.0〜。writer は版を黙って上げない）。`pdfVersion: "2.0"` は
  明示した変換で、Info の題名などは XMP に移る。**PDF/A-3b を目標にしている案件では 2.0 にすると
  -3b を名乗れなくなるので、黙って従わない。** Phase 0 の組み合わせの表に照らして利用者に選ばせる
- **PDF/UA-2 の文書に `add_annotation` で `alt` を渡すと、`/Contents` にも同じ値が入る**（ISO 14289-2
  §8.9.4.2 が同じ値を求める。v0.28.1〜）。`contents` と `alt` を違う値で渡すと `INVALID_ARGUMENT` になる
  ので、PDF/UA-2 では 1 つの文を両方に使う
- エラーが返ったら [references/error-codes.md](references/error-codes.md) の分岐に従う。
  ガード系（SIGNED_PDF / TAGGED_PDF）は**利用者に確認してから**解除フラグで再試行する
- **前段が失敗したら、後段は「失敗」ではなく「スキップ」と記録する。** 直列化された操作は
  前段の出力を入力に取るため、前段の失敗は後段を巻き添えにする。両者を混ぜると
  レポートを読む人が原因を誤る（実測: 生成の失敗が署名の `ENOENT` に化け、
  「署名も失敗」と読めるログになった）

### Phase 2 — 読み戻し（reader・水準 readback 以上）

生成物に対して観測する。**合否は言わない**（それは Phase 3 の仕事）:

🔴 **reader v0.14.0 以降は、まず応答の `scope` を読む。** `read_text` と
`extract_structured_text` は 2 つの別々の読み（ページから文字を取り出すこと・
その文字が Unicode に変換できるかの観測）から作られ、どちらも単独で失敗する。
`scope.textExtraction` が `failed` なら **`pages[].text` は `null`** であって
空文字ではない。**`null` を「本文が空だった」として読み戻しに数えない** ——
それは「読み戻せなかった」であり、下の照合はどれも成立していない。
その場合は水準 `readback` を満たしていないので、Publish Report にそう書く
（ゲートを通っていないのに「読み戻し済み」と名乗らない）。

1. `read_text` — 意図した本文が抽出できるか（writer の既知リスク: 描画と抽出は独立に壊れる）
2. **タグ付き出力では `extract_structured_text` も呼ぶ** — 論理順（ISO 32000-2 §14.8.2.5 の
   構造木深さ優先走査）で役割付きの本文が返る。`read_text` は座標順なので、
   「H1 は何と書いてあるか」「読み順は意図どおりか」の照合はこちらが向く
3. `inspect_fonts` — フォントが埋め込まれているか（conformance 水準では必須の観測）
4. `inspect_tags` — タグ付き指定時、構造木が意図どおりか（見出し階層・表・リスト）
5. `get_metadata` — title / lang / producer。**reader 0.20.0 から XMP も読む**（pdf-reader-mcp#42）。
   PDF 2.0 の出力（`pdfa-4`・`-4f`、PDF 2.0 で生成）は Info 辞書を持たないことがあり、
   reader 0.19.x 以前では題名・日付が `null` になっていた。`metadataSources.title` で
   題名の出どころ（`info` / `xmp`）を、`metadataConflicts` で 2 つの値の食い違いを確かめる。
   `xmpMetadata.status` が `unreadable` のときの `null` を「題名が無い」と書かない

#### 入力との照合（writer の戻り値を成功の証拠にしない）

**writer が正常終了しても、要求した内容が出力に入っているとは限らない。**
読み戻したテキストを**入力と突き合わせる**こと。目視ではなく、少なくとも次を機械的に確認する:

- 入力に含めた識別子・固有名詞・数値が読み戻しに残っているか
  （実測: `create_markdown_pdf` は Markdown のインライン装飾記号を除去するため、
  `snake_case` の `_` が消えて `identify_conformance` → `identifyconformance` になる。
  **exit 0・warnings なし**で発生する）
- 段落・見出しの件数が入力と一致するか（欠落・重複の検出。実測: `drawTitle` を渡さずに、
  本文が `# 題名` で始まる Markdown を渡すと、`title` と本文の先頭見出しが両方 H1 になり
  `H1` が 2 つになる。Phase 0 の項目 3 のとおり `drawTitle: false` で防ぐ）

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
2. タグ付き出力 → `validate_conformance`（`output.declares.pdfua` が 1 なら `pdfua-1`、2 なら `pdfua-2`。engine は auto）
3. PDF/A 宣言のある入出力 → `validate_conformance`（`output.declares.pdfa` に対応する flavour）
4. **採点結果の `engine` を先に読む。** `verapdf` なら権威的結果、`native` なら
   検査サブセットに過ぎない（`compliant: null` は「適合」ではない）。
   native fallback だった場合、conformance 水準の判定は**保留**として扱う
5. 署名済み入力を（了解の上で）編集した場合 → `verify_integrity` で「署名が無効化された」
   ことを**意図どおり**と確認し、レポートに明記する
6. **鍵を導けない暗号化文書**（利用者パスワード付き）の PDF/A 判定は、verify v0.35.0 から
   **エンジンにかかわらず `ENCRYPTED_PDF`**（`retryable: true`。`next_actions` が `password` を付けた
   `validate_conformance` の呼び直しを案内する）。違反の一覧は返らない。パスワードを知っていれば
   渡して測る。知らなければ PDF/A の判定は**未実施**と書く（「違反あり」とも「適合」とも書かない）。
   PDF/UA の判定はエラーにならず、構造の規則を「見ていない」と返す（`ua-no-encryption-barrier`
   だけは暗号化辞書から判定される）

#### 測る flavour の決め方（writer v0.28.0〜）

最後に書いた writer の結果の `output.declares` から決める。**宣言したものはすべて測る**:

| `output.declares` | 測る flavour |
|---|---|
| `pdfua: 1` | `pdfua-1` |
| `pdfua: 2` | `pdfua-2` |
| `pdfa: "3b"` | `pdfa-3b` |
| `pdfa: "4"` | `pdfa-4` |
| `pdfa: "4f"` | `pdfa-4f` |
| `{}` | 宣言は無い。PDF/UA・PDF/A を要件にしていたなら、どこかの手順で宣言が落ちている（Phase 1 に戻る） |

`declares` は**自己申告**で、writer は適合を確かめていない。合否は `validate_conformance` で採る。
writer v0.27.x 以前で `output` が無いときは、`identify_conformance` の自己申告から決める。

#### 🔴 「宣言を作るツール」を使ったら、検証は省略できない

**`ensure_pdfa` と `ensure_tagged` は「宣言」を書く道具で、「適合」は作らない。**

- **`ensure_pdfa` を呼んだら `validate_conformance` は必須**（flavour は `ensure_pdfa` に渡したものと同じ文字列）。
  この 2 つは**不可分**。`ensure_pdfa` 単発で Phase 5 に進んではいけない。
  **水準が `none` / `readback` でも、`ensure_pdfa` を使ったなら PDF/A の検証だけは通す** —
  さもなくば「PDF/A-3b と名乗るが誰も検査していないファイル」を納品することになる
- 同じ理由で **`ensure_tagged` を呼んだら PDF/UA を測る**。PDF 1.x の入力なら `pdfua-1`、PDF 2.0 の入力なら
  `pdfua-2`（writer v0.27.0 から、PDF 2.0 の入力には PDF/UA-2 を宣言する。`output.declares.pdfua` で確かめる）
- **`ensure_pdfa` と `ensure_tagged` は必ず warnings を返す**（「CLAIMS PDF/A-3b … conformance was NOT checked」。-4 なら「CLAIMS PDF/A-4」。`ensure_tagged` は「CLAIMS PDF/UA-1 (pdfuaid:part=1) … NOT checked」、PDF 2.0 の入力では「CLAIMS PDF/UA-2 (pdfuaid:part=2, rev=2024) … NOT checked」）。
  これは異常ではなく設計。**レポートに転記する**。warnings が無ければ writer 側の欠陥を疑う（`ensure_tagged` は writer v0.21.0 まで、他に警告が無いとこの 1 行を返さなかった）

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

違反数は **`failedRules`**（規則の数）で比べる。`violations[]` の長さで数えてはいけない —
v0.29.0 から一覧は 200 件で切られ（`violationsTruncated`）、200 件を超える文書では差が消える。
増えた規則を名指しするときは、`violationsTruncated` が立っていたら「一覧は全 <total> 件のうち
<returned> 件」と添える。

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
**PDF/UA-1（ISO 14289-1）と PDF/UA-2（ISO 14289-2）は T1** で条文を引ける（pdf-spec-mcp の `pdfua1`・`pdfua2`）。**同じレポートの中で書き方が変わる**ことに注意。

| 書いてよい | 書いてはいけない |
|---|---|
| 「**veraPDF が** PDF/A-3b **COMPLIANT と判定**（146/146）」 | 「ISO 19005-3 **準拠**」 |
| 「**veraPDF が** PDF/A-4f **COMPLIANT と判定**（109/109）」 | 「ISO 19005-4 **準拠**」 |
| 「veraPDF の指摘: `6.1.3-1` /ID 欠落」 | 「ISO 19005-3 §6.1.3 **に違反**」（条文を確認していない） |
| 「PDF/UA-1 7.1 が文書タイトルを要求する（条文）」 | — （PDF/UA は T1 なので断定してよい） |
| 「PDF/UA-2 §8.9.4.2 が注釈の Contents と Alt を同じ値にすることを要求する（条文）」 | — （同上） |

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
