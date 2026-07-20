# PDF/UA 準拠の実測ノートと違反→修正対応表

すべて veraPDF（`--flavour ua1`、106 規則）での実測に基づく。
判定は必ず pdf-verify-mcp の `validate_conformance` で取ること。

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

8. **`create_markdown_pdf` はインライン装飾記号を除去するため、`snake_case` の `_` が消える**
   （2026-07-20 実測: `identify_conformance` → `identifyconformance`）。
   **exit 0・warnings なし**で起きるため、読み戻して入力と照合しない限り気づけない。
   関数名・ツール名・識別子を含む技術文書では実害がある。当面の回避策は、
   該当語を `_` を含まない表記にするか、生成後に Phase 2 の照合で検出して利用者に報告する
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
