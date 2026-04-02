# Puppet Metadata Inference Playbook

Detailed file-by-file patterns for inferring every `metadata.json` field from
module source files. Use this reference when the quick-reference tables are not
specific enough or when you encounter edge cases.

---

## 1. Git Remote → `name`, `author`, `source`, `project_page`, `issues_url`

### Parse patterns

```
# HTTPS remotes
https://github.com/{author}/{repo}.git
https://github.com/{author}/{repo}
https://gitlab.com/{author}/{repo}.git
https://bitbucket.org/{author}/{repo}.git

# SSH remotes
git@github.com:{author}/{repo}.git
git@gitlab.com:{author}/{repo}.git
git@bitbucket.org:{author}/{repo}.git
```

### Derivation

```
author       = {author} portion, lowercased
repo_name    = {repo} portion, strip trailing .git
module_name  = repo_name with any leading "puppet-" stripped
               "puppet-apache" → "apache"
               "puppet-ntp"    → "ntp"
               "myorg-stdlib"  → "stdlib"   ← note: no puppet- prefix here
name         = "{author}-{module_name}"
source       = "https://{host}/{author}/{repo_name}"  ← always HTTPS, no .git
project_page = source
issues_url   = source + "/issues"
```

### Edge cases

**No remote at all**: Read `.git/config` directly. If `[remote "origin"]`
section exists but has no `url =` line, treat as unavailable. Fall back to
directory name for `module_name` and `git config user.name` for `author`.

**Multiple remotes**: Always prefer `origin`. If no `origin`, use the first
remote alphabetically.

**Org URL vs personal URL**: Both are handled identically — the first path
segment is always the author.

**Numeric author names**: Valid. `org123-module` is acceptable.

---

## 2. `CHANGELOG.md` → `version`

### Version header patterns (highest to lowest specificity)

```
## [1.2.3] - 2024-01-15        ← keepachangelog format (most common)
## [1.2.3]
### 1.2.3
### Release 1.2.3
= 1.2.3 =                      ← MediaWiki style
**1.2.3** (YYYY-MM-DD)
v1.2.3
```

### Algorithm

1. Scan from top of file.
2. Collect all strings matching `\d+\.\d+\.\d+` found in heading-level lines.
3. Take the highest SemVer value (not the first occurrence — the highest by
   SemVer comparison). Unreleased sections (`## [Unreleased]`) are excluded.
4. If no match is found in headings, return absent.

### Edge cases

**Pre-release versions** (`1.0.0-rc1`): Strip the pre-release suffix before
using. Emit the clean SemVer in metadata.json.

**Date-only headers** (`## 2024-01-15`): Ignore — no version string present.

**CHANGELOG.rst or HISTORY.md**: Apply the same patterns if CHANGELOG.md is
absent and these files exist.

---

## 3. `LICENSE` / `LICENSE.md` → `license`

### Match phrases → SPDX identifiers

```
"Apache License, Version 2.0"         → Apache-2.0
"Apache License 2.0"                  → Apache-2.0
"MIT License"                         → MIT
"Permission is hereby granted, free of charge"  → MIT (body match)
"GNU General Public License v3"       → GPL-3.0-only
"GNU General Public License, version 3"  → GPL-3.0-only
"GNU General Public License v2"       → GPL-2.0-only
"GNU General Public License, version 2"  → GPL-2.0-only
"version 2 or (at your option) any"   → GPL-2.0-or-later
"BSD 2-Clause"                        → BSD-2-Clause
"Simplified BSD License"              → BSD-2-Clause
"BSD 3-Clause"                        → BSD-3-Clause
"New BSD License"                     → BSD-3-Clause
"Modified BSD License"                → BSD-3-Clause
"Mozilla Public License, v. 2.0"     → MPL-2.0
"Mozilla Public License 2.0"         → MPL-2.0
"GNU Lesser General Public License"  → LGPL-2.1-only (check version)
"ISC License"                         → ISC
"The Unlicense"                       → Unlicense
```

Read the first 20 lines of the LICENSE file. One of the above phrases will
almost always appear in the first few lines of a standard license header.

---

## 4. `README.md` → `summary`, `tags`

### Summary extraction

```
Algorithm:
1. Skip front matter (--- ... --- YAML blocks at top)
2. Skip the first # heading (module title, not a description)
3. Skip lines containing [![  (badge lines)
4. Skip ## Table of Contents headings and the TOC list below them
5. Take the first non-empty paragraph (separated by blank lines)
6. Strip Markdown inline formatting: **bold**, _italic_, `code`, [links](url)
7. Truncate to 144 characters, breaking at a word boundary
```

### Tags extraction

```
# Collect from README headings:
  Look for ## or ### headings.
  Skip generic headings: "Table of Contents", "Contents", "Overview",
  "Introduction", "Usage", "Installation", "Requirements", "Examples",
  "Contributing", "License", "Changelog", "Release Notes", "Support",
  "Reference", "Limitations", "Development".
  Extract the remaining heading text, lowercased.
  Split on spaces and hyphens. Keep nouns only (drop articles, prepositions).

# Collect from manifest class names:
  manifests/init.pp    → the module name itself
  manifests/config.pp  → skip (generic)
  manifests/install.pp → skip (generic)
  manifests/service.pp → skip (generic)
  manifests/apache.pp  → "apache"
  manifests/php/fpm.pp → "php", "fpm"
  
# Deduplicate, module name first, max 10 tags total
# Normalize: lowercase, replace underscores with hyphens, strip special chars
```

---

## 5. `.fixtures.yml` → `dependencies`

### Full file structure

```yaml
fixtures:
  repositories:           # ← git repos, NOT Forge modules — skip these
    role:
      repo: 'https://github.com/myorg/puppet-role'
  forge_modules:          # ← THIS is where Forge dependencies live
    stdlib:               # short name (no slash) → resolve via namespace table
      repo: "puppetlabs/stdlib"
      ref: "8.5.0"
    concat:
      repo: "puppetlabs/concat"
    mymodule:             # short name without explicit repo → use namespace table
  symlinks:               # ← symlinks section — ignore for metadata
    mymodule:
      path: "../.."
```

### Parsing rules

```
For each entry under fixtures.forge_modules:
  
  Case 1 — entry has repo: key
    If repo value contains "/" → it IS a forge module
      author = part before "/"
      module = part after "/"
      full_name = "{author}-{module}"
      
      If ref: key exists:
        version = ref value (strip "v" prefix if present)
        range = ">= {version} < {next_major}.0.0"
      Else:
        range = lookup from known ranges table, or flag REVIEW_NEEDED
    
    If repo value is a full URL (https://):
      → This is a git repo, not a Forge module. Skip.
  
  Case 2 — entry has no repo: key (bare shorthand)
    key_name = the YAML key (e.g., "stdlib")
    If key_name contains "/": treat as "author/module" directly
    Else: look up in known namespaces table
      If found: use the mapped full name
      If not found: emit "UNKNOWN_AUTHOR-{key_name}", flag REVIEW_NEEDED
```

### `ref:` → version range conversion

```
ref: "8.5.0"   → ">= 8.5.0 < 9.0.0"   (next major boundary)
ref: "v8.5.0"  → ">= 8.5.0 < 9.0.0"   (strip v prefix first)
ref: "8.5"     → ">= 8.5.0 < 9.0.0"   (add .0 to normalize)
```

---

## 6. `Puppetfile` → `dependencies`

### Line patterns

```ruby
# Simple Forge module (no version specified)
mod 'puppetlabs/stdlib'
mod "puppetlabs/stdlib"

# With version constraint
mod 'puppetlabs/stdlib', '8.5.0'
mod 'puppetlabs/stdlib', '>= 8.0.0'
mod 'puppetlabs/stdlib', :latest

# With tag (treat tag as minimum version)
mod 'puppetlabs/stdlib',
  :tag => '8.5.0'

# Git source — NOT a Forge module, skip
mod 'myorg/internal_module',
  :git => 'https://internal.git.myorg.com/puppet/mymodule'

# Commented line — skip
# mod 'puppetlabs/stdlib'
```

### Parsing rules

```
For each mod line:
  
  Extract author/module from first quoted argument.
  Convert slash to hyphen: "puppetlabs/stdlib" → "puppetlabs-stdlib"
  
  If :git key present → private/internal module, skip and flag.
  
  If version string present:
    If ">= X.Y.Z" → use as-is (add upper bound if missing)
    If "~> X.Y"   → convert per ~> table
    If "X.Y.Z"    → treat as exact lower bound: ">= X.Y.Z < {next_major}.0.0"
    If :latest    → no version info, use known ranges table or flag
  
  If no version → use known ranges table or flag REVIEW_NEEDED
```

---

## 7. `manifests/**/*.pp` — Two Passes

### Pass A: Cross-module reference detection → `dependencies`

Patterns to match (where `X` is a module name different from the current module):

```puppet
include X
include X::subclass
include X::sub::sub

contain X
contain X::subclass

require X
require X::subclass

class { 'X': }
class { 'X::subclass': }
class { "X": }

# Resource-style references to other modules' defined types
X::resource_type { 'title': }

# Ensure we don't match the current module's own classes
```

**Extraction algorithm:**
1. Get `current_module` = module name from Step 1.
2. For each pattern match, extract the top-level module name
   (everything before the first `::`, or the whole name if no `::`).
3. Filter out: `current_module`, Puppet builtins (`file`, `service`, `package`,
   `exec`, `user`, `group`, `cron`, `notify`, `augeas`, `host`, `mount`,
   `scheduled_task`, `selmodule`, `ssh_authorized_key`, `tidy`, `resources`).
4. Deduplicate. Resolve each name using the namespace lookup sequence:
   `.fixtures.yml` → `Puppetfile` → known namespaces table → flag REVIEW_NEEDED.

### Pass B: OS conditional detection → `operatingsystem_support`

Patterns to match:

```puppet
# Direct OS name comparison
$facts['os']['name'] == 'Ubuntu'
$facts['os']['name'] =~ /Ubuntu/
"${facts['os']['name']}" == 'Ubuntu'

# OS family comparison
$facts['os']['family'] == 'RedHat'
$osfamily == 'RedHat'
$facts['osfamily'] == 'RedHat'

# Full OS variable (older style)
$operatingsystem == 'CentOS'
$::operatingsystem == 'CentOS'

# Case statement (collect all case values)
case $facts['os']['name'] {
  'RedHat', 'CentOS', 'Rocky': { ... }
  'Ubuntu', 'Debian': { ... }
}
case $osfamily {
  'RedHat': { ... }
  'Debian': { ... }
}

# Release version alongside OS name
$facts['os']['release']['major'] == '8'
$facts['os']['name'] == 'Ubuntu' and $facts['os']['release']['full'] == '22.04'
```

**Extraction algorithm:**
1. Collect all quoted strings after `==`, `=~`, or in case statement value lists
   that are on the same line or immediately preceded by an OS-related variable.
2. Classify each string as:
   - Known OS name (from canonical list in spec) → OS name signal
   - Known family name (RedHat, Debian, Suse, etc.) → family signal
   - Version-like string (`'7'`, `'8'`, `'20.04'`) → release signal for the
     most recently seen OS/family on the same case branch
3. If a release string is found alongside a specific OS name on the same case
   branch, associate it with that OS.
4. If a release string is found alongside a family name, do not associate it
   with individual OS names — use default release lists instead.

---

## 8. `Gemfile` → `requirements`

### Patterns

```ruby
# Explicit version
gem 'puppet', '>= 7.0.0'
gem 'puppet', '~> 7.0'
gem "puppet", ">= 7.0.0", "< 9.0.0"

# Environment variable with default
gem 'puppet', ENV['PUPPET_GEM_VERSION'] || '>= 7.0.0'
gem 'puppet', ENV.fetch('PUPPET_GEM_VERSION', '7.0.0')

# PDK-managed Gemfile (defers to pdk gem bundle)
# If Gemfile contains only "source 'https://rubygems.org'"
# and references pdk-templates, skip and look at .github/workflows instead
```

### Extraction

```
Match lines containing gem 'puppet' or gem "puppet".
Extract version arguments:
  '>= X.Y.Z'  → lower bound X.Y.Z
  '~> X.Y'    → lower bound X.Y.0
  '< A.0.0'   → upper bound A.0.0 (if present as separate argument)
  'X.Y.Z'     → exact version (treat as lower bound, add next-major upper)
  Default value from ENV.fetch → lower bound

If no upper bound found in Gemfile, use next major above lower bound.
```

---

## 9. `.github/workflows/*.yml` → `requirements`

### Patterns

```yaml
# Matrix with puppet_version
strategy:
  matrix:
    puppet_version:
      - '~> 7.0'
      - '~> 8.0'

# Environment variable in matrix
env:
  PUPPET_GEM_VERSION: '~> 7.0'

# PDK acceptance tests matrix
matrix:
  include:
    - puppet_version: 'latest'
      ruby_version: '3.2'
```

### Extraction

```
Collect all values under puppet_version: keys and PUPPET_GEM_VERSION: env vars.
Strip ~> prefix: '~> 7.0' → '7.0.0'
Ignore 'latest' (not a version number — fall back to other sources).
Lower bound = min of all collected versions.
Upper bound = next major above max collected version.
```

---

## 10. Edge Case Decision Trees

### No .fixtures.yml AND no Puppetfile, but manifests have cross-module refs

```
For each cross-module reference found in manifests:
  
  1. Check known namespaces table in quick-reference.md
     FOUND     → use mapped name + known range (or REVIEW_NEEDED for range)
     NOT FOUND → emit "UNKNOWN_AUTHOR-{name}", flag REVIEW_NEEDED
  
  2. Do NOT fabricate author names. If unsure, always flag.
```

### OS conditionals present but no releases found anywhere

```
For each OS name collected:
  Use default release list from quick-reference.md.
  Add a note to Review Summary:
  "operatingsystem_support releases for {OS} defaulted to {releases} —
   verify these match your module's actual tested platforms."
```

### Module has no CI config at all (no Gemfile, no .github/, no .travis.yml)

```
Check metadata.json requirements field first.
  If present and non-empty → use it as-is.
  If absent → emit ">= 7.0.0 < 9.0.0" and flag:
  "requirements defaulted to >= 7.0.0 < 9.0.0 — no Gemfile or CI config found.
   Check your Gemfile or pdk.yaml to confirm supported Puppet versions."
```

### Conflicting OS signals (manifests say RedHat, Vagrantfile says Ubuntu)

```
Merge both sources. Include OS entries from all sources.
Do not discard signals from any source — more is better for OS support.
Only deduplicate within the same OS name.
```

### Module with no manifests/ directory (tasks-only or plans-only module)

```
Skip all manifest scanning steps (5c, 7a, 7b passes).
Set dependencies = [] and flag as REVIEW_NEEDED with:
"No manifests found — dependencies cannot be inferred from Puppet code.
 Check Puppetfile or .fixtures.yml manually."
Set operatingsystem_support = [] and flag similarly.
```

### `init.pp` defines multiple classes (unusual but valid)

```
Use the class whose name matches the module directory name exactly.
This is the "main" class. Use its @summary tag for the summary field.
Other classes in init.pp are secondary and do not affect name/summary inference.
```
