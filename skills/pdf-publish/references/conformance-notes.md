# 準拠の実測ノートと違反→修正対応表（PDF/UA-1 + PDF/A-3b + PDF/A-4）

すべて veraPDF での実測に基づく（**PDF/UA-1** = `--flavour ua1`・106 規則 /
**PDF/A-3b** = `--flavour 3b`・146 規則 / **PDF/A-4** = `--flavour 4`・**109 規則**、
**PDF/A-4f** = `--flavour 4f`・109 規則）。
判定は必ず pdf-verify-mcp の `validate_conformance` で取ること。

> **PDF/UA-1（ISO 14289-1）は T1** = 条文を引ける。**PDF/A（ISO 19005）は T2** = 条文が家に無く、
> 言えるのは「veraPDF はこう判定した」まで。**同じ文書の中で書き方が変わる**ことに注意。

## 違反 clause → writer 操作の対応表（Phase 4 修正ループ用）

| veraPDF の違反 | 原因 | writer での修正 |
|---|---|---|
| 7.18.4-1（Widget が Form タグ外） | タグ付き文書に未タグのフォーム | `tag_form_fields` |
| 7.18.3-1（/Tabs が S でない） | 注釈/Widget のあるページ | `tag_form_fields`（Widget 由来）/ `add_annotation` は自動対応済み |
| 7.18.1-3（フィールドに /TU 無し） | 代替名の欠落 | `tag_form_fields` の `labels` |
| 7.21.4.1-1（フォント非埋め込み） | 標準フォント（Helvetica）使用 | `fontPath` を指定して**再生成**（後付け埋め込みは不可） |
| 7.21.5-1（グリフ幅の不整合） | 外部ツールの壊れたサブセット | writer で再生成（writer は harfbuzz 事前サブセットで安全。ADR-7/8） |
| 7.1-3（タグ外コンテンツ） | flatten の焼き込み、外部ツールの描画 | flatten を避ける。writer の watermark / stamp / annotation は自動で Artifact 化される |
| 7.1（タイトル無し） | title 未指定 | `tagged: true` は `title` 必須（writer がエラーで止める） |
| 7.2（/Lang） | 言語宣言の欠落・誤り | create 時に `lang` を明示（省略時は推定 + warnings） |

**writer の能力外**（人手レビュー・Tier C 待ち）: 既存タグ木の再構成、読み順の修正、
本文コンテンツの再タグ付け、画像への /Alt 後付け（B-4 実装まで）。

### PDF/A-3b（`pdfa-3b`）の違反 → writer 操作

> **ルール ID は veraPDF のもので、ISO 19005-3 の条文は引けない（T2）。**
> 「是正の仕方」だけは ISO 32000-1 を引いて示せる場合がある（下表の「条文（T1）」列）。

| veraPDF の違反 | 原因 | writer での修正 | 条文（T1） |
|---|---|---|---|
| `6.1.3-1`（trailer に /ID 無し） | create 系は /ID を書かない | **`ensure_pdfa`** | 値の作り方は ISO 32000-1 §14.4。ただし §14.4 は "optional but should"、Table 15（§7.5.5）も「`Encrypt` があれば Required、なければ optional」— **非暗号化文書で無条件に必須にしているのは PDF/A 側（T2）** |
| `6.2.4.3-2`（DeviceRGB に DefaultRGB も OutputIntent も無い） | 色空間が device-dependent のまま | **`ensure_pdfa`**（sRGB の OutputIntent を catalog に追加。ICC を生成して埋め込む） | OutputIntent 辞書は ISO 32000-1 Table 365 / §14.11.5 |
| `6.6.4-1`（PDF/A Identification が無い） | XMP に pdfaid が無い | **`ensure_pdfa`** | — |
| フォント非埋め込み | 標準フォント（Helvetica）使用 | `fontPath` を指定して**再生成**（`ensure_pdfa` では直らない） | ISO 32000-1 §9.9 |
| 暗号化されている | 入力が暗号化 PDF | writer は復号を持たない。復号済みファイルを用意する | — |

**`ensure_pdfa` で直るのは上 3 件だけ。** フォント・暗号化・JavaScript・LZW は直らないので、
**`ensure_pdfa` を掛けても COMPLIANT にならないケースは普通にある**（実測で通ったのは
writer 自身が作った PDF だったから = フォント埋め込み済み・暗号化なし）。

> **writer v0.17.0 からは、その一部を検証の前に知らせてくれる。**
> `ensure_pdfa` の結果に `declarationRisks` があれば、**書いた宣言は測ると落ちると
> 既に分かっている**。現状の唯一のコードは `FONT_NOT_EMBEDDED`（`affected` に該当フォント名）。
> **これは veraPDF を回す前に出るので、無駄な検証を 1 往復省ける**うえ、
> Publish Report に「なぜ落ちたか」を writer 由来の事実として書ける。
> ⚠️ **`declarationRisks` が空でも適合の証拠にはならない** — writer が観測できる不適合は
> フォント埋め込みだけで、暗号化・JavaScript・LZW は依然として検証でしか分からない。

**実測（2026-07-25）**: 電帳法検体（請求書 + CSV 添付）に `attach_file` → `ensure_pdfa` を
掛けて **143/146 → 146/146 COMPLIANT**、同時に `pdfua-1` も **106/106 を維持**、
添付（`/AF` + `/EmbeddedFiles`）も生存。

### PDF/A-4（`pdfa-4` / `pdfa-4f`）の違反 → writer 操作

> **-3b と要求が違う。番号を差し替えただけでは通らない。** -4 は PDF 2.0 基盤なので、
> **入力を `pdfVersion: "2.0"` で作っておく**のが前提（1.7 の文書に後から掛けると
> `ensure_pdfa` がヘッダを上げるが、署名済みなら上げられないので拒否される）。

| veraPDF の違反 | 意味 | writer での修正 |
|---|---|---|
| `6.1.2-1` | ヘッダが `%PDF-2.n` でない | `create_*` の **`pdfVersion: "2.0"`**（`ensure_pdfa(flavour: "pdfa-4")` も上げる） |
| `6.1.3-1` | trailer に `/ID` 無し | `ensure_pdfa` |
| `6.1.3-4` / `6.1.3-5` | **Info 辞書が存在する**（`/PieceInfo` が無い限り置けない。置くなら `ModDate` だけ） | `ensure_pdfa`（Info を削除する） |
| `6.2.4.3-2` | DeviceRGB に OutputIntent が無い | `ensure_pdfa` |
| `6.7.3-1` / `6.7.3-5` | pdfaid が無い / `pdfaid:rev` が `2020` でない | `ensure_pdfa` |
| `6.7.3-3` | `pdfaid:conformance` が `F` でない（-4f を名乗るのに） | `ensure_pdfa(flavour: "pdfa-4f")` |
| **`6.9-3`** | **添付ファイル自身が PDF/A でない** | **`flavour` を `pdfa-4f` に変える**（添付を消すのではない） |

**`6.9-3` が -4 でいちばん引っかかる。** 素の `pdfa-4` は添付が PDF/A であることを要求するので、
**CSV / JSON を同梱した瞬間に非適合**になる。電帳法の使い方は事実上つねに `-4f`。
実測: 同一文書が `pdfa-4` で **108/109**、`pdfa-4f` で **109/109 COMPLIANT**。

**Info の扱いが -3b と決定的に違う。** ISO 32000-2 §14.3.3 は Info に `CreationDate` と
`ModDate` を残してよいとするが、**-4 は Info そのものを許さない**。作成日時は
`xmp:CreateDate` が持つので情報は失われない。`get_metadata` で読み戻すと Info 由来の
題名・作成者が消えて見えるが、**それは正しい状態**（XMP を見ること）。

**実測（2026-07-27）**: PDF 2.0 で生成 → `attach_file`（CSV）→ `ensure_pdfa(flavour: "pdfa-4f")`
→ **109/109 COMPLIANT**、`/AF` と添付は qpdf 読み戻しで生存。

## 実測で判明している落とし穴

1. **`tagged: true` × 標準フォントは必ず違反**（7.21.4.1-1）。日本語の有無に関係なく、
   タグ付き生成には埋め込みフォントが必要。writer v0.8.0+ は warnings で予告する
2. **タグ付き文書に AcroForm があるだけでは通らない**。修復前の実測はちょうど
   7.18.1-3 / 7.18.3-1 / 7.18.4-1 の 3 違反 → `tag_form_fields` 適用後 106/106 COMPLIANT
3. **flatten はタグ付き文書を必ず壊す**（7.1-3 実測）。writer は既定で拒否する。
   「値を固定したい」要望には、readonly 化ではなく「対話フォームのまま + tag_form_fields」
   を第一候補として提案する
4. **watermark / stamp_page_numbers / attach_file / add_annotation はタグ付き文書でも
   準拠を維持する**（Artifact 化 / Annot タグ内包 / veraPDF 実測済み）。修正ループ中に
   これらを疑わない
5. **見出しレベルは正規化される**（H1 始まり・飛ばさない）。Markdown の `# → ###` は
   構造上 H1 → H2 になる。「見出しが仕様と違う」という指摘はまずこれを疑う
6. **機械検証の限界**: veraPDF が見るのは「存在」であって「適切さ」ではない。
   /Alt や /TU の中身、読み順の妥当性は人手レビュー事項としてレポートに残す
7. **埋め込み有無の確定は verify に委ね、reader の観測だけを根拠に修正ループへ入らない。**
   かつて reader の `inspect_fonts` は Type0 (CIDFont) の埋め込みを "Embedded: No" と
   誤報告した（FontFile3 が DescendantFont 側の FontDescriptor にあるため。2026-07-17 実測）。
   **これは reader 0.9.1 で解消済み** — 2026-07-20 実測では writer 0.14.0 の出力
   （`CIDFontType0` + `FontFile3` + サブセットタグ）に対し `isEmbedded: true` が正しく返り、
   veraPDF も 106/106 で一致した。
   ただし**原則は残る**: reader は観測、verify は判定であり、両者が食い違ったときに
   従うのは verify。この項は「実例が解消しても原則は残る」ことの記録として置いている

8. **`snake_case` の `_` が消える不具合（B-17）は writer 側で修正済み**（2026-07-21）。
   2026-07-20 実測では `identify_conformance` → `identifyconformance` となり、
   **exit 0・warnings なし**・バッククォートでも防げなかった。
   修正版では `_` の強調判定に語中条件が入り、コードスパンも保護される。
   **ただし writer のバージョンを確認すること** — 0.14.0 以前を掴んでいる環境では今も壊れる。
   Phase 2 の「入力との照合」は、この種の欠陥が**バージョン差で復活しうる**ため引き続き必須
9. **`title` と本文の先頭見出しが両方 H1 になる**（2026-07-20 実測: `roleCounts` の `H1` が 2）。
   veraPDF PDF/UA-1 は通る（106/106）ので**違反ではない**が、構造木を読み戻して
   再生成する用途では見出しが重複する。`title` と本文 `#` を重複させない書き方を提案する
10. **リストの `/Lbl` は出力されない**（実測: 構造木は `L → LI → LBody` のみ）。
   箇条書き記号・番号は LBody の本文に焼き込まれる。ISO 32000-2 §14.8.4.8.2 Table 370 の
   LI の行は「LI structure elements **often include** Lbl」という NOTE であって `shall` では
   ないため**適合違反ではない**。ただし読み戻して再生成すると記号が二重化する

## 操作順序の根拠（Phase 1）

- `tag_form_fields` は**フォームが確定してから**（後からフィールドを足したら再実行。冪等なので安全）
- Artifact 系（watermark / stamp）は本文構造に影響しないため順序自由だが、
  `flatten` だけは他のすべての後（そもそもタグ付きでは使わない）
- `attach_file` は catalog 操作のみで構造木に触れない — いつでも安全
- **`ensure_pdfa` は必ず最後**（PDF/A 要件のときだけ使う。flavour = -3b / -4 / -4f）。とくに **`attach_file` の後**:
  - 正順（`attach_file` → `ensure_pdfa`）は**実測済み** — 添付が生き残り 146/146 COMPLIANT
  - **逆順は未検証**。後から `/AF` が増えると `AFRelationship` が `Unspecified` のままになりうる
  - `ensure_pdfa` の後に本文・構造を変える操作を足したら、**PDF/A の採点はやり直し**
