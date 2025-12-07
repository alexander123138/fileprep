# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2025-12-07

### Added
- **Initial Release**: First stable release of fileprep library
- **File Format Support**: CSV, TSV, LTSV, Parquet, Excel (.xlsx) with compression support (gzip, bzip2, xz, zstd)
- **Preprocessing Tags (`prep`)**: Comprehensive struct tag-based preprocessing
  - Basic preprocessors: `trim`, `ltrim`, `rtrim`, `lowercase`, `uppercase`, `default`
  - String transformation: `replace`, `prefix`, `suffix`, `truncate`, `strip_html`, `strip_newline`, `collapse_space`
  - Character filtering: `remove_digits`, `remove_alpha`, `keep_digits`, `keep_alpha`, `trim_set`
  - Padding: `pad_left`, `pad_right`
  - Advanced: `normalize_unicode`, `nullify`, `coerce`, `fix_scheme`, `regex_replace`
- **Validation Tags (`validate`)**: Compatible with go-playground/validator syntax
  - Basic validators: `required`, `boolean`
  - Character type validators: `alpha`, `alphaunicode`, `alphaspace`, `alphanumeric`, `alphanumunicode`, `numeric`, `number`, `ascii`, `printascii`, `multibyte`
  - Numeric comparison: `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `min`, `max`, `len`
  - String validators: `oneof`, `lowercase`, `uppercase`, `eq_ignore_case`, `ne_ignore_case`
  - String content: `startswith`, `startsnotwith`, `endswith`, `endsnotwith`, `contains`, `containsany`, `containsrune`, `excludes`, `excludesall`, `excludesrune`
  - Format validators: `email`, `uri`, `url`, `http_url`, `https_url`, `url_encoded`, `datauri`, `uuid`
  - Network validators: `ip_addr`, `ip4_addr`, `ip6_addr`, `cidr`, `cidrv4`, `cidrv6`, `fqdn`, `hostname`, `hostname_rfc1123`, `hostname_port`
  - Cross-field validators: `eqfield`, `nefield`, `gtfield`, `gtefield`, `ltfield`, `ltefield`, `fieldcontains`, `fieldexcludes`
- **Name-Based Column Binding**: Automatic snake_case conversion with `name` tag override
- **filesql Integration**: Returns `io.Reader` for direct use with filesql
- **Detailed Error Reporting**: Row and column information for each validation error

### Technical Details
- **Memory Optimization**: In-place record processing, pre-allocated output buffers, streaming parsers for CSV/TSV/LTSV
- **XLSX Streaming**: Uses excelize streaming API to reduce memory usage for large files
- **Parquet Buffer Reuse**: Reusable row buffer across row groups to reduce allocations
- **Format-Specific Limitations**:
  - XLSX: Only the first sheet is processed
  - LTSV: Maximum line size is 10MB
