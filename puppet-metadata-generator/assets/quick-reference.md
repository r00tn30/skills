# Puppets — Quick Reference

## THE CORE RULE (internalize this first)

> **1.** Read the codebase before writing a single field. Never ask for values that exist in files.
> **2.** PDK owns `pdk-version`, `template-url`, `template-ref` — never touch them.

---

## INSPECTION ORDER (run all, do not skip)

```
 1. metadata.json           (existing values to preserve)
 2. .git/config             (name, author, source, project_page, issues_url)
 3. CHANGELOG.md            (version)
 4. LICENSE / LICENSE.md    (license)
 5. README.md               (summary, tags)
 6. manifests/init.pp       (summary fallback — @summary tag)
 7. .fixtures.yml           (dependencies — priority 1)
 8. Puppetfile              (dependencies — priority 2)
 9. manifests/**/*.pp       (dependencies cross-refs, OS conditionals)
10. Gemfile                 (requirements — priority 1)
11. .github/workflows/*.yml (requirements — priority 2)
12. .travis.yml             (requirements — priority 3)
13. .sync.yml / pdk.yaml    (requirements — priority 4)
14. spec/acceptance/*.yml   (operatingsystem_support — nodesets)
15. Vagrantfile             (operatingsystem_support — box names)
```

---

## FILE → FIELD MAPPING TABLE

| File | Fields it informs |
|---|---|
| `metadata.json` (existing) | All (preserves non-placeholder values) |
| `.git/config` / git remote | `name`, `author`, `source`, `project_page`, `issues_url` |
| `CHANGELOG.md` | `version` |
| `LICENSE` / `LICENSE.md` | `license` |
| `README.md` | `summary`, `tags` |
| `manifests/init.pp` | `summary` (@summary tag fallback), `tags` |
| `manifests/**/*.pp` | `dependencies` (cross-module refs), `operatingsystem_support` |
| `.fixtures.yml` | `dependencies` (priority 1) |
| `Puppetfile` | `dependencies` (priority 2) |
| `Gemfile` | `requirements` (priority 1) |
| `.github/workflows/*.yml` | `requirements` (priority 2) |
| `.travis.yml` | `requirements` (priority 3) |
| `.sync.yml` / `pdk.yaml` | `requirements` (priority 4) |
| `spec/acceptance/*.yml` / `.nodesets.yml` | `operatingsystem_support` (platforms) |
| `Vagrantfile` | `operatingsystem_support` (box names) |

---

## PDK-OWNED FIELDS — NEVER WRITE THESE

```
pdk-version
template-url
template-ref
```

If present in existing `metadata.json`, copy them through unchanged.
Do not infer, do not overwrite.

---

## OS FAMILY → OS NAMES EXPANSION

| Family / `$osfamily` value | Puppet Forge OS Names to include |
|---|---|
| `RedHat` | `RedHat`, `CentOS`, `Rocky`, `AlmaLinux` |
| `Debian` | `Debian`, `Ubuntu` |
| `Suse` | `SLES`, `OpenSuSE` |
| `Archlinux` | `Archlinux` |
| `FreeBSD` | `FreeBSD` |
| `Darwin` | `Darwin` |
| `windows` | `windows` |
| `Solaris` | `Solaris` |
| `AIX` | `AIX` |

**Rule**: If `$osfamily == 'RedHat'` with no specific OS names → emit all four.
If specific OS names are also found (e.g., `CentOS` and `Rocky` only) → use
only those, not the full family expansion.

---

## DEFAULT OS RELEASE LISTS (use when no releases found in code)

| OS Name | Default releases |
|---|---|
| `RedHat` | `["7", "8", "9"]` |
| `CentOS` | `["7", "8"]` |
| `Rocky` | `["8", "9"]` |
| `AlmaLinux` | `["8", "9"]` |
| `OracleLinux` | `["7", "8", "9"]` |
| `Scientific` | `["7"]` |
| `Debian` | `["10", "11", "12"]` |
| `Ubuntu` | `["20.04", "22.04", "24.04"]` |
| `SLES` | `["12", "15"]` |
| `OpenSuSE` | `["15"]` |
| `windows` | `["2016", "2019", "2022"]` |
| `FreeBSD` | `["13", "14"]` |
| `Darwin` | `["13", "14"]` |

---

## KNOWN MODULE NAMESPACES — SHORTHAND RESOLUTION

When a `.fixtures.yml` key has no slash (e.g., `stdlib:`) apply these defaults:

| Short name | Full Forge name |
|---|---|
| `stdlib` | `puppetlabs-stdlib` |
| `concat` | `puppetlabs-concat` |
| `apt` | `puppetlabs-apt` |
| `apache` | `puppetlabs-apache` |
| `mysql` | `puppetlabs-mysql` |
| `postgresql` | `puppetlabs-postgresql` |
| `java` | `puppetlabs-java` |
| `tomcat` | `puppetlabs-tomcat` |
| `firewall` | `puppetlabs-firewall` |
| `ntp` | `puppetlabs-ntp` |
| `vcsrepo` | `puppetlabs-vcsrepo` |
| `inifile` | `puppetlabs-inifile` |
| `puppet_agent` | `puppetlabs-puppet_agent` |
| `archive` | `puppet-archive` |
| `module_data` | `choria-module_data` |
| `epel` | `stahnma-epel` |
| `profiles` | *(internal — flag as `REVIEW_NEEDED`)* |
| `roles` | *(internal — flag as `REVIEW_NEEDED`)* |

If not in this table and has no explicit author: emit `"UNKNOWN_AUTHOR-{name}"`
and flag as `REVIEW_NEEDED`.

---

## VERSION_REQUIREMENT FORMAT RULES

| Rule | Correct | Wrong |
|---|---|---|
| Use explicit range | `>= 7.0.0 < 9.0.0` | `>= 7.0.0` (no upper bound) |
| Never use tilde-arrow | `>= 8.0.0 < 9.0.0` | `~> 8.0` |
| Never pin exact | `>= 4.25.0 < 5.0.0` | `4.25.0` |
| Never use wildcards | `>= 4.0.0 < 5.0.0` | `4.*` |
| Upper bound = next major | `>= 8.5.0 < 9.0.0` | `>= 8.5.0 < 10.0.0` (too wide) |
| Minor lower bound is fine | `>= 8.5.0 < 9.0.0` | ✓ |
| Patch lower bound is fine | `>= 8.5.1 < 9.0.0` | ✓ |

**`~>` conversion table:**

| Found in fixtures/Puppetfile | Emit in metadata.json |
|---|---|
| `~> 8.0` | `>= 8.0.0 < 9.0.0` |
| `~> 8.5` | `>= 8.5.0 < 9.0.0` |
| `~> 8.5.0` | `>= 8.5.0 < 8.6.0` |
| `~> 4` | `>= 4.0.0 < 5.0.0` |

---

## KNOWN DEPENDENCY VERSION RANGES (starting points — verify on Forge)

| Forge Module | Use this range |
|---|---|
| `puppetlabs-stdlib` | `>= 8.0.0 < 10.0.0` |
| `puppetlabs-concat` | `>= 7.0.0 < 10.0.0` |
| `puppetlabs-apt` | `>= 8.0.0 < 10.0.0` |
| `puppetlabs-apache` | `>= 8.0.0 < 13.0.0` |
| `puppetlabs-mysql` | `>= 12.0.0 < 16.0.0` |
| `puppetlabs-postgresql` | `>= 8.0.0 < 11.0.0` |
| `puppetlabs-java` | `>= 7.0.0 < 11.0.0` |
| `puppetlabs-firewall` | `>= 3.0.0 < 7.0.0` |
| `puppetlabs-inifile` | `>= 5.0.0 < 7.0.0` |
| `puppetlabs-vcsrepo` | `>= 5.0.0 < 7.0.0` |
| `puppetlabs-ntp` | `>= 4.0.0 < 6.0.0` |
| `puppet-archive` | `>= 6.0.0 < 8.0.0` |
| `stahnma-epel` | `>= 1.3.0 < 4.0.0` |

These are starting points only. Always verify current versions at
`forge.puppet.com/{author}/{module}` before publishing.

---

## CONFLICT RESOLUTION DECISION TREE

```
Does metadata.json already have this field?
  │
  ├─ NO ──────────────────────────────────────────→ Infer and write
  │
  └─ YES
       │
       ├─ Is it a placeholder? ("", "FIXME", "0.0.1", "0.1.0",
       │  "username-module_name")
       │     │
       │     ├─ YES ──────────────────────────────→ Replace with inferred value
       │     │
       │     └─ NO
       │           │
       │           ├─ Is it a PDK field? (pdk-version, template-url,
       │           │  template-ref)
       │           │     │
       │           │     ├─ YES ──────────────────→ Preserve exactly, skip
       │           │     │
       │           │     └─ NO ─────────────────→ Keep existing value,
       │           │                               note as KEPT_EXISTING
       │           │                               in Review Summary
```

---

## SENTINELS AND FLAGS REFERENCE

| Situation | Emit in JSON | Note in Review Summary |
|---|---|---|
| Cannot infer a field | `"REVIEW_NEEDED: {reason}"` | Yes — explain why and how to fix |
| Existing value ≠ inferred | Keep existing | Yes — `KEPT_EXISTING: {field}` |
| Private dep (`:git` to internal host) | Omit from array | Yes — explain it was excluded |
| Unknown dep author | `"UNKNOWN_AUTHOR-{name}"` | Yes — ask user to confirm namespace |
| `~>` found in fixtures/Puppetfile | Convert (no sentinel) | No |
| license defaulting to Apache-2.0 | Emit `"Apache-2.0"` | Only if LICENSE file is absent |
