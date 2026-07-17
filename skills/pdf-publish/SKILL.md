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

## 前提 MCP

| MCP | 必須/任意 | 役割 |
|---|---|---|
| pdf-writer-mcp (v0.8.0+) | **必須** | 生成（Tier 0）・編集（Tier A/B）・PDF/UA 修復のすべて |
| pdf-verify-mcp | 品質ゲート案件では**必須** | validate_conformance（veraPDF 委譲）/ verify_integrity |
| pdf-reader-mcp | 推奨 | 読み戻し（テキスト抽出・フォント・タグ・メタデータの観測） |
| pdf-spec-mcp | 任意 | 違反時の ISO 32000 / 14289 条項の根拠引用 |

pdf-writer-mcp が未接続なら成立しない。`npx @shuji-bonji/pdf-writer-mcp@latest` の接続を
案内して停止する。品質ゲート水準 `conformance`（後述）の案件で pdf-verify-mcp が無い場合も
中止する — ゲート無しで「準拠」を名乗るのは虚偽になる。reader が無い場合は縮退動作し、
レポートに「読み戻し: 未実施（ツール未接続）」と明記する。

## 手順

### Phase 0 — 要件確認

利用者と次を合意する。曖昧なまま進めない（後半の手戻りがループ上限を浪費する）:

1. **成果物の種別** — 新規生成（text / markdown / table）か、既存 PDF の編集か
2. **品質ゲート水準** — 明示がなければ内容から推定して提案する:

   | 水準 | 内容 | 既定で選ぶ場面 |
   |---|---|---|
   | `none` | 生成のみ（ゲート無し） | 下書き・使い捨て |
   | `readback` | reader で読み戻し観測まで | 一般の納品物 |
   | `conformance` | verify で COMPLIANT まで | `tagged` 指定時・アクセシビリティ/長期保存要件・行政/医療向け |

3. **日本語の有無** → 含むなら `fontPath` / 環境変数 `PDF_WRITER_FONT` の確認。
   **`tagged: true` なら日本語が無くても埋め込みフォントが必須**（標準 Helvetica は
   PDF/UA-1 7.21.4.1 で必ず違反 — veraPDF 実測）。`title` も必須（7.1）
4. **電帳法・PDF/A-3 文脈か** → attach_file の `relationship`（Data / Source）を確認
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
```

- 各ツールの `warnings` をすべて収集する（lang 推定・グリフ置換・/TU 代用など。
  Phase 5 のレポートに全件載せる）
- エラーが返ったら [references/error-codes.md](references/error-codes.md) の分岐に従う。
  ガード系（SIGNED_PDF / TAGGED_PDF）は**利用者に確認してから**解除フラグで再試行する

### Phase 2 — 読み戻し（reader・水準 readback 以上）

生成物に対して観測する。**合否は言わない**（それは Phase 3 の仕事）:

1. `read_text` — 意図した本文が抽出できるか（writer の既知リスク: 描画と抽出は独立に壊れる）
2. `inspect_fonts` — フォントが埋め込まれているか（conformance 水準では必須の観測）
3. `inspect_tags` — タグ付き指定時、構造木が意図どおりか（見出し階層・表・リスト）
4. `get_metadata` — title / lang / producer

### Phase 3 — 品質ゲート（verify・水準 conformance）

1. タグ付き出力 → `validate_conformance`（`flavour: "pdfua-1"`、engine は auto）
2. PDF/A 宣言のある入出力 → `validate_conformance`（該当 flavour）
3. 署名済み入力を（了解の上で）編集した場合 → `verify_integrity` で「署名が無効化された」
   ことを**意図どおり**と確認し、レポートに明記する

結果の扱い:

| verify の結果 | 行動 |
|---|---|
| COMPLIANT（veraPDF） | 納品へ。規則数（例 106/106）をレポートに記載 |
| 違反あり・writer で修正可能 | Phase 4 の修正ループへ |
| 違反あり・writer の能力外 | 人手レビュー要請。Tier C 未実装（本文編集・タグ木保守）であることを明記 |
| `compliant: null`（native エンジン） | 「検査サブセットで違反なし＝必要条件のみ」と明記。veraPDF 導入を提案 |
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
   成果物パス・実行した操作列・読み戻しの観測・verify の判定（エンジンと規則数）・
   warnings 全件・ループ回数
2. Phase 0 で合意していれば**実行ログ（JSONL 1 行）**を追記する。
   これが read-write-verify ループの学習データ（verify の verdict がラベル）になる

## やらないこと

- 合否の自前判定（verify の結果以外を根拠に「準拠」と言うこと）
- verify 未接続での conformance 水準の納品
- 利用者の了解なしの `allowBreakingSignatures` / `allowBreakingTags`
- `outputPath` 省略での大きな PDF 生成（base64 溢れ）
- 内容（文章そのもの）の品質保証 — このパイプラインが保証するのは構造・準拠性・
  抽出可能性であって、文章の正しさではない
