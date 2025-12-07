# fileprep

[![Go Reference](https://pkg.go.dev/badge/github.com/nao1215/fileprep.svg)](https://pkg.go.dev/github.com/nao1215/fileprep)
[![Go Report Card](https://goreportcard.com/badge/github.com/nao1215/fileprep)](https://goreportcard.com/report/github.com/nao1215/fileprep)
[![MultiPlatformUnitTest](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml/badge.svg)](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml)
![Coverage](https://raw.githubusercontent.com/nao1215/octocovs-central-repo/main/badges/nao1215/fileprep/coverage.svg)

[English](../../README.md) | [Español](../es/README.md) | [Français](../fr/README.md) | [한국어](../ko/README.md) | [Русский](../ru/README.md) | [中文](../zh-cn/README.md)

![fileprep-logo](../images/fileprep-logo-small.png)

**fileprep** は、CSV、TSV、LTSV、Parquet、Excelなどの構造化データを、軽量なstructタグルールでクリーニング、正規化、バリデーションするためのGoライブラリです。gzip、bzip2、xz、zstdストリームをシームレスにサポートしています。

## なぜfileprepなのか？

私は [nao1215/filesql](https://github.com/nao1215/filesql) を開発しました。これはCSV、TSV、LTSV、Parquet、ExcelファイルにSQLクエリを実行できるライブラリです。また、CSVファイルのバリデーション用に [nao1215/csv](https://github.com/nao1215/csv) も作成しました。

機械学習を勉強する中で、「[nao1215/csv](https://github.com/nao1215/csv) を [nao1215/filesql](https://github.com/nao1215/filesql) と同じファイル形式に対応させれば、両者を組み合わせてETLのような操作ができる」と気づきました。このアイデアが **fileprep** の誕生につながりました。データの前処理/バリデーションとSQLベースのファイルクエリを橋渡しするライブラリです。

## 機能

- 複数ファイル形式対応: CSV, TSV, LTSV, Parquet, Excel (.xlsx)
- 圧縮対応: gzip (.gz), bzip2 (.bz2), xz (.xz), zstd (.zst)
- 名前ベースのカラムバインディング: フィールドは自動的に `snake_case` カラム名にマッチ、`name` タグでカスタマイズ可能
- structタグベースの前処理 (`prep` タグ): trim、lowercase、uppercase、デフォルト値など
- structタグベースのバリデーション (`validate` タグ): required など
- [filesql](https://github.com/nao1215/filesql) とのシームレスな統合: filesqlで直接使用できる `io.Reader` を返却
- 詳細なエラーレポート: 各エラーの行と列の情報

## インストール

```bash
go get github.com/nao1215/fileprep
```

## 必要要件

- Go バージョン: 1.24 以降
- 対応OS:
  - Linux
  - macOS
  - Windows

## クイックスタート

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/nao1215/fileprep"
)

// User は前処理とバリデーション付きのユーザーレコードを表します
type User struct {
    Name  string `prep:"trim" validate:"required"`
    Email string `prep:"trim,lowercase"`
    Age   string
}

func main() {
    csvData := `name,email,age
  John Doe  ,JOHN@EXAMPLE.COM,30
Jane Smith,jane@example.com,25
`

    processor := fileprep.NewProcessor(fileprep.FileTypeCSV)
    var users []User

    reader, result, err := processor.Process(strings.NewReader(csvData), &users)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }

    fmt.Printf("処理完了: %d行中 %d行が有効\n", result.RowCount, result.ValidRowCount)

    for _, user := range users {
        fmt.Printf("Name: %q, Email: %q\n", user.Name, user.Email)
    }

    // readerはfilesqlに直接渡すことができます
    _ = reader
}
```

出力:
```
処理完了: 2行中 2行が有効
Name: "John Doe", Email: "john@example.com"
Name: "Jane Smith", Email: "jane@example.com"
```

## 前処理タグ (`prep`)

複数のタグを組み合わせることができます: `prep:"trim,lowercase,default=N/A"`

### 基本的な前処理

| タグ | 説明 | 例 |
|-----|------|-----|
| `trim` | 前後の空白を削除 | `prep:"trim"` |
| `ltrim` | 先頭の空白を削除 | `prep:"ltrim"` |
| `rtrim` | 末尾の空白を削除 | `prep:"rtrim"` |
| `lowercase` | 小文字に変換 | `prep:"lowercase"` |
| `uppercase` | 大文字に変換 | `prep:"uppercase"` |
| `default=value` | 空の場合にデフォルト値を設定 | `prep:"default=N/A"` |

### 文字列変換

| タグ | 説明 | 例 |
|-----|------|-----|
| `replace=old:new` | すべての出現を置換 | `prep:"replace=;:,"` |
| `prefix=value` | 文字列を先頭に追加 | `prep:"prefix=ID_"` |
| `suffix=value` | 文字列を末尾に追加 | `prep:"suffix=_END"` |
| `truncate=N` | N文字に制限 | `prep:"truncate=100"` |
| `strip_html` | HTMLタグを削除 | `prep:"strip_html"` |
| `strip_newline` | 改行を削除 (LF, CRLF, CR) | `prep:"strip_newline"` |
| `collapse_space` | 複数のスペースを1つに | `prep:"collapse_space"` |

### 文字フィルタリング

| タグ | 説明 | 例 |
|-----|------|-----|
| `remove_digits` | すべての数字を削除 | `prep:"remove_digits"` |
| `remove_alpha` | すべてのアルファベットを削除 | `prep:"remove_alpha"` |
| `keep_digits` | 数字のみを保持 | `prep:"keep_digits"` |
| `keep_alpha` | アルファベットのみを保持 | `prep:"keep_alpha"` |
| `trim_set=chars` | 指定文字を両端から削除 | `prep:"trim_set=@#$"` |

### パディング

| タグ | 説明 | 例 |
|-----|------|-----|
| `pad_left=N:char` | N文字まで左にパディング | `prep:"pad_left=5:0"` |
| `pad_right=N:char` | N文字まで右にパディング | `prep:"pad_right=10: "` |

### 高度な前処理

| タグ | 説明 | 例 |
|-----|------|-----|
| `normalize_unicode` | UnicodeをNFC形式に正規化 | `prep:"normalize_unicode"` |
| `nullify=value` | 特定の文字列を空として扱う | `prep:"nullify=NULL"` |
| `coerce=type` | 型変換 (int, float, bool) | `prep:"coerce=int"` |
| `fix_scheme=scheme` | URLスキームを追加/修正 | `prep:"fix_scheme=https"` |
| `regex_replace=pattern:replacement` | 正規表現による置換 | `prep:"regex_replace=\\d+:X"` |

## バリデーションタグ (`validate`)

複数のタグを組み合わせることができます: `validate:"required,email"`

### 基本的なバリデータ

| タグ | 説明 | 例 |
|-----|------|-----|
| `required` | フィールドは空であってはならない | `validate:"required"` |
| `boolean` | true, false, 0, または 1 である必要がある | `validate:"boolean"` |

### フォーマットバリデータ

| タグ | 説明 | 例 |
|-----|------|-----|
| `email` | 有効なメールアドレス | `validate:"email"` |
| `uri` | 有効なURI | `validate:"uri"` |
| `url` | 有効なURL | `validate:"url"` |
| `uuid` | 有効なUUID | `validate:"uuid"` |

### クロスフィールドバリデータ

| タグ | 説明 | 例 |
|-----|------|-----|
| `eqfield=Field` | 値が別のフィールドと等しい | `validate:"eqfield=Password"` |
| `nefield=Field` | 値が別のフィールドと等しくない | `validate:"nefield=OldPassword"` |
| `gtfield=Field` | 値が別のフィールドより大きい | `validate:"gtfield=MinPrice"` |
| `ltfield=Field` | 値が別のフィールドより小さい | `validate:"ltfield=MaxPrice"` |

## サポートされているファイル形式

| 形式 | 拡張子 | 圧縮形式 |
|------|--------|----------|
| CSV | `.csv` | `.csv.gz`, `.csv.bz2`, `.csv.xz`, `.csv.zst` |
| TSV | `.tsv` | `.tsv.gz`, `.tsv.bz2`, `.tsv.xz`, `.tsv.zst` |
| LTSV | `.ltsv` | `.ltsv.gz`, `.ltsv.bz2`, `.ltsv.xz`, `.ltsv.zst` |
| Excel | `.xlsx` | `.xlsx.gz`, `.xlsx.bz2`, `.xlsx.xz`, `.xlsx.zst` |
| Parquet | `.parquet` | `.parquet.gz`, `.parquet.bz2`, `.parquet.xz`, `.parquet.zst` |

**Excelファイルについての注意**: **最初のシート**のみが処理されます。複数シートのワークブックでは、後続のシートは無視されます。

## filesqlとの連携

```go
// 前処理とバリデーションでファイルを処理
processor := fileprep.NewProcessor(fileprep.FileTypeCSV)
var records []MyRecord

reader, result, err := processor.Process(file, &records)
if err != nil {
    return err
}

// バリデーションエラーをチェック
if result.HasErrors() {
    for _, e := range result.ValidationErrors() {
        log.Printf("行 %d, カラム %s: %s", e.Row, e.Column, e.Message)
    }
}

// 前処理済みデータをfilesqlに渡す
ctx := context.Background()
builder := filesql.NewBuilder().
    AddReader(reader, "my_table", filesql.FileTypeCSV)

validatedBuilder, err := builder.Build(ctx)
if err != nil {
    return err
}

db, err := validatedBuilder.Open(ctx)
if err != nil {
    return err
}
defer db.Close()

// 前処理済みデータに対してSQLクエリを実行
rows, err := db.QueryContext(ctx, "SELECT * FROM my_table WHERE age > 20")
```

## メモリ使用量

fileprepは処理のために**ファイル全体をメモリに読み込みます**。これによりランダムアクセスとマルチパス操作が可能になりますが、大きなファイルには影響があります:

| ファイルサイズ | 概算メモリ | 推奨事項 |
|----------------|------------|----------|
| < 100 MB | ファイルサイズの約2-3倍 | 直接処理 |
| 100-500 MB | 500 MB - 1.5 GB | メモリ監視、チャンク処理を検討 |
| > 500 MB | > 1.5 GB | ファイル分割またはストリーミング代替案を使用 |

## 関連プロジェクト

- [nao1215/filesql](https://github.com/nao1215/filesql) - CSV, TSV, LTSV, Parquet, Excel用のSQLドライバー
- [nao1215/csv](https://github.com/nao1215/csv) - バリデーション付きCSV読み込みとシンプルなDataFrame

## コントリビューション

コントリビューションを歓迎します！詳細は [Contributing Guide](../../CONTRIBUTING.md) をご覧ください。

## サポート

このプロジェクトが役に立つと思われた場合は、以下をご検討ください:

- GitHubでスターを付ける - 他の人がプロジェクトを発見するのに役立ちます
- [スポンサーになる](https://github.com/sponsors/nao1215) - あなたのサポートがプロジェクトを存続させ、継続的な開発のモチベーションになります

スター、スポンサーシップ、コントリビューションを通じたあなたのサポートが、このプロジェクトを前進させる力です。ありがとうございます！

## ライセンス

このプロジェクトはMITライセンスの下でライセンスされています - 詳細は [LICENSE](../../LICENSE) ファイルをご覧ください。
