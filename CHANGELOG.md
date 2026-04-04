# Changelog

## [0.2.0] — 2026-04-04

### Fixed

- **`upload_file` password was always "Incorrect"** — gofile stores and compares the `attributeValue` string directly; the previous SHA-256 hashing of the password before sending was wrong. Reverted to storing the raw random password.
- **`upload_file` path check failed on Windows** — inbox boundary was checked with a hardcoded `'/'` separator. Replaced with `path.sep` so it works on both Unix and Windows.
- **`upload_file` API errors were silently swallowed** — added `pwRes.ok` and `expRes.ok` checks so gofile API failures surface as errors instead of producing a broken download link.

### Added

- **`fullAccess` config option in `access.json`** — controls whether `upload_file` is restricted to the inbox directory (`false`, default) or may access any file on the host (`true`). The MCP tool description and security instructions update dynamically based on this setting.
- **`history.log` startup context** — `examples/CLAUDE.md` now instructs Claude to read the last 15 000 lines of `history.log` on startup (was 200), restoring full conversation context after a restart.

### Updating from 0.1.0

Run the install command again to get the latest version:

```
claude plugin install line@claude-line-channel
```
