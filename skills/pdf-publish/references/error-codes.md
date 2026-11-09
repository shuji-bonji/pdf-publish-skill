# 構造化エラーの分岐表（writer・reader・verify）

PDF family の 4 サーバーは、失敗を同じ形で返す（pdf-agent-stack `docs/specs/06` §2.3。
reader 0.19.0・writer 0.23.3・verify 0.33.0・spec 0.12.0 から）。ツール結果は `isError: true` で、
本文は平らな JSON のオブジェクト 1 つ:

```jsonc
{
  "error": "英語の 1 文",              // 必須
  "code": "SIGNED_PDF",               // 必須。ここで分岐する
  "retryable": true,                  // 必須。引数・フラグ・環境を変えれば成功しうるか
  "hint": "英語の補足",               // 任意
  "next_actions": [{ "action": "...", "reason": "...", "example": { ... } }],  // 任意
  "detail": { ... }                   // 任意
}
```

原則:

- **`code` で分岐する。** `error`・`hint` の文を grep して推測しない（文は英語で、版で変わる）
- **`next_actions` が再試行の具体形（`example`）まで持っている**ときは、それに従う
- **`retryable: true` でも、ガード系（`SIGNED_PDF`・`TAGGED_PDF`）は利用者の確認を先に取る**。
  `retryable` は「引数を変えれば通りうる」であって「黙って変えてよい」ではない
- **`retryable: false` を同じ引数で呼び直さない**
- 必須の引数が無い・宣言に無いキーを渡した、は SDK がハンドラの前で拒み、
  `Input validation error: …` の文のまま返る（この形ではない）。引数を直して呼び直す

全サーバーの一覧はサイトの [Error Codes](https://shuji-bonji.github.io/pdf-agent-stack/ja/reference/error-codes) にある。
ここには、この Skill の流れ（Phase 1 書く → Phase 2 読み戻す → Phase 3 採点する）で出会うものと、
そのときの行動だけを書く。

## Phase 1 — writer

| code | 意味 | `retryable` | Skill の行動 |
|---|---|---|---|
| `SIGNED_PDF` | 署名ガード停止（/ByteRange 検知） | `true` | **注釈追加なら第一候補は `preserveSignatures: true`**（v0.9.0〜。署名を保持したまま増分更新で追加。タグ無し文書のみ）。それ以外の編集は利用者に「署名を無効化してよいか」を確認 → 可なら `allowBreakingSignatures: true` で再試行し、レポートに「署名は意図的に無効化」と明記。**認証署名（DocMDP）P=1/P=2 の文書は preserveSignatures でも拒否される**（ISO 32000-2 §12.8.2.2: 注釈は P=3 のみ許可）— この場合は変更自体を断念するか破壊的編集の了解を取る |
| `SIGNED_PDF`（`ensure_pdfa(flavour: "pdfa-4"/"pdfa-4f")` で） | 署名済み・かつ入力が PDF 1.7 | `true` | **PDF/A-4 は `%PDF-2.n` を要求するが、増分更新はファイル先頭のヘッダを書き換えられない**（書き換えれば署名が壊れる）。`preserveSignatures` を外して再署名するか、**-3b に倒す**かを利用者に選ばせる。writer が黙って版を上げることはない |
| `TAGGED_PDF` | タグ付き文書を壊す操作（flatten 等） | `true` | 原則**代替案を先に出す**（flatten せず対話フォームのまま納品 + tag_form_fields で準拠化）。それでも必要なら利用者確認 → `allowBreakingTags: true` + レポートに「PDF/UA 非準拠になった」と明記 |
| `FONT_REQUIRED` | 標準フォント（Helvetica）で書けない字 × フォント未指定。v0.26.0 からは、WinAnsiEncoding に符号の無い U+0100 以上の字（日本語・`ł`・`✔` など）だけ。v0.25.x までは U+0100 以上の字すべてで、`“` `”` `–` `—` `€` `…` `™` も断られていた | `true` | `fontPath`（Noto Sans JP SubsetOTF/JP 推奨）か `PDF_WRITER_FONT` を案内して再試行。v0.26.0 から、原因の字が `detail.characters`（`U+XXXX` の配列）と `detail.count` に入る。どの字でフォントが要るのかを利用者に示すときは、`error` の文ではなく `detail` を使う |
| `FONT_NOT_FOUND` | フォントのパス誤り | `true` | 絶対パスか・実在するかを確認して再試行 |
| `MISSING_GLYPH` | フォント未収録文字（標準フォントでは、WinAnsiEncoding に符号の無い U+0100 未満の字。制御文字・U+007F〜U+009F） | `true` | 利用者に方針確認: 別フォント / `onMissingGlyph: "replace"`（`〓` に置き換える。フォントに `〓` が無ければ `?`。標準フォントでは必ず `?`）/ `"ignore"`（空白を描く）。replace・ignore で進めた場合は warnings をレポートに転記。原因の字は v0.26.0 から `detail.characters`（`U+XXXX`、最大 1,000 字）と `detail.count`（全体の数）に入る。`error` の文は 10 字で切れるので、利用者に示す字の一覧は `detail` から作る |
| `DOC_NOT_FOUND` | 入力 PDF のパス誤り | `false` | 絶対パスか・実在するかを確認する。相対パスは受け付けない（`INVALID_ARGUMENT`） |
| `ENCRYPTED_PDF` | 暗号化入力 | `false` | 復号済みファイルの用意を依頼（writer は復号も `password` 引数も持たない）。v0.24.0 から `detail.keyDerived` が付く: `true`（空の利用者パスワードで開ける）なら writer の外で復号すれば編集でき、読むだけなら reader・verify がそのまま読める。`false` なら利用者パスワードでの復号が要る。**v0.24.0 から暗号化を署名より先に確かめる**ので、署名付きの暗号化文書は `SIGNED_PDF` ではなくこれになる（`preserveSignatures` で再試行しない） |
| `FILE_TOO_LARGE` | 入力 100MB 超 | `false` | 分割（split_pdf は入力側なので不可）や元データの縮小を相談 |
| `UNSUPPORTED_PDF_FEATURE` | XFA フォーム等 | `false` | 対応不可と報告（XFA は ISO 32000-2 で非推奨・PDF/UA-1 7.15 で禁止） |
| `INVALID_ARGUMENT` | 引数不正 | `true` | メッセージに違反フィールドが列挙される。修正して再試行。**tag_form_fields をタグ無し文書に当てた場合もこれ**（hint が create tagged / ensure_tagged へ誘導する） |
| `INVALID_ARGUMENT`（`ensure_pdfa(flavour: "pdfa-4"/"pdfa-4f")` で、`next_actions[0].action` が `retry_with_flavour`・`example.flavour` が `pdfa-3b`） | 入力が PDF/UA-1 を宣言していて、PDF/A-4 の PDF 2.0 ヘッダーと両立しない（v0.24.0〜） | `true` | `example` の `pdfa-3b` に**黙って切り替えない**。SKILL.md Phase 0 の組み合わせの表に戻る（-3b + PDF/UA-1 に倒すか、PDF 2.0 で作り直して PDF/UA-2 + PDF/A-4 にするかを利用者に選ばせる） |
| `INVALID_ARGUMENT`（`ensure_pdfa(flavour: "pdfa-3b")` で、`next_actions[0].action` が `retry_with_flavour`・`example.flavour` が `pdfa-4`） | 入力が PDF 2.0 で、PDF/A-3 の `%PDF-1.n` ヘッダーと両立しない（PDF/UA-2 の入力は v0.27.0〜、タグ無しは v0.28.0〜）。エラーの文に入力の版と宣言が入る | `true` | `example` の `pdfa-4` に**黙って切り替えない**。目標が PDF/A-3b なら PDF 1.7 で作り直す、PDF/A-4 でよいなら `pdfa-4`（添付があれば `pdfa-4f`）にする、を利用者に選ばせる。writer は版を下げない |
| `INVALID_ARGUMENT`（`merge_pdfs` で、`next_actions[0].example` が `{ "pdfVersion": "2.0" }`） | 入力に PDF 2.0 と PDF 1.x が混ざっている（v0.28.0〜）。エラーの文に入力ごとの版が入る。`pdfVersion: "1.7"` を渡して PDF 2.0 の入力があっても同じ | `true` | `pdfVersion: "2.0"` は明示した変換（Info の題名などは XMP に移り、trailer `/ID` が付く）。PDF/A-3b を目標にしている案件では -3b を名乗れなくなるので**黙って従わない**。PDF 1.x の入力だけで結合するか、2.0 にするかを利用者に選ばせる |
| `INVALID_ARGUMENT`（PDF/UA-2 の文書への `add_annotation` で、`next_actions[0].action` が `retry_with_alt_as_contents`） | `contents` と `alt` が違う値（v0.28.1〜）。ISO 14289-2 §8.9.4.2 は注釈の `Contents` と囲む構造要素の `Alt` を同じ値にすることを求める | `true` | 1 つの文を両方に使う（`alt` だけを渡せば `Contents` にも入る）。どちらの文にするかは利用者の意図を確かめる |
| `INVALID_ARGUMENT`（`detail.reason` が `PERMISSION_DENIED`・`NOT_A_DIRECTORY` など） | 出力先に書けない（v0.24.0〜。v0.23.x は `INTERNAL_ERROR`）。`error` と `detail.path` が**出力のパス**を名指す | `true` | 書ける `outputPath` に変えるか、フォルダーの権限を直して再試行する。`detail.cause` に元の例外の文がある |
| `INVALID_PDF` | 入力が ISO 32000 の構造の条文に反していて読めない | `false` | 入力の入手元に差し戻し。pdf-trust skill での受入監査を提案。**サーバーの故障ではない** |
| `INTERNAL_ERROR` | 想定外 | `false` | 再試行せず、引数とエラーを添えて利用者へ報告（バグの可能性） |

## Phase 2 — reader（読み戻し）

Skill が書いた直後のファイルを読むので、ここでのエラーは**出力側の問題**として扱う。

| code | 意味 | `retryable` | Skill の行動 |
|---|---|---|---|
| `INVALID_PDF` | 出力が構造の条文に反していて読めない | `false` | **writer が壊れたファイルを書いた**ということ。納品しない。writer の操作・引数とエラーを添えて利用者へ報告し、Phase 3 には進まない |
| `ENCRYPTED_PDF` | 暗号化されていて reader が読めない（reader に `password` 引数は無い） | `false` | 読み戻しは**未実施**として Publish Report に書く（「読み戻し済み」と名乗らない） |
| `DOC_NOT_FOUND` | 出力のパスにファイルが無い | `false` | writer に渡した `outputPath` と同じ絶対パスかを確かめる |
| `RENDERER_NOT_AVAILABLE` | `render_page` の描画の部品 `@hyzyla/pdfium` が無い | `true` | 目視の確認は**未実施**として書く。部品を入れれば同じプロセスで通る |
| `INTERNAL_ERROR` | reader の故障 | `false` | 読み戻しは未実施として書き、利用者へ報告 |

エラーではなく成功応答の中で「読めなかった」が返ることもある（`scope` の `failed`、
`isTagged: null` など）。それは SKILL.md の Phase 2 の扱いに従う。

## Phase 3 — verify（品質ゲート）

| code | 意味 | `retryable` | Skill の行動 |
|---|---|---|---|
| `VERAPDF_NOT_FOUND` / `VERAPDF_NOT_AVAILABLE` | veraPDF が無いか動かない | `true` | `PDF_VERIFY_VERAPDF` の設定を案内する。`engine: "native"` は検査サブセットなので、conformance 水準の判定は**保留**（SKILL.md Phase 3 の 4） |
| `VERAPDF_NO_RESULT` | veraPDF は動いたが結果を返さなかった | `false` | 合格でも故障でもない。conformance 水準の判定は**保留**として書く |
| `ENCRYPTED_PDF` | 暗号化されているので判定しなかった。veraPDF のある環境の暗号化文書と、v0.35.0 からは**鍵を導けない暗号化文書の PDF/A 判定（エンジンにかかわらず）** | `true` | 鍵を導けない文書なら `next_actions` が `password` を付けた `validate_conformance` を案内する。パスワードを知っていれば渡して再試行。知らなければ PDF/A の判定は**未実施**として書く（「違反あり」とも書かない）。PDF/A はそもそも暗号化を許さない（ISO 19005）ので、鍵を導ける文書でも veraPDF の実行は未実施として書く |
| `WRONG_PASSWORD` | パスワードが誤っているか、Standard 以外のセキュリティハンドラ | `true` | 利用者にパスワードを確かめる |
| `INVALID_PDF` | 出力が構造の条文に反していて読めない | `false` | **構造の違反という所見**として扱う（「採点できなかった」に入れない）。納品しない |
| `INVALID_ARGUMENT` | 引数不正（知らない `flavour` など。PDF/A-4 に `pdfa-4b` は無い） | `true` | `detail.reason` を見て引数を直す |
| `INVALID_BYTE_RANGE` | 署名の `/ByteRange` が壊れている | `false` | 署名済み入力を編集した案件なら、署名が壊れた所見として報告する |
| `INTERNAL_ERROR` | verify の故障 | `false` | 採点は未実施として書き、利用者へ報告。conformance 水準の案件は納品しない |

**旧い名前（外した）。** verify 0.33.x は移行の版で、新しいコードを返しながら旧名を
`detail.legacyCode` に、`error`・`hint` と同じ値を `message`・`suggestion` に残していた。
**verify 0.34.0 で旧名を外した。** 分岐は `code` で行う。旧名を読んでいた場合は次のように置き換える。

| 旧名 | いまの名前 |
|---|---|
| `FILE_NOT_FOUND` | `DOC_NOT_FOUND` |
| `PARSE_FAILED` | `INVALID_PDF` |
| `MISSING_PATH`・`RELATIVE_PATH`・`NOT_A_FILE`・`INVALID_FLAVOUR` | `INVALID_ARGUMENT`（旧名は `detail.reason` にある） |
| `message`・`suggestion` | `error`・`hint` |

## フォームフィールド名の発見

`fill_form` / `tag_form_fields` で**存在しないフィールド名**を指定すると、エラーに
実在する全フィールド名と型が列挙される。フィールド一覧の専用ツールは不要 —
わざと軽い誤指定をするのではなく、まず reader の `inspect_annotations` で観測するか、
エラーが出たらその列挙を使う。
