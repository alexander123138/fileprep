# fileprep

[![Go Reference](https://pkg.go.dev/badge/github.com/nao1215/fileprep.svg)](https://pkg.go.dev/github.com/nao1215/fileprep)
[![Go Report Card](https://goreportcard.com/badge/github.com/nao1215/fileprep)](https://goreportcard.com/report/github.com/nao1215/fileprep)
[![MultiPlatformUnitTest](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml/badge.svg)](https://github.com/nao1215/fileprep/actions/workflows/unit_test.yml)
![Coverage](https://raw.githubusercontent.com/nao1215/octocovs-central-repo/main/badges/nao1215/fileprep/coverage.svg)

[English](../../README.md) | [日本語](../ja/README.md) | [Español](../es/README.md) | [한국어](../ko/README.md) | [Русский](../ru/README.md) | [中文](../zh-cn/README.md)

![fileprep-logo](../images/fileprep-logo-small.png)

**fileprep** est une bibliothèque Go pour nettoyer, normaliser et valider des données structurées (CSV, TSV, LTSV, Parquet et Excel) via des règles légères basées sur les balises struct, avec un support transparent des flux gzip, bzip2, xz et zstd.

## Pourquoi fileprep ?

J'ai développé [nao1215/filesql](https://github.com/nao1215/filesql), qui permet d'exécuter des requêtes SQL sur des fichiers comme CSV, TSV, LTSV, Parquet et Excel. J'ai également créé [nao1215/csv](https://github.com/nao1215/csv) pour la validation des fichiers CSV.

En étudiant l'apprentissage automatique, j'ai réalisé : "Si j'étends [nao1215/csv](https://github.com/nao1215/csv) pour supporter les mêmes formats de fichiers que [nao1215/filesql](https://github.com/nao1215/filesql), je pourrais les combiner pour effectuer des opérations de type ETL". Cette idée a conduit à la création de **fileprep** : une bibliothèque qui fait le pont entre le prétraitement/validation des données et les requêtes SQL sur fichiers.

## Fonctionnalités

- Support multi-formats : CSV, TSV, LTSV, Parquet, Excel (.xlsx)
- Support de la compression : gzip (.gz), bzip2 (.bz2), xz (.xz), zstd (.zst)
- Liaison de colonnes par nom : Les champs correspondent automatiquement aux noms de colonnes en `snake_case`, personnalisable via la balise `name`
- Prétraitement basé sur les balises struct (`prep`) : trim, lowercase, uppercase, valeurs par défaut
- Validation basée sur les balises struct (`validate`) : required et plus
- Intégration [filesql](https://github.com/nao1215/filesql) : Retourne `io.Reader` pour une utilisation directe avec filesql
- Rapport d'erreurs détaillé : Informations de ligne et colonne pour chaque erreur

## Installation

```bash
go get github.com/nao1215/fileprep
```

## Prérequis

- Version Go : 1.24 ou ultérieure
- Systèmes d'exploitation :
  - Linux
  - macOS
  - Windows

## Démarrage Rapide

```go
package main

import (
    "fmt"
    "os"
    "strings"

    "github.com/nao1215/fileprep"
)

// User représente un enregistrement utilisateur avec prétraitement et validation
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
        fmt.Printf("Erreur : %v\n", err)
        return
    }

    fmt.Printf("Traité %d lignes, %d valides\n", result.RowCount, result.ValidRowCount)

    for _, user := range users {
        fmt.Printf("Nom : %q, Email : %q\n", user.Name, user.Email)
    }

    // reader peut être passé directement à filesql
    _ = reader
}
```

Sortie :
```
Traité 2 lignes, 2 valides
Nom : "John Doe", Email : "john@example.com"
Nom : "Jane Smith", Email : "jane@example.com"
```

## Balises de Prétraitement (`prep`)

Plusieurs balises peuvent être combinées : `prep:"trim,lowercase,default=N/A"`

### Préprocesseurs de Base

| Balise | Description | Exemple |
|--------|-------------|---------|
| `trim` | Supprimer les espaces au début/fin | `prep:"trim"` |
| `ltrim` | Supprimer les espaces au début | `prep:"ltrim"` |
| `rtrim` | Supprimer les espaces à la fin | `prep:"rtrim"` |
| `lowercase` | Convertir en minuscules | `prep:"lowercase"` |
| `uppercase` | Convertir en majuscules | `prep:"uppercase"` |
| `default=value` | Définir une valeur par défaut si vide | `prep:"default=N/A"` |

### Transformation de Chaînes

| Balise | Description | Exemple |
|--------|-------------|---------|
| `replace=old:new` | Remplacer toutes les occurrences | `prep:"replace=;:,"` |
| `prefix=value` | Ajouter une chaîne au début | `prep:"prefix=ID_"` |
| `suffix=value` | Ajouter une chaîne à la fin | `prep:"suffix=_END"` |
| `truncate=N` | Limiter à N caractères | `prep:"truncate=100"` |
| `strip_html` | Supprimer les balises HTML | `prep:"strip_html"` |
| `strip_newline` | Supprimer les sauts de ligne | `prep:"strip_newline"` |
| `collapse_space` | Réduire les espaces multiples en un seul | `prep:"collapse_space"` |

## Balises de Validation (`validate`)

Plusieurs balises peuvent être combinées : `validate:"required,email"`

### Validateurs de Base

| Balise | Description | Exemple |
|--------|-------------|---------|
| `required` | Le champ ne doit pas être vide | `validate:"required"` |
| `boolean` | Doit être true, false, 0 ou 1 | `validate:"boolean"` |
| `email` | Adresse email valide | `validate:"email"` |
| `url` | URL valide | `validate:"url"` |
| `uuid` | UUID valide | `validate:"uuid"` |

### Validateurs Inter-champs

| Balise | Description | Exemple |
|--------|-------------|---------|
| `eqfield=Field` | Valeur égale à un autre champ | `validate:"eqfield=Password"` |
| `nefield=Field` | Valeur différente d'un autre champ | `validate:"nefield=OldPassword"` |

## Formats de Fichiers Supportés

| Format | Extension | Compressé |
|--------|-----------|-----------|
| CSV | `.csv` | `.csv.gz`, `.csv.bz2`, `.csv.xz`, `.csv.zst` |
| TSV | `.tsv` | `.tsv.gz`, `.tsv.bz2`, `.tsv.xz`, `.tsv.zst` |
| LTSV | `.ltsv` | `.ltsv.gz`, `.ltsv.bz2`, `.ltsv.xz`, `.ltsv.zst` |
| Excel | `.xlsx` | `.xlsx.gz`, `.xlsx.bz2`, `.xlsx.xz`, `.xlsx.zst` |
| Parquet | `.parquet` | `.parquet.gz`, `.parquet.bz2`, `.parquet.xz`, `.parquet.zst` |

**Note sur les fichiers Excel** : Seule la **première feuille** est traitée. Les feuilles suivantes dans les classeurs multi-feuilles seront ignorées.

## Utilisation Mémoire

fileprep charge le **fichier entier en mémoire** pour le traitement. Cela permet l'accès aléatoire et les opérations multi-passes, mais a des implications pour les gros fichiers :

| Taille du Fichier | Mémoire Approx. | Recommandation |
|-------------------|-----------------|----------------|
| < 100 Mo | ~2-3x taille du fichier | Traitement direct |
| 100-500 Mo | ~500 Mo - 1.5 Go | Surveiller la mémoire, considérer le fractionnement |
| > 500 Mo | > 1.5 Go | Diviser les fichiers ou utiliser des alternatives en streaming |

## Projets Liés

- [nao1215/filesql](https://github.com/nao1215/filesql) - Driver SQL pour CSV, TSV, LTSV, Parquet, Excel
- [nao1215/csv](https://github.com/nao1215/csv) - Lecture CSV avec validation et DataFrame simple

## Contributions

Les contributions sont les bienvenues ! Veuillez consulter le [Guide de Contribution](../../CONTRIBUTING.md) pour plus de détails.

## Support

Si vous trouvez ce projet utile, veuillez considérer :

- Lui donner une étoile sur GitHub - cela aide les autres à découvrir le projet
- [Devenir sponsor](https://github.com/sponsors/nao1215) - votre soutien maintient le projet en vie

## Licence

Ce projet est sous licence MIT - voir le fichier [LICENSE](../../LICENSE) pour plus de détails.
