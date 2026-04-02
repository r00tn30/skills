---
name: puppet-metadata-generator
description: >
  Puppet module metadata.json auto-populator. Invoke whenever a user mentions
  metadata.json in a Puppet module context, asks to update, generate, populate,
  or fix module metadata, or runs pdk new module, pdk update, or pdk validate.
  Inspect the module codebase first — never ask for values that can be inferred
  from files. Produce a complete, valid metadata.json block. Triggers
  proactively: if you are in a Puppet module directory (contains manifests/init.pp
  or a metadata.json with an author-module name field), apply this skill before
  writing or editing metadata.json.
---

# Puppets — Puppet Module Metadata Auto-Populator

**CRITICAL**: Before writing a single field in `metadata.json`, run the full
inspection sequence below. Never guess. Never ask for values that exist in the
codebase. Infer first, flag ambiguity second, ask the user only as a last resort.
If a field cannot be inferred with confidence, emit it with a `"REVIEW_NEEDED"`
sentinel and add it to the Review Summary at the end.

Full reference material in:
- `references/puppet-metadata-spec.md` — field types, Forge validation rules,
  accepted OS names, known dependency version ranges
- `references/inference-playbook.md` — file-by-file regex patterns, edge cases,
  ambiguous signal handling
- `assets/quick-reference.md` — rapid-lookup tables (file→field, OS families,
  version_requirement format, known namespaces)

---

## Trigger Conditions

Invoke this skill whenever ANY of the following is true:

- The user mentions `metadata.json` in a Puppet module directory
- The user asks to "update", "generate", "populate", or "fix" module metadata
- The user runs or asks about `pdk new module`, `pdk update`, or `pdk validate`
- The user asks about module dependencies, OS support, or Puppet version ranges
- You are about to write or edit `metadata.json` in a directory that contains
  `manifests/init.pp` or a `metadata.json` that already has a `"name"` field
  in the `author-module` format

---

## Inspection Sequence — Run All Steps Before Writing Output

Execute every step below. Do not short-circuit. Gather all signals first,
then produce the output block once.

### Step 1 — Establish module identity

1. Read `metadata.json` if it exists. Note every field that already has a
   non-empty, non-placeholder value — these are **existing values** that must
   not be silently overwritten.
2. Read `.git/config` or run `git remote get-url origin` to get the remote URL.
   Parse `author` and `repo_name` from the pattern:
   - `https://github.com/{author}/{repo_name}`
   - `git@github.com:{author}/{repo_name}.git`
   - Same patterns for `gitlab.com`, `bitbucket.org`
3. Derive `module_name` from `repo_name`: strip any `puppet-` prefix.
   `puppet-apache` → `apache`. `puppet-ntp` → `ntp`. `myorg-stdlib` → `stdlib`.
4. Derive `name` field: `{author}-{module_name}` (hyphen separator, all lowercase).
5. If remote URL is unavailable, fall back to `git config user.name` for
   author and the current directory name (minus `puppet-` prefix) for module
   name. Flag both as `REVIEW_NEEDED`.

### Step 2 — Read version and license

**Version:**
1. Check existing `metadata.json` → `version` field. If non-empty and not
   `"0.0.1"` or `"0.1.0"`, use it as-is.
2. If absent or a default placeholder, scan `CHANGELOG.md` for the most
   recent version header. Match patterns: `## [1.2.3]`, `### 1.2.3`,
   `= v1.2.3 =`, `**1.2.3**`. Take the highest SemVer found.
3. If no CHANGELOG, emit `"1.0.0"` and flag as `REVIEW_NEEDED`.

**License:**
1. Read `LICENSE` or `LICENSE.md`. Map content to SPDX identifier:
   - Contains "Apache License, Version 2.0" → `"Apache-2.0"`
   - Contains "MIT License" → `"MIT"`
   - Contains "GNU General Public License v3" → `"GPL-3.0-only"`
   - Contains "GNU General Public License v2" → `"GPL-2.0-only"`
   - Contains "BSD 2-Clause" → `"BSD-2-Clause"`
   - Contains "BSD 3-Clause" → `"BSD-3-Clause"`
2. If file is absent, emit `"Apache-2.0"` (Forge standard default) and flag
   as `REVIEW_NEEDED`.

### Step 3 — Read summary

1. Read `README.md`. Take the first non-empty paragraph that is NOT:
   - A badge line (contains `[![`)
   - A heading (starts with `#`)
   - A table-of-contents link (starts with `* [` or `- [`)
   - A blank line or horizontal rule
   Truncate to 144 characters if longer (Puppet Forge hard limit).
2. If README is absent or the first paragraph is uninformative, open
   `manifests/init.pp` and extract the first `@summary` tag from the
   class doc comment block:
   ```puppet
   # @summary This module manages Apache HTTP Server
   ```
3. If neither source yields a summary, emit `"REVIEW_NEEDED"`.

### Step 4 — Derive source URLs

1. `source` = the git remote URL from Step 1, normalized to HTTPS with no
   trailing slash and no `.git` suffix:
   `git@github.com:org/puppet-apache.git` → `https://github.com/org/puppet-apache`
2. `project_page` = same as `source` (default; user may override separately).
3. `issues_url` = `source + "/issues"`.
4. If the remote URL is unavailable, flag all three as `REVIEW_NEEDED`.

### Step 5 — Infer dependencies

Run all four sub-steps and merge results. Earlier sub-steps have higher
priority for version ranges when the same module appears in multiple sources.

**5a. Read `.fixtures.yml`** (highest priority)
- Parse the `fixtures.forge_modules` section.
- Each entry with a `repo:` key pointing to `forge.puppet.com` or a
  `author/module` slug is a dependency.
- If the entry key has no slash (e.g., `stdlib:`), resolve the namespace
  using the known namespaces table in `assets/quick-reference.md`.
- If `ref:` is present, use it as the minimum version:
  `ref: "8.5.0"` → `">= 8.5.0 < 9.0.0"`.
- If `ref:` is absent, use the known range from the table or flag as
  `REVIEW_NEEDED`.

  ```yaml
  # .fixtures.yml example
  fixtures:
    forge_modules:
      stdlib:
        repo: "puppetlabs/stdlib"
        ref: "8.5.0"
  ```
  → `{ "name": "puppetlabs-stdlib", "version_requirement": ">= 8.5.0 < 9.0.0" }`

**5b. Read `Puppetfile`** (second priority)
- Parse lines matching `mod 'author/module'` or `mod "author/module"`.
- If a version argument or `:tag` is present, use it as the lower bound.
- Skip entries with `:git =>` pointing to non-Forge URLs (private/internal
  modules). Flag them in the Review Summary as private dependencies.

**5c. Scan `manifests/**/*.pp`** (third priority — cross-module references)
- For each `.pp` file, extract class/resource references that belong to a
  different module namespace than the current module:
  ```puppet
  include stdlib
  include other_module::something
  contain other_module::something
  require other_module::something
  class { 'other_module::something': }
  ```
- Collect unique top-level module names. Exclude the current module's own
  namespace.
- For each collected name, look it up in `.fixtures.yml` and `Puppetfile`
  to resolve the full `author/module`. If not found, emit
  `"UNKNOWN_AUTHOR-{name}"` and flag as `REVIEW_NEEDED`.

**5d. Cross-reference version ranges**
- For every resolved dependency, look up the known range in
  `references/puppet-metadata-spec.md` → "Known Dependency Version Ranges".
- If no known range exists and no version hint was found in Steps 5a/5b,
  emit `">= 0.0.0 < 1.0.0"` and flag as `REVIEW_NEEDED`.

**Merge rule**: If a module appears in multiple sub-steps, use the version
range from the highest-priority sub-step (5a > 5b > 5c > 5d). Never emit
duplicate entries.

**~> conversion**: If `.fixtures.yml` or `Puppetfile` uses `~>`, convert:
- `~> 8.0` → `">= 8.0.0 < 9.0.0"`
- `~> 8.5` → `">= 8.5.0 < 9.0.0"`
- `~> 8.5.0` → `">= 8.5.0 < 8.6.0"`
Never emit `~>` in `metadata.json`.

### Step 6 — Infer Puppet version requirements

Check sources in priority order. Stop at the first source that yields a version.

1. **`Gemfile`** — match:
   - `gem 'puppet', '>= X.Y.Z'`
   - `ENV.fetch('PUPPET_GEM_VERSION', 'X.Y.Z')`
   Extract version as lower bound.
2. **`.github/workflows/*.yml`** — match:
   - `puppet_version:` matrix entries
   - `PUPPET_GEM_VERSION:` env vars
   Collect all listed versions.
3. **`.travis.yml`** — match `PUPPET_GEM_VERSION=X.Y.Z` in `env:` matrix.
4. **`.sync.yml` or `pdk.yaml`** — match `supported_releases:` list.
5. If none found, emit `">= 7.0.0 < 9.0.0"` and flag as `REVIEW_NEEDED`.

**Compute the range:**
- Lower bound = minimum version found across all sources, rounded to `X.0.0`
  unless a specific minor/patch is required.
- Upper bound = next major above the maximum found version.
- Format: `">= {lower} < {upper_major}.0.0"`

### Step 7 — Infer operatingsystem_support

Run all sub-steps and merge. More specific signals override general ones.

**7a. Scan all `manifests/**/*.pp`** for OS conditionals:
```puppet
$facts['os']['name'] == 'Ubuntu'
$facts['os']['family'] == 'RedHat'
$osfamily == 'Debian'
$operatingsystem == 'CentOS'
case $facts['os']['name'] { 'RedHat', 'CentOS': { ... } }
case $osfamily { 'RedHat': { ... } }
```
Collect every quoted string that is a known OS name or family name.
Extract any release/version strings adjacent to an OS name.

**7b. Check `spec/acceptance/` nodesets**
Read `.nodesets.yml` or any `*_nodeset.yml`. Extract `platform:` values.
Map platform strings: `centos-7-x86_64` → CentOS 7, `ubuntu-2004-x86_64`
→ Ubuntu 20.04, `debian-11-x86_64` → Debian 11.

**7c. Check `Vagrantfile`**
Match `box:` or `config.vm.box =` lines. Map box names:
`centos/7` → CentOS 7, `ubuntu/focal64` → Ubuntu 20.04,
`ubuntu/jammy64` → Ubuntu 22.04, `rockylinux/8` → Rocky 8.

**7d. Expand OS families → OS names**
Use the family mapping from `assets/quick-reference.md`:
- `RedHat` → RedHat, CentOS, Rocky, AlmaLinux
- `Debian` → Debian, Ubuntu
- Direct OS name match → include only that OS, not the whole family

If both a family name AND specific OS names are found for the same family,
use only the specific OS names (they are more precise).

**7e. Populate releases**
- If release strings (`'7'`, `'8'`, `'20.04'`) are found alongside an OS
  name, include only those releases.
- If no release strings are found for an OS, use the default release list
  from `assets/quick-reference.md` → "Default OS Release Lists".
- Sort releases ascending (numerically for major versions, lexically for
  point releases like `20.04`).

### Step 8 — Infer tags

1. Read `README.md` headings (`## Features`, `## Usage`, `## Overview`).
   Extract meaningful nouns. Exclude generic words: module, puppet, class,
   manifest, file, service, package, resource, install, configure.
2. Read class names in `manifests/`: `manifests/apache.pp` → `apache`,
   `manifests/php/fpm.pp` → `php`, `fpm`.
3. Limit to 10 tags maximum. All tags must be lowercase alphanumeric with
   hyphens only — no underscores, no spaces.
4. Deduplicate. The module name itself is always the first tag.

---

## Field-by-Field Inference Rules Summary

| Field | Primary Source | Fallback | Flag if |
|---|---|---|---|
| `name` | git remote URL | directory name | Remote unavailable |
| `version` | `metadata.json` → `CHANGELOG.md` | `"1.0.0"` | No CHANGELOG |
| `author` | git remote URL | `git config user.name` | Remote unavailable |
| `summary` | `README.md` first para | `init.pp` `@summary` | Neither found |
| `license` | `LICENSE` file | `"Apache-2.0"` | File absent |
| `source` | git remote URL (HTTPS) | — | Remote unavailable |
| `project_page` | = `source` | — | Source flagged |
| `issues_url` | `source + "/issues"` | — | Source flagged |
| `dependencies` | `.fixtures.yml` → `Puppetfile` → manifests | `">= 0.0.0 < 1.0.0"` | Unknown author or version range |
| `requirements` | `Gemfile` → `.github/workflows` → `.travis.yml` → `pdk.yaml` | `">= 7.0.0 < 9.0.0"` | No CI config found |
| `operatingsystem_support` | `.pp` conditionals → nodesets → `Vagrantfile` | defaults from spec | No OS signals anywhere |
| `tags` | README headings + class names | module name only | — |

---

## PDK Field Ownership — Never Touch These

PDK manages the following fields and will overwrite any manual values on the
next `pdk update` run. If they exist in the current `metadata.json`, preserve
them exactly as-is. Do not include them in the inferred output section — they
belong to PDK.

```
pdk-version
template-url
template-ref
```

---

## Conflict Resolution — Existing vs Inferred

When `metadata.json` already exists:

1. **Non-empty existing value ≠ inferred value**: Keep the existing value.
   Emit a note in the Review Summary:
   `"KEPT_EXISTING: {field} — inferred '{inferred}' but existing is '{existing}'"`
2. **Placeholder existing value** (`""`, `"FIXME"`, `"0.0.1"`, `"0.1.0"` for
   version, `"username-module_name"` for name): Replace with inferred value.
3. **PDK-managed fields**: Preserve unconditionally. Never modify.
4. **Missing field**: Add with inferred value (or `REVIEW_NEEDED` sentinel).

---

## Output Format

Always produce three sections in this order:

**1. Complete `metadata.json` block** — every required field present, PDK
fields preserved from existing data if present, `REVIEW_NEEDED` sentinels
for fields that could not be inferred:

```json
{
  "name": "myorg-apache",
  "version": "2.3.0",
  "author": "myorg",
  "summary": "Manages Apache HTTP Server installation and configuration",
  "license": "Apache-2.0",
  "source": "https://github.com/myorg/puppet-apache",
  "project_page": "https://github.com/myorg/puppet-apache",
  "issues_url": "https://github.com/myorg/puppet-apache/issues",
  "dependencies": [
    {
      "name": "puppetlabs-stdlib",
      "version_requirement": ">= 8.0.0 < 9.0.0"
    }
  ],
  "requirements": [
    {
      "name": "puppet",
      "version_requirement": ">= 7.0.0 < 9.0.0"
    }
  ],
  "operatingsystem_support": [
    {
      "operatingsystem": "RedHat",
      "operatingsystemrelease": ["7", "8", "9"]
    },
    {
      "operatingsystem": "Ubuntu",
      "operatingsystemrelease": ["20.04", "22.04"]
    }
  ],
  "tags": ["apache", "http", "web"]
}
```

**2. Review Summary** — list every `REVIEW_NEEDED` item with the reason and
a specific suggestion for how the user can resolve it:

```
## Review Summary

- **version**: No CHANGELOG.md found. Defaulted to "1.0.0". Update to match
  your actual release version.
- **dependencies.version_requirement (myorg-internal)**: Module found in
  manifests/ but not on Puppet Forge. Confirm if a public Forge equivalent
  exists or remove from dependencies.
```

**3. Confidence Summary** — one line per field group:

```
## Confidence Summary

Fully inferred:    name, author, source, project_page, issues_url, license,
                   summary, operatingsystem_support
Kept existing:     version (2.3.0 from metadata.json)
Needs review:      dependencies.version_requirement (internal-myapp)
```

---

## REVIEW_NEEDED — When to Flag and When Not To

**Always flag:**
- `version`: No CHANGELOG and metadata has `"0.0.1"` or `"0.1.0"` default
- `author`: Git remote URL unavailable
- `summary`: README absent and no `@summary` tag in `init.pp`
- Any `dependency` found in manifests but not in `.fixtures.yml`/`Puppetfile`
  (unknown author namespace)
- Any `version_requirement` where the module is not in the known ranges table
  and no version hint was found
- `requirements`: No Gemfile, no CI config, no `pdk.yaml` found
- `operatingsystem_support`: No OS conditionals, no nodesets, no Vagrantfile

**Do NOT flag:**
- `tags` — always has at least the module name; never a blocker
- `license` defaulting to `"Apache-2.0"` — it is the Forge standard default
- `project_page` and `issues_url` derived from `source` — they are deterministic
- `~>` conversion — this is a mechanical transform, not an ambiguity

---

## Edge Cases

**No git remote available**: Use current directory name for module name; flag
`author`, `source`, `project_page`, and `issues_url` as `REVIEW_NEEDED`. Emit:
"Run `git remote add origin <URL>` and re-run this skill for accurate author
and source fields."

**Module has multiple remotes**: Use `origin`. If `origin` is absent, use the
first remote alphabetically and flag as `REVIEW_NEEDED`.

**Monorepo (multiple modules in subdirectories)**: Treat each subdirectory
containing `manifests/init.pp` as a separate module. Run the full inspection
sequence for the target subdirectory only. Do not aggregate dependencies
across sibling modules.

**Private/internal dependencies**: Dependencies with `:git =>` pointing to a
private host are not Forge modules. Exclude them from the `dependencies` array.
Flag them in the Review Summary: "Private module `{name}` found in Puppetfile
— verify if a public Forge equivalent exists."

**`init.pp` absent, split class layout**: Use the module directory name for
the module name, not the class name from a specific manifest file. Split class
layouts are normal and do not affect metadata inference.

**`~>` in fixtures or Puppetfile**: Convert to explicit range before emitting.
`~> 8.0` → `">= 8.0.0 < 9.0.0"`. Never emit `~>` in `metadata.json`.

---

## When to Consult References

- **All field types, Forge validation rules, accepted OS names**: `references/puppet-metadata-spec.md`
- **File-by-file regex patterns, ambiguous signal handling**: `references/inference-playbook.md`
- **Quick field→file lookup, OS families, version rules, known namespaces**: `assets/quick-reference.md`
