# fileprep

[![Go Reference](https://pkg.go.dev/badge/github.com/nao1215/fileprep.svg)](https://pkg.go.dev/github.com/nao1215/fileprep)
[![Go Report Card](https://goreportcard.com/badge/github.com/nao1215/fileprep)](https://goreportcard.com/report/github.com/nao1215/fileprep)
[![MultiPlatformUnitTest](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml/badge.svg)](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml)
![Coverage](https://raw.githubusercontent.com/nao1215/octocovs-central-repo/main/badges/nao1215/fileprep/coverage.svg)

[English](../../README.md) | [日本語](../ja/README.md) | [Français](../fr/README.md) | [한국어](../ko/README.md) | [Русский](../ru/README.md) | [中文](../zh-cn/README.md)

![fileprep-logo](../images/fileprep-logo-small.png)

**fileprep** es una biblioteca de Go para limpiar, normalizar y validar datos estructurados (CSV, TSV, LTSV, Parquet y Excel) mediante reglas ligeras basadas en etiquetas struct, con soporte transparente para flujos gzip, bzip2, xz y zstd.

## ¿Por qué fileprep?

Desarrollé [nao1215/filesql](https://github.com/nao1215/filesql), que permite ejecutar consultas SQL en archivos como CSV, TSV, LTSV, Parquet y Excel. También creé [nao1215/csv](https://github.com/nao1215/csv) para la validación de archivos CSV.

Mientras estudiaba aprendizaje automático, me di cuenta: "Si extiendo [nao1215/csv](https://github.com/nao1215/csv) para soportar los mismos formatos de archivo que [nao1215/filesql](https://github.com/nao1215/filesql), podría combinarlos para realizar operaciones tipo ETL". Esta idea llevó a la creación de **fileprep**: una biblioteca que conecta el preprocesamiento/validación de datos con las consultas SQL basadas en archivos.

## Características

- Soporte de múltiples formatos: CSV, TSV, LTSV, Parquet, Excel (.xlsx)
- Soporte de compresión: gzip (.gz), bzip2 (.bz2), xz (.xz), zstd (.zst)
- Vinculación de columnas por nombre: Los campos coinciden automáticamente con nombres de columna en `snake_case`, personalizable con la etiqueta `name`
- Preprocesamiento basado en etiquetas struct (`prep`): trim, lowercase, uppercase, valores por defecto
- Validación basada en etiquetas struct (`validate`): required y más
- Integración con [filesql](https://github.com/nao1215/filesql): Devuelve `io.Reader` para uso directo con filesql
- Informe detallado de errores: Información de fila y columna para cada error

## Instalación

```bash
go get github.com/nao1215/fileprep
```

## Requisitos

- Versión de Go: 1.24 o posterior
- Sistemas Operativos:
  - Linux
  - macOS
  - Windows

## Inicio Rápido

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/nao1215/fileprep"
)

// User representa un registro de usuario con preprocesamiento y validación
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

    fmt.Printf("Procesadas %d filas, %d válidas\n", result.RowCount, result.ValidRowCount)

    for _, user := range users {
        fmt.Printf("Nombre: %q, Email: %q\n", user.Name, user.Email)
    }

    // reader puede pasarse directamente a filesql
    _ = reader
}
```

Salida:
```
Procesadas 2 filas, 2 válidas
Nombre: "John Doe", Email: "john@example.com"
Nombre: "Jane Smith", Email: "jane@example.com"
```

## Etiquetas de Preprocesamiento (`prep`)

Se pueden combinar múltiples etiquetas: `prep:"trim,lowercase,default=N/A"`

### Preprocesadores Básicos

| Etiqueta | Descripción | Ejemplo |
|----------|-------------|---------|
| `trim` | Eliminar espacios al inicio/final | `prep:"trim"` |
| `ltrim` | Eliminar espacios al inicio | `prep:"ltrim"` |
| `rtrim` | Eliminar espacios al final | `prep:"rtrim"` |
| `lowercase` | Convertir a minúsculas | `prep:"lowercase"` |
| `uppercase` | Convertir a mayúsculas | `prep:"uppercase"` |
| `default=value` | Establecer valor por defecto si está vacío | `prep:"default=N/A"` |

### Transformación de Cadenas

| Etiqueta | Descripción | Ejemplo |
|----------|-------------|---------|
| `replace=old:new` | Reemplazar todas las ocurrencias | `prep:"replace=;:,"` |
| `prefix=value` | Agregar cadena al inicio | `prep:"prefix=ID_"` |
| `suffix=value` | Agregar cadena al final | `prep:"suffix=_END"` |
| `truncate=N` | Limitar a N caracteres | `prep:"truncate=100"` |
| `strip_html` | Eliminar etiquetas HTML | `prep:"strip_html"` |
| `strip_newline` | Eliminar saltos de línea | `prep:"strip_newline"` |
| `collapse_space` | Colapsar múltiples espacios en uno | `prep:"collapse_space"` |

## Etiquetas de Validación (`validate`)

Se pueden combinar múltiples etiquetas: `validate:"required,email"`

### Validadores Básicos

| Etiqueta | Descripción | Ejemplo |
|----------|-------------|---------|
| `required` | El campo no debe estar vacío | `validate:"required"` |
| `boolean` | Debe ser true, false, 0 o 1 | `validate:"boolean"` |
| `email` | Dirección de correo válida | `validate:"email"` |
| `url` | URL válida | `validate:"url"` |
| `uuid` | UUID válido | `validate:"uuid"` |

### Validadores de Campo Cruzado

| Etiqueta | Descripción | Ejemplo |
|----------|-------------|---------|
| `eqfield=Field` | Valor igual a otro campo | `validate:"eqfield=Password"` |
| `nefield=Field` | Valor diferente a otro campo | `validate:"nefield=OldPassword"` |

## Formatos de Archivo Soportados

| Formato | Extensión | Comprimido |
|---------|-----------|------------|
| CSV | `.csv` | `.csv.gz`, `.csv.bz2`, `.csv.xz`, `.csv.zst` |
| TSV | `.tsv` | `.tsv.gz`, `.tsv.bz2`, `.tsv.xz`, `.tsv.zst` |
| LTSV | `.ltsv` | `.ltsv.gz`, `.ltsv.bz2`, `.ltsv.xz`, `.ltsv.zst` |
| Excel | `.xlsx` | `.xlsx.gz`, `.xlsx.bz2`, `.xlsx.xz`, `.xlsx.zst` |
| Parquet | `.parquet` | `.parquet.gz`, `.parquet.bz2`, `.parquet.xz`, `.parquet.zst` |

**Nota sobre archivos Excel**: Solo se procesa la **primera hoja**. Las hojas siguientes en libros de trabajo de múltiples hojas serán ignoradas.

## Uso de Memoria

fileprep carga el **archivo completo en memoria** para su procesamiento. Esto permite acceso aleatorio y operaciones de múltiples pasadas, pero tiene implicaciones para archivos grandes:

| Tamaño de Archivo | Memoria Aprox. | Recomendación |
|-------------------|----------------|---------------|
| < 100 MB | ~2-3x tamaño del archivo | Procesamiento directo |
| 100-500 MB | ~500 MB - 1.5 GB | Monitorear memoria, considerar fragmentación |
| > 500 MB | > 1.5 GB | Dividir archivos o usar alternativas de streaming |

## Proyectos Relacionados

- [nao1215/filesql](https://github.com/nao1215/filesql) - Driver SQL para CSV, TSV, LTSV, Parquet, Excel
- [nao1215/csv](https://github.com/nao1215/csv) - Lectura de CSV con validación y DataFrame simple

## Contribuciones

¡Las contribuciones son bienvenidas! Por favor consulte la [Guía de Contribución](../../CONTRIBUTING.md) para más detalles.

## Soporte

Si encuentra útil este proyecto, por favor considere:

- Darle una estrella en GitHub - ayuda a otros a descubrir el proyecto
- [Convertirse en patrocinador](https://github.com/sponsors/nao1215) - su apoyo mantiene vivo el proyecto

## Licencia

Este proyecto está licenciado bajo la Licencia MIT - vea el archivo [LICENSE](../../LICENSE) para más detalles.
