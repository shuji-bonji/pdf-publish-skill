# writer 構造化エラーの分岐表

pdf-writer-mcp v0.7.0+ は family 契約の構造化エラーを返す:

```jsonc
{
  "error": "1文の人間可読メッセージ",
  "code": "SIGNED_PDF",              // ここで分岐する
  "hint": "追加情報（任意）",
  "next_actions": [{ "action": "...", "reason": "...", "example": { ... } }],
  "retryable": true                   // フラグや引数を変えれば再試行できる
}
```

原則: **`next_actions` が再試行の具体形（example）まで持っている**。メッセージ文字列を
grep して推測しない。`retryable: true` でも、ガード系は利用者の確認を先に取る。

## コード別の対応

| code | 意味 | Skill の行動 |
|---|---|---|
| `SIGNED_PDF` | 署名ガード停止（/ByteRange 検知） | **注釈追加なら第一候補は `preserveSignatures: true`**（v0.9.0〜。署名を保持したまま増分更新で追加。タグ無し文書のみ）。それ以外の編集は利用者に「署名を無効化してよいか」を確認 → 可なら `allowBreakingSignatures: true` で再試行し、レポートに「署名は意図的に無効化」と明記。**認証署名（DocMDP）P=1/P=2 の文書は preserveSignatures でも拒否される**（ISO 32000-2 §12.8.2.2: 注釈は P=3 のみ許可）— この場合は変更自体を断念するか破壊的編集の了解を取る |
| `SIGNED_PDF`（`ensure_pdfa(flavour: "pdfa-4"/"pdfa-4f")` で） | 署名済み・かつ入力が PDF 1.7 | **PDF/A-4 は `%PDF-2.n` を要求するが、増分更新はファイル先頭のヘッダを書き換えられない**（書き換えれば署名が壊れる）。`preserveSignatures` を外して再署名するか、**-3b に倒す**かを利用者に選ばせる。writer が黙って版を上げることはない |
| `TAGGED_PDF` | タグ付き文書を壊す操作（flatten 等） | 原則**代替案を先に出す**（flatten せず対話フォームのまま納品 + tag_form_fields で準拠化）。それでも必要なら利用者確認 → `allowBreakingTags: true` + レポートに「PDF/UA 非準拠になった」と明記 |
| `FONT_REQUIRED` | 非 Latin 文字 × フォント未指定 | `fontPath`（Noto Sans JP SubsetOTF/JP 推奨）か `PDF_WRITER_FONT` を案内して再試行 |
| `MISSING_GLYPH` | フォント未収録文字 | 利用者に方針確認: 別フォント / `onMissingGlyph: "replace"`（〓置換）/ `"ignore"`。replace・ignore で進めた場合は warnings をレポートに転記 |
| `ENCRYPTED_PDF` | 暗号化入力 | 復号済みファイルの用意を依頼（writer は復号を持たない） |
| `DOC_NOT_FOUND` / `FONT_NOT_FOUND` | パス誤り | 絶対パスか・実在するかを確認。相対パスは受け付けない |
| `FILE_TOO_LARGE` | 入力 100MB 超 | 分割（split_pdf は入力側なので不可）や元データの縮小を相談 |
| `UNSUPPORTED_PDF_FEATURE` | XFA フォーム等 | 対応不可と報告（XFA は ISO 32000-2 で非推奨・PDF/UA-1 7.15 で禁止） |
| `INVALID_ARGUMENT` | 引数不正 | メッセージに違反フィールドが列挙される。修正して再試行。**tag_form_fields をタグ無し文書に当てた場合もこれ**（hint が create tagged / ensure_tagged へ誘導する） |
| `INVALID_PDF` | 破損・パース不能 | 入力の入手元に差し戻し。pdf-trust skill での受入監査を提案 |
| `INTERNAL_ERROR` | 想定外 | 再試行せず、引数とエラーを添えて利用者へ報告（バグの可能性） |

## フォームフィールド名の発見

`fill_form` / `tag_form_fields` で**存在しないフィールド名**を指定すると、エラーに
実在する全フィールド名と型が列挙される。フィールド一覧の専用ツールは不要 —
わざと軽い誤指定をするのではなく、まず reader の `inspect_annotations` で観測するか、
エラーが出たらその列挙を使う。
