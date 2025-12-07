# fileprep

[![Go Reference](https://pkg.go.dev/badge/github.com/nao1215/fileprep.svg)](https://pkg.go.dev/github.com/nao1215/fileprep)
[![Go Report Card](https://goreportcard.com/badge/github.com/nao1215/fileprep)](https://goreportcard.com/report/github.com/nao1215/fileprep)
[![MultiPlatformUnitTest](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml/badge.svg)](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml)
![Coverage](https://raw.githubusercontent.com/nao1215/octocovs-central-repo/main/badges/nao1215/fileprep/coverage.svg)

[English](../../README.md) | [日本語](../ja/README.md) | [Español](../es/README.md) | [Français](../fr/README.md) | [Русский](../ru/README.md) | [中文](../zh-cn/README.md)

![fileprep-logo](../images/fileprep-logo-small.png)

**fileprep**은 CSV, TSV, LTSV, Parquet, Excel 등의 구조화된 데이터를 경량 struct 태그 규칙으로 정리, 정규화, 검증하기 위한 Go 라이브러리입니다. gzip, bzip2, xz, zstd 스트림을 원활하게 지원합니다.

## 왜 fileprep인가?

저는 CSV, TSV, LTSV, Parquet, Excel 파일에 SQL 쿼리를 실행할 수 있는 [nao1215/filesql](https://github.com/nao1215/filesql)을 개발했습니다. 또한 CSV 파일 검증을 위해 [nao1215/csv](https://github.com/nao1215/csv)도 만들었습니다.

머신러닝을 공부하면서, "[nao1215/csv](https://github.com/nao1215/csv)를 [nao1215/filesql](https://github.com/nao1215/filesql)과 동일한 파일 형식으로 확장하면, 둘을 결합하여 ETL과 같은 작업을 수행할 수 있다"는 것을 깨달았습니다. 이 아이디어가 **fileprep**의 탄생으로 이어졌습니다: 데이터 전처리/검증과 SQL 기반 파일 쿼리를 연결하는 라이브러리입니다.

## 기능

- 다중 파일 형식 지원: CSV, TSV, LTSV, Parquet, Excel (.xlsx)
- 압축 지원: gzip (.gz), bzip2 (.bz2), xz (.xz), zstd (.zst)
- 이름 기반 컬럼 바인딩: 필드는 자동으로 `snake_case` 컬럼명에 매칭, `name` 태그로 커스터마이즈 가능
- struct 태그 기반 전처리 (`prep` 태그): trim, lowercase, uppercase, 기본값 등
- struct 태그 기반 검증 (`validate` 태그): required 등
- [filesql](https://github.com/nao1215/filesql)과의 원활한 통합: filesql에서 직접 사용할 수 있는 `io.Reader` 반환
- 상세한 에러 보고: 각 에러의 행과 열 정보

## 설치

```bash
go get github.com/nao1215/fileprep
```

## 요구 사항

- Go 버전: 1.24 이상
- 지원 OS:
  - Linux
  - macOS
  - Windows

## 빠른 시작

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/nao1215/fileprep"
)

// User는 전처리 및 검증이 포함된 사용자 레코드를 나타냅니다
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

    fmt.Printf("처리 완료: %d행 중 %d행이 유효\n", result.RowCount, result.ValidRowCount)

    for _, user := range users {
        fmt.Printf("Name: %q, Email: %q\n", user.Name, user.Email)
    }

    // reader는 filesql에 직접 전달할 수 있습니다
    _ = reader
}
```

출력:
```
처리 완료: 2행 중 2행이 유효
Name: "John Doe", Email: "john@example.com"
Name: "Jane Smith", Email: "jane@example.com"
```

## 전처리 태그 (`prep`)

여러 태그를 조합할 수 있습니다: `prep:"trim,lowercase,default=N/A"`

### 기본 전처리

| 태그 | 설명 | 예시 |
|------|------|------|
| `trim` | 앞뒤 공백 제거 | `prep:"trim"` |
| `ltrim` | 앞쪽 공백 제거 | `prep:"ltrim"` |
| `rtrim` | 뒤쪽 공백 제거 | `prep:"rtrim"` |
| `lowercase` | 소문자로 변환 | `prep:"lowercase"` |
| `uppercase` | 대문자로 변환 | `prep:"uppercase"` |
| `default=value` | 비어있을 경우 기본값 설정 | `prep:"default=N/A"` |

### 문자열 변환

| 태그 | 설명 | 예시 |
|------|------|------|
| `replace=old:new` | 모든 항목 치환 | `prep:"replace=;:,"` |
| `prefix=value` | 문자열 앞에 추가 | `prep:"prefix=ID_"` |
| `suffix=value` | 문자열 뒤에 추가 | `prep:"suffix=_END"` |
| `truncate=N` | N자로 제한 | `prep:"truncate=100"` |
| `strip_html` | HTML 태그 제거 | `prep:"strip_html"` |
| `strip_newline` | 줄바꿈 제거 (LF, CRLF, CR) | `prep:"strip_newline"` |
| `collapse_space` | 연속 공백을 하나로 축소 | `prep:"collapse_space"` |

### 문자 필터링

| 태그 | 설명 | 예시 |
|------|------|------|
| `remove_digits` | 모든 숫자 제거 | `prep:"remove_digits"` |
| `remove_alpha` | 모든 알파벳 제거 | `prep:"remove_alpha"` |
| `keep_digits` | 숫자만 유지 | `prep:"keep_digits"` |
| `keep_alpha` | 알파벳만 유지 | `prep:"keep_alpha"` |
| `trim_set=chars` | 지정 문자를 양끝에서 제거 | `prep:"trim_set=@#$"` |

### 패딩

| 태그 | 설명 | 예시 |
|------|------|------|
| `pad_left=N:char` | N자까지 왼쪽 패딩 | `prep:"pad_left=5:0"` |
| `pad_right=N:char` | N자까지 오른쪽 패딩 | `prep:"pad_right=10: "` |

### 고급 전처리

| 태그 | 설명 | 예시 |
|------|------|------|
| `normalize_unicode` | Unicode를 NFC 형식으로 정규화 | `prep:"normalize_unicode"` |
| `nullify=value` | 특정 문자열을 빈 값으로 처리 | `prep:"nullify=NULL"` |
| `coerce=type` | 타입 변환 (int, float, bool) | `prep:"coerce=int"` |
| `fix_scheme=scheme` | URL 스킴 추가/수정 | `prep:"fix_scheme=https"` |
| `regex_replace=pattern:replacement` | 정규식으로 치환 | `prep:"regex_replace=\\d+:X"` |

## 검증 태그 (`validate`)

여러 태그를 조합할 수 있습니다: `validate:"required,email"`

### 기본 검증자

| 태그 | 설명 | 예시 |
|------|------|------|
| `required` | 필드가 비어있으면 안 됨 | `validate:"required"` |
| `boolean` | true, false, 0, 또는 1이어야 함 | `validate:"boolean"` |

### 형식 검증자

| 태그 | 설명 | 예시 |
|------|------|------|
| `email` | 유효한 이메일 주소 | `validate:"email"` |
| `uri` | 유효한 URI | `validate:"uri"` |
| `url` | 유효한 URL | `validate:"url"` |
| `uuid` | 유효한 UUID | `validate:"uuid"` |

### 필드 간 검증자

| 태그 | 설명 | 예시 |
|------|------|------|
| `eqfield=Field` | 다른 필드와 값이 같음 | `validate:"eqfield=Password"` |
| `nefield=Field` | 다른 필드와 값이 다름 | `validate:"nefield=OldPassword"` |
| `gtfield=Field` | 다른 필드보다 큼 | `validate:"gtfield=MinPrice"` |
| `ltfield=Field` | 다른 필드보다 작음 | `validate:"ltfield=MaxPrice"` |

## 지원 파일 형식

| 형식 | 확장자 | 압축 형식 |
|------|--------|----------|
| CSV | `.csv` | `.csv.gz`, `.csv.bz2`, `.csv.xz`, `.csv.zst` |
| TSV | `.tsv` | `.tsv.gz`, `.tsv.bz2`, `.tsv.xz`, `.tsv.zst` |
| LTSV | `.ltsv` | `.ltsv.gz`, `.ltsv.bz2`, `.ltsv.xz`, `.ltsv.zst` |
| Excel | `.xlsx` | `.xlsx.gz`, `.xlsx.bz2`, `.xlsx.xz`, `.xlsx.zst` |
| Parquet | `.parquet` | `.parquet.gz`, `.parquet.bz2`, `.parquet.xz`, `.parquet.zst` |

**Excel 파일 참고**: **첫 번째 시트**만 처리됩니다. 여러 시트가 있는 워크북에서는 이후 시트가 무시됩니다.

## 메모리 사용량

fileprep은 처리를 위해 **전체 파일을 메모리에 로드**합니다. 이를 통해 랜덤 액세스와 다중 패스 연산이 가능하지만, 대용량 파일에는 영향이 있습니다:

| 파일 크기 | 예상 메모리 | 권장 사항 |
|-----------|-------------|-----------|
| < 100 MB | 파일 크기의 약 2-3배 | 직접 처리 |
| 100-500 MB | 500 MB - 1.5 GB | 메모리 모니터링, 청크 처리 고려 |
| > 500 MB | > 1.5 GB | 파일 분할 또는 스트리밍 대안 사용 |

## 관련 프로젝트

- [nao1215/filesql](https://github.com/nao1215/filesql) - CSV, TSV, LTSV, Parquet, Excel용 SQL 드라이버
- [nao1215/csv](https://github.com/nao1215/csv) - 검증 기능이 있는 CSV 읽기 및 간단한 DataFrame

## 기여

기여를 환영합니다! 자세한 내용은 [Contributing Guide](../../CONTRIBUTING.md)를 참조하세요.

## 지원

이 프로젝트가 유용하다면 다음을 고려해 주세요:

- GitHub에서 스타 주기 - 다른 사람들이 프로젝트를 발견하는 데 도움이 됩니다
- [스폰서 되기](https://github.com/sponsors/nao1215) - 여러분의 지원이 프로젝트를 유지하고 지속적인 개발의 동기가 됩니다

## 라이선스

이 프로젝트는 MIT 라이선스 하에 배포됩니다 - 자세한 내용은 [LICENSE](../../LICENSE) 파일을 참조하세요.
