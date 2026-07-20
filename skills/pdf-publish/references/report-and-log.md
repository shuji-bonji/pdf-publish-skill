# Publish Report テンプレートと実行ログ（JSONL）仕様

## Publish Report

納品時に必ず添える。単発生成でも省略しない（warnings の伝達漏れが事故になる）。

```markdown
# Publish Report

- 成果物: /abs/path/deliverable.pdf（3 pages, 91,788 bytes）
- 品質ゲート水準: conformance (PDF/UA-1)
- 実行した操作: create_markdown_pdf(tagged, fontPath=NotoSansJP)
    → tag_form_fields(labels: 3) → stamp_page_numbers → attach_file(Data)
    → flatten_form: **スキップ**（前段 tag_form_fields が未実行のため。失敗ではない）

## 読み戻し（reader・観測）

- テキスト抽出: OK（本文冒頭一致）
- 入力との照合: 識別子 12 件中 12 件残存 / 見出し 5 件・段落 9 件で入力と一致
- 要求機能の実在: attach_file → 添付 1 件を確認 / add_bookmarks → 未実行
- 論理順（extract_structured_text）: H1→P→H2→L — 意図どおり
- フォント: NotoSansJP-Regular（埋め込みサブセット・isEmbedded: true）
- 構造: H1→H2、Table、L/LI — 意図どおり
- メタデータ: title="…" / lang="ja"

## 判定（verify）

- identify_conformance（**自己申告**）: PDF/UA-1 宣言あり
- validate_conformance（**第三者採点**）pdfua-1 → **COMPLIANT (106/106)** — engine: verapdf
- 宣言と採点の差分: **一致**（乖離があればここに明記する）
- （編集案件）入力の採点との差分: 1 違反 → 0 違反（修復が効いた）
- （署名済み入力を編集した場合）verify_integrity: 署名は意図どおり無効化

## warnings（writer からの事実報告・全件）

- Inferred document language as "ja"; pass "lang" explicitly to override.
- No label given for 1 field(s); the field name was used as /TU (agree). …

## 人手レビュー事項

- /TU・/Alt の文言が適切か（機械検証は存在しか見ない）
- ループ回数: 1（初回で通過）
```

書き方の規律:

- 判定行には必ず**エンジン**（verapdf / native）を書く。native の「違反なし」は
  `compliant: null` = 必要条件のみであり、COMPLIANT と書いてはならない
- **自己申告（identify）と第三者採点（validate）を必ず並べて書く。** 片方だけでは
  「宣言が嘘」も「実体の違反」も見逃す。両者が食い違ったら、それが所見の本体
- **失敗とスキップを書き分ける。** 前段の失敗に巻き添えになった段は「スキップ」であり、
  「失敗」と書くと原因の所在が読み手に伝わらない
- warnings は writer が返したものを**全件そのまま**転記する（要約で情報を落とさない）
- 未実施項目（reader 未接続など）は「未実施」と明記する — 黙って落とすと
  「チェック済みで問題なし」と誤読される（pdf-trust と同じ規律）

## 実行ログ（JSONL）— 学習データ工場の出荷記録

read-write-verify ループの実行記録。**verify の verdict がラベル**になり、
PDF 専門 LLM（family の北極星）の学習データになる。

- **opt-in**: Phase 0 で合意した場合のみ書く。勝手に書かない
- 既定の保存先: 納品先と同じディレクトリの `publish-log.jsonl`（利用者指定があればそちら）
- 1 実行 = 1 行。追記のみ（過去行を書き換えない）
- **PII 配慮**: `intent` と `args_digest` に本文・個人情報を入れない。
  ツール名と引数の**形**（キー名・件数）だけを記録する

```jsonc
{
  "ts": "2026-07-17T12:34:56+09:00",
  "intent": "月次報告書を PDF/UA で PDF 化",          // 要求の要旨（PII を含めない）
  "gate": "conformance",
  "tool_calls": [
    { "tool": "create_markdown_pdf", "args_shape": ["markdown", "title", "tagged", "lang", "fontPath", "outputPath"] },
    { "tool": "tag_form_fields", "args_shape": ["inputPath", "labels(3)", "outputPath"] }
  ],
  "readback": { "text_ok": true, "fonts_embedded": true, "tags_ok": true,
                "input_match": true, "features_present": ["attachment"] },
  "declared": { "pdfa": null, "pdfua": "1" },           // identify_conformance（自己申告）
  "verdict": { "engine": "verapdf", "flavour": "pdfua-1", "compliant": true, "violations": [] },
  "baseline": null,                                     // 編集案件は入力採点の verdict を入れる
  "skipped": [],                                        // 前段失敗で実行しなかった段
  "loops": 1,
  "warnings_count": 2,
  "errors": []                                        // 遭遇した code の列（再試行含む）
}
```

違反があった実行こそ価値が高い（負例 + 修正列）。`violations` には clause ID を、
`errors` には writer の `code` を残す。スキーマの確定・収集方法は localLLM
プロジェクト（訓練基盤）側と合わせて更新する。
