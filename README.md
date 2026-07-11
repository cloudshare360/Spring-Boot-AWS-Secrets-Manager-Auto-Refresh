# Spring Boot AWS Secrets Manager Auto Refresh

Guides in this repository explain how to keep Spring Boot services running when AWS Secrets Manager rotates PostgreSQL RDS and OpenSearch credentials.

## Documents

- [Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation](Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation.md)
- [Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation - v1](Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation-v1.md)
- [Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation - v2](Spring Boot Auto Secret Refresh for AWS Secrets Manager Rotation-v2.md)
- [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style](Spring Boot AWS Secrets Manager Auto Refresh — Legacy Style.md)
- [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties](Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties.md)
- [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties - v1](Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-v1.md)
- [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties - v2](Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-v2.md)
- [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties - Consolidated](Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-consolidated.md)

## What Changed

- Removed duplicate extensionless document copies.
- Standardized documentation into Markdown files (`.md`).
- Kept both modern and legacy coding style variants.

## Legacy Ignore-Properties Version Map

| File | Type | Purpose |
| --- | --- | --- |
| [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties.md](Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties.md) | Base | Initial legacy guide with ignore-unknown handling. |
| [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-v1.md](Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-v1.md) | v1 | Practical implementation hardening and corrected snippets. |
| [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-v2.md](Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-v2.md) | v2 | Production controls, operational checklist, and validation strategy. |
| [Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-consolidated.md](Spring Boot AWS Secrets Manager Auto Refresh - Legacy Style - Ignore Extra Secret Properties-consolidated.md) | Consolidated | Merged v1 + v2 into one single recommended version. |

## Legacy Ignore-Properties Comparison (Base vs v1 vs v2)

| Capability | Base | v1 | v2 |
| --- | --- | --- | --- |
| Ignore extra secret fields | Yes | Yes | Yes |
| Snippet quality and consistency | Basic | Improved | Improved |
| AWSCURRENT usage | Partial | Yes | Yes |
| Required-field validation before apply | No | Yes | Yes |
| OpenSearch old-client close on refresh | No | Yes | Yes |
| Last-known-good runtime guidance | No | Partial | Yes |
| Concurrency and refresh-storm guidance | Basic | Basic | Advanced |
| Operational checklist and test matrix | No | Minimal | Full |