# fileprep

[![Go Reference](https://pkg.go.dev/badge/github.com/nao1215/fileprep.svg)](https://pkg.go.dev/github.com/nao1215/fileprep)
[![Go Report Card](https://goreportcard.com/badge/github.com/nao1215/fileprep)](https://goreportcard.com/report/github.com/nao1215/fileprep)
[![MultiPlatformUnitTest](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml/badge.svg)](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml)
![Coverage](https://raw.githubusercontent.com/nao1215/octocovs-central-repo/main/badges/nao1215/fileprep/coverage.svg)

[English](../../README.md) | [日本語](../ja/README.md) | [Español](../es/README.md) | [Français](../fr/README.md) | [한국어](../ko/README.md) | [Русский](../ru/README.md)

![fileprep-logo](../images/fileprep-logo-small.png)

**fileprep** 是一个用于清理、规范化和验证结构化数据（CSV、TSV、LTSV、Parquet 和 Excel）的 Go 库，通过轻量级的 struct 标签规则实现，无缝支持 gzip、bzip2、xz 和 zstd 压缩流。

## 为什么选择 fileprep？

我开发了 [nao1215/filesql](https://github.com/nao1215/filesql)，它可以对 CSV、TSV、LTSV、Parquet 和 Excel 文件执行 SQL 查询。我还创建了 [nao1215/csv](https://github.com/nao1215/csv) 用于 CSV 文件验证。

在学习机器学习的过程中，我意识到："如果将 [nao1215/csv](https://github.com/nao1215/csv) 扩展为支持与 [nao1215/filesql](https://github.com/nao1215/filesql) 相同的文件格式，就可以将两者结合起来执行类似 ETL 的操作"。这个想法促成了 **fileprep** 的诞生：一个连接数据预处理/验证与基于 SQL 的文件查询的库。

## 功能

- 多文件格式支持：CSV、TSV、LTSV、Parquet、Excel (.xlsx)
- 压缩支持：gzip (.gz)、bzip2 (.bz2)、xz (.xz)、zstd (.zst)
- 基于名称的列绑定：字段自动匹配 `snake_case` 列名，可通过 `name` 标签自定义
- 基于 struct 标签的预处理（`prep` 标签）：trim、lowercase、uppercase、默认值等
- 基于 struct 标签的验证（`validate` 标签）：required 等
- [filesql](https://github.com/nao1215/filesql) 无缝集成：返回 `io.Reader` 可直接用于 filesql
- 详细的错误报告：每个错误包含行和列信息

## 安装

```bash
go get github.com/nao1215/fileprep
```

## 要求

- Go 版本：1.24 或更高
- 支持的操作系统：
  - Linux
  - macOS
  - Windows

## 快速开始

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/nao1215/fileprep"
)

// User 表示一个带有预处理和验证的用户记录
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
        fmt.Printf("错误：%v\n", err)
        return
    }

    fmt.Printf("处理完成：%d 行中 %d 行有效\n", result.RowCount, result.ValidRowCount)

    for _, user := range users {
        fmt.Printf("姓名：%q，邮箱：%q\n", user.Name, user.Email)
    }

    // reader 可以直接传递给 filesql
    _ = reader
}
```

输出：
```
处理完成：2 行中 2 行有效
姓名："John Doe"，邮箱："john@example.com"
姓名："Jane Smith"，邮箱："jane@example.com"
```

## 预处理标签（`prep`）

可以组合多个标签：`prep:"trim,lowercase,default=N/A"`

### 基本预处理器

| 标签 | 描述 | 示例 |
|------|------|------|
| `trim` | 删除前后空白 | `prep:"trim"` |
| `ltrim` | 删除前导空白 | `prep:"ltrim"` |
| `rtrim` | 删除尾部空白 | `prep:"rtrim"` |
| `lowercase` | 转换为小写 | `prep:"lowercase"` |
| `uppercase` | 转换为大写 | `prep:"uppercase"` |
| `default=value` | 如果为空则设置默认值 | `prep:"default=N/A"` |

### 字符串转换

| 标签 | 描述 | 示例 |
|------|------|------|
| `replace=old:new` | 替换所有出现 | `prep:"replace=;:,"` |
| `prefix=value` | 在开头添加字符串 | `prep:"prefix=ID_"` |
| `suffix=value` | 在结尾添加字符串 | `prep:"suffix=_END"` |
| `truncate=N` | 限制为 N 个字符 | `prep:"truncate=100"` |
| `strip_html` | 删除 HTML 标签 | `prep:"strip_html"` |
| `strip_newline` | 删除换行符（LF、CRLF、CR） | `prep:"strip_newline"` |
| `collapse_space` | 将多个空格压缩为一个 | `prep:"collapse_space"` |

### 字符过滤

| 标签 | 描述 | 示例 |
|------|------|------|
| `remove_digits` | 删除所有数字 | `prep:"remove_digits"` |
| `remove_alpha` | 删除所有字母 | `prep:"remove_alpha"` |
| `keep_digits` | 只保留数字 | `prep:"keep_digits"` |
| `keep_alpha` | 只保留字母 | `prep:"keep_alpha"` |
| `trim_set=chars` | 从两端删除指定字符 | `prep:"trim_set=@#$"` |

### 填充

| 标签 | 描述 | 示例 |
|------|------|------|
| `pad_left=N:char` | 左填充至 N 个字符 | `prep:"pad_left=5:0"` |
| `pad_right=N:char` | 右填充至 N 个字符 | `prep:"pad_right=10: "` |

### 高级预处理

| 标签 | 描述 | 示例 |
|------|------|------|
| `normalize_unicode` | 将 Unicode 规范化为 NFC 格式 | `prep:"normalize_unicode"` |
| `nullify=value` | 将特定字符串视为空 | `prep:"nullify=NULL"` |
| `coerce=type` | 类型转换（int、float、bool） | `prep:"coerce=int"` |
| `fix_scheme=scheme` | 添加/修复 URL 方案 | `prep:"fix_scheme=https"` |
| `regex_replace=pattern:replacement` | 正则表达式替换 | `prep:"regex_replace=\\d+:X"` |

## 验证标签（`validate`）

可以组合多个标签：`validate:"required,email"`

### 基本验证器

| 标签 | 描述 | 示例 |
|------|------|------|
| `required` | 字段不能为空 | `validate:"required"` |
| `boolean` | 必须是 true、false、0 或 1 | `validate:"boolean"` |

### 格式验证器

| 标签 | 描述 | 示例 |
|------|------|------|
| `email` | 有效的电子邮件地址 | `validate:"email"` |
| `uri` | 有效的 URI | `validate:"uri"` |
| `url` | 有效的 URL | `validate:"url"` |
| `uuid` | 有效的 UUID | `validate:"uuid"` |

### 跨字段验证器

| 标签 | 描述 | 示例 |
|------|------|------|
| `eqfield=Field` | 值等于另一个字段 | `validate:"eqfield=Password"` |
| `nefield=Field` | 值不等于另一个字段 | `validate:"nefield=OldPassword"` |
| `gtfield=Field` | 值大于另一个字段 | `validate:"gtfield=MinPrice"` |
| `ltfield=Field` | 值小于另一个字段 | `validate:"ltfield=MaxPrice"` |

## 支持的文件格式

| 格式 | 扩展名 | 压缩格式 |
|------|--------|----------|
| CSV | `.csv` | `.csv.gz`、`.csv.bz2`、`.csv.xz`、`.csv.zst` |
| TSV | `.tsv` | `.tsv.gz`、`.tsv.bz2`、`.tsv.xz`、`.tsv.zst` |
| LTSV | `.ltsv` | `.ltsv.gz`、`.ltsv.bz2`、`.ltsv.xz`、`.ltsv.zst` |
| Excel | `.xlsx` | `.xlsx.gz`、`.xlsx.bz2`、`.xlsx.xz`、`.xlsx.zst` |
| Parquet | `.parquet` | `.parquet.gz`、`.parquet.bz2`、`.parquet.xz`、`.parquet.zst` |

**Excel 文件说明**：只处理**第一个工作表**。多工作表工作簿中的后续工作表将被忽略。

## 内存使用

fileprep 将**整个文件加载到内存**中进行处理。这允许随机访问和多遍操作，但对大文件有影响：

| 文件大小 | 预计内存 | 建议 |
|----------|----------|------|
| < 100 MB | 约为文件大小的 2-3 倍 | 直接处理 |
| 100-500 MB | 500 MB - 1.5 GB | 监控内存，考虑分块处理 |
| > 500 MB | > 1.5 GB | 拆分文件或使用流式替代方案 |

## 相关项目

- [nao1215/filesql](https://github.com/nao1215/filesql) - CSV、TSV、LTSV、Parquet、Excel 的 SQL 驱动
- [nao1215/csv](https://github.com/nao1215/csv) - 带验证的 CSV 读取和简单 DataFrame

## 贡献

欢迎贡献！详情请参阅 [Contributing Guide](../../CONTRIBUTING.md)。

## 支持

如果您觉得这个项目有用，请考虑：

- 在 GitHub 上给个星 - 这有助于其他人发现这个项目
- [成为赞助者](https://github.com/sponsors/nao1215) - 您的支持帮助维护项目并激励持续开发

## 许可证

本项目采用 MIT 许可证 - 详情请参阅 [LICENSE](../../LICENSE) 文件。
