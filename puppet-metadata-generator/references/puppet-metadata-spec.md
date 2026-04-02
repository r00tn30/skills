# Puppet Module Metadata — Full Specification

## Field Reference

### `name` (required)
- **Type**: string
- **Format**: `author-module_name` — lowercase author, hyphen separator,
  lowercase module name with underscores (no hyphens in the module name part)
- **Forge regex**: `^[a-z][a-z0-9_]*-[a-z][a-z0-9_]*$`
- **Length**: max 200 characters combined
- **Examples**: `puppetlabs-stdlib`, `myorg-apache`, `voxpupuli-nginx`
- **Common mistake**: including `puppet-` in the module name part.
  `myorg-puppet_apache` is wrong — the repo name `puppet-apache` maps to
  module name `apache`, so the field is `myorg-apache`.
- **PDK behavior**: Set once during `pdk new module`. PDK does NOT update it
  automatically afterward. The directory name and this field can diverge.

### `version` (required)
- **Type**: string
- **Format**: SemVer `MAJOR.MINOR.PATCH` — no `v` prefix, no build metadata
- **Forge validation**: Must be valid SemVer.
  - `1.0.0` ✓
  - `v1.0.0` ✗ (no `v` prefix)
  - `1.0` ✗ (missing patch)
  - `1.0.0-rc1` ✗ (pre-release not accepted by Forge uploads)
- **PDK behavior**: PDK does not auto-increment. User bumps manually or via
  `pdk release prep`.

### `author` (required)
- **Type**: string
- **Format**: lowercase alphanumeric with underscores and hyphens
- **Forge validation**: Must match the author portion of the `name` field
  exactly. If `name` is `myorg-apache`, `author` must be `myorg`.
- **Case rule**: Must be lowercase. `MyOrg` → `myorg`.

### `summary` (required)
- **Type**: string
- **Length**: 1–144 characters (Puppet Forge hard limit enforced at publish)
- **Content**: One-line human-readable description. No Markdown. No trailing
  period required but acceptable.
- **Forge validation**: Required for module publishing. Missing or blank
  causes `pdk validate` to error.
- **PDK behavior**: Set once during `pdk new module` via interactive prompt.
  Not touched by `pdk update`.

### `license` (required)
- **Type**: string (SPDX identifier)
- **Common values and their LICENSE file trigger phrases**:

  | SPDX Identifier | LICENSE file contains |
  |---|---|
  | `Apache-2.0` | "Apache License, Version 2.0" |
  | `MIT` | "MIT License" or "Permission is hereby granted, free of charge" |
  | `GPL-3.0-only` | "GNU General Public License v3" or "version 3" |
  | `GPL-2.0-only` | "GNU General Public License v2" or "version 2" (no "or later") |
  | `GPL-2.0-or-later` | "GPL version 2 or later" / "v2+" |
  | `BSD-2-Clause` | "BSD 2-Clause" or "Simplified BSD" |
  | `BSD-3-Clause` | "BSD 3-Clause" or "New BSD" or "Modified BSD" |
  | `MPL-2.0` | "Mozilla Public License, v. 2.0" |
  | `LGPL-2.1-only` | "GNU Lesser General Public License v2.1" |

- **Forge validation**: Accepts any SPDX identifier. Unknown values produce a
  warning but do not block publishing.
- **Default**: `Apache-2.0` — used by all `puppetlabs/` and `voxpupuli/`
  modules. Safe to default to when LICENSE file is absent.

### `source` (required)
- **Type**: string (URL)
- **Format**: HTTPS URL, no trailing slash, no `.git` suffix
- **Example**: `https://github.com/puppetlabs/puppetlabs-stdlib`
- **Forge validation**: Must be a parseable URL. No format enforcement beyond
  that. SSH URLs will not render as clickable links on the Forge.
- **Conversion**: `git@github.com:org/repo.git` → `https://github.com/org/repo`

### `project_page` (optional but strongly recommended)
- **Type**: string (URL)
- **Default**: Same as `source`
- **Forge display**: Shown as "Project" link on the module page

### `issues_url` (optional but strongly recommended)
- **Type**: string (URL)
- **Default**: `source + "/issues"` for GitHub/GitLab repos
- **Forge display**: Shown as "Issues" link on the module page

### `dependencies` (required when the module has external dependencies)
- **Type**: array of dependency objects
- **Schema per entry**:
  ```json
  {
    "name": "author-module_name",
    "version_requirement": ">= X.Y.Z < A.0.0"
  }
  ```
- **Forge validation**:
  - `name` must match `^[a-z][a-z0-9_]*-[a-z][a-z0-9_]*$`
  - `version_requirement` must be valid SemVer range syntax
  - Duplicate entries cause `pdk validate` failure
  - Circular dependencies (A depends on B, B depends on A) are rejected
- **Empty array**: Valid for modules with no external dependencies. Omitting
  the field entirely is also valid but explicit `[]` is preferred.
- **PDK behavior**: PDK never populates or updates `dependencies`.

### `requirements` (required in practice)
- **Type**: array of requirement objects
- **Schema per entry**:
  ```json
  {
    "name": "puppet",
    "version_requirement": ">= 7.0.0 < 9.0.0"
  }
  ```
- **Supported `name` values**:
  - `"puppet"` — the Puppet agent. Used by virtually every module.
  - `"pe"` — Puppet Enterprise only. Rarely used; omit unless the module is
    explicitly PE-only.
  - `"facter"` — Facter version. Rarely needed; modern modules omit this.
- **Standard entry**: Almost every module has exactly one entry with
  `"name": "puppet"`.
- **Typical ranges in use (2025/2026)**:
  - `">= 7.0.0 < 9.0.0"` — modern modules supporting Puppet 7 and 8
  - `">= 6.0.0 < 9.0.0"` — older maintained modules still supporting Puppet 6
  - `">= 8.0.0 < 9.0.0"` — new modules targeting only Puppet 8
- **PDK behavior**: PDK does not auto-populate `requirements`.

### `operatingsystem_support` (required for Forge publishing)
- **Type**: array of OS support objects
- **Schema per entry**:
  ```json
  {
    "operatingsystem": "RedHat",
    "operatingsystemrelease": ["7", "8", "9"]
  }
  ```
- **`operatingsystem` accepted values (Puppet Forge canonical names)**:

  | Canonical Name | Notes |
  |---|---|
  | `RedHat` | Red Hat Enterprise Linux |
  | `CentOS` | CentOS Linux |
  | `Rocky` | Rocky Linux |
  | `AlmaLinux` | AlmaLinux OS |
  | `OracleLinux` | Oracle Linux |
  | `Scientific` | Scientific Linux |
  | `Fedora` | Fedora |
  | `Debian` | Debian GNU/Linux |
  | `Ubuntu` | Ubuntu |
  | `SLES` | SUSE Linux Enterprise Server |
  | `OpenSuSE` | openSUSE |
  | `Archlinux` | Arch Linux |
  | `Gentoo` | Gentoo Linux |
  | `FreeBSD` | FreeBSD |
  | `OpenBSD` | OpenBSD |
  | `Darwin` | macOS |
  | `windows` | Microsoft Windows (lowercase!) |
  | `Solaris` | Solaris |
  | `AIX` | IBM AIX |
  | `Amazon` | Amazon Linux |

  **Important**: `windows` is lowercase. All others are title-case or
  CamelCase exactly as shown. Using wrong case will cause Forge validation
  warnings.

- **`operatingsystemrelease` values**:
  - Linux distributions: major version strings — `"7"`, `"8"`, `"9"`
  - Ubuntu: point release strings — `"20.04"`, `"22.04"`, `"24.04"`
  - Debian: major strings — `"10"`, `"11"`, `"12"`
  - Windows: year strings — `"2016"`, `"2019"`, `"2022"`
  - Always strings, never integers

---

## version_requirement Format Reference

### Accepted Syntax

| Pattern | Meaning | Use for |
|---|---|---|
| `>= 8.0.0 < 9.0.0` | Any 8.x.x | Standard range |
| `>= 8.5.0 < 9.0.0` | 8.5.0 and above within 8.x | When minimum minor matters |
| `>= 8.5.1 < 9.0.0` | 8.5.1 and above within 8.x | When minimum patch matters |
| `>= 7.0.0 < 9.0.0` | Any 7.x.x or 8.x.x | Spanning two majors |

### Forbidden Patterns

| Pattern | Problem | Correct Alternative |
|---|---|---|
| `~> 8.0` | Rubygems syntax, not SemVer | `>= 8.0.0 < 9.0.0` |
| `>= 8.0.0` | No upper bound — will match future breaking versions | `>= 8.0.0 < 9.0.0` |
| `8.0.0` | Exact pin — too strict | `>= 8.0.0 < 9.0.0` |
| `4.*` | Wildcard — not valid SemVer range | `>= 4.0.0 < 5.0.0` |
| `>= 8.0.0 <= 8.5.0` | Closed range — too restrictive | `>= 8.0.0 < 9.0.0` |

### ~> Conversion Table

| Puppetfile / fixtures `~>` value | Metadata `version_requirement` |
|---|---|
| `~> 4` | `>= 4.0.0 < 5.0.0` |
| `~> 8.0` | `>= 8.0.0 < 9.0.0` |
| `~> 8.5` | `>= 8.5.0 < 9.0.0` |
| `~> 8.5.0` | `>= 8.5.0 < 8.6.0` |
| `~> 8.5.1` | `>= 8.5.1 < 8.6.0` |

---

## Known Dependency Version Ranges

These are current as of early 2026. Always verify against the actual Forge
before publishing.

| Forge Module | Recommended Range | Notes |
|---|---|---|
| `puppetlabs-stdlib` | `>= 8.0.0 < 10.0.0` | Required by nearly every module |
| `puppetlabs-concat` | `>= 7.0.0 < 10.0.0` | File concatenation |
| `puppetlabs-apt` | `>= 8.0.0 < 10.0.0` | Debian/Ubuntu APT management |
| `puppetlabs-apache` | `>= 8.0.0 < 13.0.0` | Apache HTTP Server |
| `puppetlabs-mysql` | `>= 12.0.0 < 16.0.0` | MySQL/MariaDB |
| `puppetlabs-postgresql` | `>= 8.0.0 < 11.0.0` | PostgreSQL |
| `puppetlabs-java` | `>= 7.0.0 < 11.0.0` | Java installation |
| `puppetlabs-tomcat` | `>= 5.0.0 < 8.0.0` | Tomcat |
| `puppetlabs-firewall` | `>= 3.0.0 < 7.0.0` | Firewall rules |
| `puppetlabs-inifile` | `>= 5.0.0 < 7.0.0` | INI file management |
| `puppetlabs-vcsrepo` | `>= 5.0.0 < 7.0.0` | VCS repo management |
| `puppetlabs-ntp` | `>= 4.0.0 < 6.0.0` | NTP management |
| `puppetlabs-puppet_agent` | `>= 4.0.0 < 8.0.0` | Puppet agent management |
| `puppet-archive` | `>= 6.0.0 < 8.0.0` | File downloads |
| `stahnma-epel` | `>= 1.3.0 < 4.0.0` | EPEL repository |
| `choria-module_data` | `>= 0.0.1 < 1.0.0` | Module-level Hiera data |

---

## Puppet Forge Validation Rules (what `pdk validate` checks)

### Errors (block publishing)
- `name` missing or wrong format
- `version` missing or not valid SemVer
- `author` missing or does not match `name` prefix
- `summary` missing or empty
- `license` missing
- Duplicate `dependencies` entries (same module name twice)
- `dependencies[].name` in wrong format (slash instead of hyphen)
- `version_requirement` using invalid syntax

### Warnings (do not block but should be fixed)
- `source` missing
- `operatingsystem_support` missing or empty
- OS name not in canonical list (e.g., `Centos` instead of `CentOS`)
- `version_requirement` with no upper bound
- `summary` longer than 144 characters

### Common Failures and Fixes

| Failure | Cause | Fix |
|---|---|---|
| "name is invalid" | Slash instead of hyphen: `puppetlabs/stdlib` | Change to `puppetlabs-stdlib` |
| "author doesn't match name" | `author: MyOrg` vs `name: myorg-module` | Lowercase author to match name prefix |
| "version is not valid SemVer" | `"1.0"` or `"v1.0.0"` | Use `"1.0.0"` format, no `v` |
| "summary too long" | Over 144 characters | Truncate to ≤ 144 characters |
| "duplicate dependency" | Same module listed twice | Remove duplicate entry |
| "invalid OS name" | Wrong case or spelling | Use canonical name from the table above |
| "version_requirement invalid" | Using `~>` or `*` | Convert to `>= X.Y.Z < A.0.0` format |
