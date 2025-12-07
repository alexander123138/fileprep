# fileprep

[![Go Reference](https://pkg.go.dev/badge/github.com/nao1215/fileprep.svg)](https://pkg.go.dev/github.com/nao1215/fileprep)
[![Go Report Card](https://goreportcard.com/badge/github.com/nao1215/fileprep)](https://goreportcard.com/report/github.com/nao1215/fileprep)
[![MultiPlatformUnitTest](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml/badge.svg)](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml)
![Coverage](https://raw.githubusercontent.com/nao1215/octocovs-central-repo/main/badges/nao1215/fileprep/coverage.svg)

[English](../../README.md) | [日本語](../ja/README.md) | [Español](../es/README.md) | [Français](../fr/README.md) | [한국어](../ko/README.md) | [中文](../zh-cn/README.md)

![fileprep-logo](../images/fileprep-logo-small.png)

**fileprep** — это библиотека Go для очистки, нормализации и валидации структурированных данных (CSV, TSV, LTSV, Parquet и Excel) с использованием лёгких правил на основе struct-тегов. Поддерживает прозрачную работу с потоками gzip, bzip2, xz и zstd.

## Почему fileprep?

Я разработал [nao1215/filesql](https://github.com/nao1215/filesql), который позволяет выполнять SQL-запросы к файлам CSV, TSV, LTSV, Parquet и Excel. Также я создал [nao1215/csv](https://github.com/nao1215/csv) для валидации CSV-файлов.

Изучая машинное обучение, я понял: «Если расширить [nao1215/csv](https://github.com/nao1215/csv) для поддержки тех же форматов файлов, что и [nao1215/filesql](https://github.com/nao1215/filesql), можно объединить их для выполнения ETL-подобных операций». Эта идея привела к созданию **fileprep**: библиотеки, соединяющей предобработку/валидацию данных с SQL-запросами к файлам.

## Возможности

- Поддержка множества форматов: CSV, TSV, LTSV, Parquet, Excel (.xlsx)
- Поддержка сжатия: gzip (.gz), bzip2 (.bz2), xz (.xz), zstd (.zst)
- Привязка колонок по имени: поля автоматически соответствуют именам колонок в `snake_case`, настраивается через тег `name`
- Предобработка на основе struct-тегов (`prep`): trim, lowercase, uppercase, значения по умолчанию
- Валидация на основе struct-тегов (`validate`): required и другие
- Интеграция с [filesql](https://github.com/nao1215/filesql): возвращает `io.Reader` для прямого использования с filesql
- Детальные отчёты об ошибках: информация о строке и колонке для каждой ошибки

## Установка

```bash
go get github.com/nao1215/fileprep
```

## Требования

- Версия Go: 1.24 или выше
- Операционные системы:
  - Linux
  - macOS
  - Windows

## Быстрый старт

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/nao1215/fileprep"
)

// User представляет запись пользователя с предобработкой и валидацией
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
        fmt.Printf("Ошибка: %v\n", err)
        return
    }

    fmt.Printf("Обработано %d строк, %d валидных\n", result.RowCount, result.ValidRowCount)

    for _, user := range users {
        fmt.Printf("Имя: %q, Email: %q\n", user.Name, user.Email)
    }

    // reader можно передать напрямую в filesql
    _ = reader
}
```

Вывод:
```
Обработано 2 строк, 2 валидных
Имя: "John Doe", Email: "john@example.com"
Имя: "Jane Smith", Email: "jane@example.com"
```

## Теги предобработки (`prep`)

Можно комбинировать несколько тегов: `prep:"trim,lowercase,default=N/A"`

### Базовые препроцессоры

| Тег | Описание | Пример |
|-----|----------|--------|
| `trim` | Удалить пробелы в начале/конце | `prep:"trim"` |
| `ltrim` | Удалить пробелы в начале | `prep:"ltrim"` |
| `rtrim` | Удалить пробелы в конце | `prep:"rtrim"` |
| `lowercase` | Преобразовать в нижний регистр | `prep:"lowercase"` |
| `uppercase` | Преобразовать в верхний регистр | `prep:"uppercase"` |
| `default=value` | Установить значение по умолчанию, если пусто | `prep:"default=N/A"` |

### Преобразование строк

| Тег | Описание | Пример |
|-----|----------|--------|
| `replace=old:new` | Заменить все вхождения | `prep:"replace=;:,"` |
| `prefix=value` | Добавить строку в начало | `prep:"prefix=ID_"` |
| `suffix=value` | Добавить строку в конец | `prep:"suffix=_END"` |
| `truncate=N` | Ограничить N символами | `prep:"truncate=100"` |
| `strip_html` | Удалить HTML-теги | `prep:"strip_html"` |
| `strip_newline` | Удалить переносы строк (LF, CRLF, CR) | `prep:"strip_newline"` |
| `collapse_space` | Сжать множественные пробелы в один | `prep:"collapse_space"` |

### Фильтрация символов

| Тег | Описание | Пример |
|-----|----------|--------|
| `remove_digits` | Удалить все цифры | `prep:"remove_digits"` |
| `remove_alpha` | Удалить все буквы | `prep:"remove_alpha"` |
| `keep_digits` | Оставить только цифры | `prep:"keep_digits"` |
| `keep_alpha` | Оставить только буквы | `prep:"keep_alpha"` |
| `trim_set=chars` | Удалить указанные символы с обоих концов | `prep:"trim_set=@#$"` |

### Выравнивание

| Тег | Описание | Пример |
|-----|----------|--------|
| `pad_left=N:char` | Дополнить слева до N символов | `prep:"pad_left=5:0"` |
| `pad_right=N:char` | Дополнить справа до N символов | `prep:"pad_right=10: "` |

### Расширенная предобработка

| Тег | Описание | Пример |
|-----|----------|--------|
| `normalize_unicode` | Нормализовать Unicode в формат NFC | `prep:"normalize_unicode"` |
| `nullify=value` | Считать определённую строку пустой | `prep:"nullify=NULL"` |
| `coerce=type` | Преобразование типа (int, float, bool) | `prep:"coerce=int"` |
| `fix_scheme=scheme` | Добавить/исправить схему URL | `prep:"fix_scheme=https"` |
| `regex_replace=pattern:replacement` | Замена по регулярному выражению | `prep:"regex_replace=\\d+:X"` |

## Теги валидации (`validate`)

Можно комбинировать несколько тегов: `validate:"required,email"`

### Базовые валидаторы

| Тег | Описание | Пример |
|-----|----------|--------|
| `required` | Поле не должно быть пустым | `validate:"required"` |
| `boolean` | Должно быть true, false, 0 или 1 | `validate:"boolean"` |

### Валидаторы формата

| Тег | Описание | Пример |
|-----|----------|--------|
| `email` | Валидный email-адрес | `validate:"email"` |
| `uri` | Валидный URI | `validate:"uri"` |
| `url` | Валидный URL | `validate:"url"` |
| `uuid` | Валидный UUID | `validate:"uuid"` |

### Межполевые валидаторы

| Тег | Описание | Пример |
|-----|----------|--------|
| `eqfield=Field` | Значение равно другому полю | `validate:"eqfield=Password"` |
| `nefield=Field` | Значение не равно другому полю | `validate:"nefield=OldPassword"` |
| `gtfield=Field` | Значение больше другого поля | `validate:"gtfield=MinPrice"` |
| `ltfield=Field` | Значение меньше другого поля | `validate:"ltfield=MaxPrice"` |

## Поддерживаемые форматы файлов

| Формат | Расширение | Сжатые форматы |
|--------|------------|----------------|
| CSV | `.csv` | `.csv.gz`, `.csv.bz2`, `.csv.xz`, `.csv.zst` |
| TSV | `.tsv` | `.tsv.gz`, `.tsv.bz2`, `.tsv.xz`, `.tsv.zst` |
| LTSV | `.ltsv` | `.ltsv.gz`, `.ltsv.bz2`, `.ltsv.xz`, `.ltsv.zst` |
| Excel | `.xlsx` | `.xlsx.gz`, `.xlsx.bz2`, `.xlsx.xz`, `.xlsx.zst` |
| Parquet | `.parquet` | `.parquet.gz`, `.parquet.bz2`, `.parquet.xz`, `.parquet.zst` |

**Примечание о файлах Excel**: Обрабатывается только **первый лист**. Последующие листы в многолистовых книгах будут проигнорированы.

## Использование памяти

fileprep загружает **весь файл в память** для обработки. Это позволяет произвольный доступ и многопроходные операции, но имеет последствия для больших файлов:

| Размер файла | Примерная память | Рекомендация |
|--------------|------------------|--------------|
| < 100 МБ | ~2-3x размера файла | Прямая обработка |
| 100-500 МБ | 500 МБ - 1.5 ГБ | Мониторинг памяти, рассмотреть разбиение |
| > 500 МБ | > 1.5 ГБ | Разделить файлы или использовать потоковые альтернативы |

## Связанные проекты

- [nao1215/filesql](https://github.com/nao1215/filesql) - SQL-драйвер для CSV, TSV, LTSV, Parquet, Excel
- [nao1215/csv](https://github.com/nao1215/csv) - Чтение CSV с валидацией и простой DataFrame

## Участие в разработке

Мы приветствуем вклад в проект! Подробнее см. [Contributing Guide](../../CONTRIBUTING.md).

## Поддержка

Если вы находите этот проект полезным, пожалуйста, рассмотрите:

- Поставить звезду на GitHub — это помогает другим найти проект
- [Стать спонсором](https://github.com/sponsors/nao1215) — ваша поддержка помогает поддерживать проект и мотивирует на дальнейшую разработку

## Лицензия

Этот проект лицензирован под лицензией MIT — подробности см. в файле [LICENSE](../../LICENSE).
