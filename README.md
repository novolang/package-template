# package-template

The starting point for an Orbit package. Clone it, rename four things,
and you have a package with a complete manifest, a module, a test
suite, a licence, a changelog, and a CI job that installs the stable
toolchain on every push and publishes to
[Orbit](https://novo-lang.org/docs/orbit.html) on a `v*` tag.

It is also a working package: the module it ships parses `key=value`
lines, so `novo test` has something to run before you have written
anything.

```
novo pkg add package-template
```

```novo
use template

fn main() [io]
    match template.parse("host = example.com")
        Some(p) => println("${p.key} -> ${p.value}")   // host -> example.com
        None    => println("not a key=value line")
```

## Starting from it

```bash
git clone https://github.com/novolang/package-template my-package-nv
cd my-package-nv
rm -rf .git && git init
```

Then, in order:

| Change | Where |
|---|---|
| `name`, `description`, `category`, `tags`, `repository`, `maintainers` | `novo.toml` — every field is commented in place |
| the module's name | rename `src/template.nv`, and the `use template` in `src/main.nv` and `tests/` |
| the licence holder | `LICENSE`, last section |
| the first entry | `CHANGELOG.md` |
| the ignored binary's name | `.gitignore` |

`novo pkg categories` prints the categories you may choose from;
[Package categories](https://novo-lang.org/docs/registry/categories.html)
says what each one holds.

Two names, not one: the **package name** is what `novo pkg add` takes
and what the repository is called, and the **module name** is the
filename under `src/`, which is what `use` names. A port of a package
from another language keeps the upstream name with `-nv`; an original
takes a plain name.

## What it gives you

| Function | |
|---|---|
| `template.parse(line: Str) -> ?Pair` | one `key=value` line, or `None` |
| `template.format(p: Pair) -> Str` | a pair back out, in the form `parse` reads |
| `template.parse_all(text: Str) -> [Pair]` | every pair in a document, blanks and `#` comments skipped |

`Pair` is `{ key: Str, value: Str }`. The first `=` separates them, so a
value may contain one; both halves are trimmed. `parse` answers `None`
for a line with no `=` and for one whose key is empty — a setting with
no name is not a setting, and inventing one moves the error to whoever
reads `p.key`.

## The layout, and why it is this one

| Path | |
|---|---|
| `novo.toml` | the manifest. Required, and the only file the registry parses |
| `src/` | the sources. **Ships whole** — every module here is compiled into every consumer's program |
| `src/main.nv` | the entry point. A library declares one too, and this one says what to `use` instead |
| `tests/` | `@test` modules. **Never ships**, and `novo test` still runs them as members of the package |
| `README.md` | the front page, and the package's documentation index |
| `CHANGELOG.md` | what changed, per version — the only thing a consumer deciding whether to upgrade can read |
| `LICENSE` | Apache-2.0 |
| `.github/workflows/check.yml` | test on every push, publish on a `v*` tag |

Tests live under `tests/` rather than `src/` because `src/` ships whole:
a suite left there is built into every downstream binary along with
whatever it imports. Only what a module marks `pub` is visible to
another package, and a test module sees the package's internals whether
they are `pub` or not — the two rules together keep `pub` an honest list
of what you are promising.

**`novo.lock` is committed.** A locked build resolves the same closure
twice and does it offline once the cache is warm; a consumer never reads
a dependency's lock, because they resolve their own closure and write
their own. It is not on the publish allow-list, so it does not ship
either way.

## The workflow

`.github/workflows/check.yml` runs on every push and on every pull
request:

1. installs the stable toolchain from
   `https://novo-lang.org/releases/install.sh`;
2. `novo pkg build` — the package compiles;
3. `novo test tests` — every `@test` module under `tests/`;
4. `novo fmt --check` — the sources are formatted;
5. the **shard audit**, `--strict`, fetched from
   `https://novo-lang.org/tools/`. It is the bar first-party packages
   are held to: formatting, documented public functions, tested public
   functions, no dead imports, pipelines where the standard library
   offers one. When novo-lang.org is not serving it, the step says so
   and does not fail — a bar you cannot fetch is not a bar you can
   enforce, and failing the build over it would teach a package author
   nothing.

On a `v*` tag it also publishes:

```bash
novo pkg publish
```

reading the token from a repository secret named `NOVO_REGISTRY_TOKEN`
(Settings → Secrets and variables → Actions). The tag's version and the
manifest's must agree — the job checks and refuses otherwise, because a
publish is permanent and a mismatched tag is the one mistake that cannot
be taken back.

The publish records **provenance**: the commit the tag names and the
package's path inside the checkout. A CI checkout is clean, so the
commit is a true claim about the bytes; publishing from a dirty tree is
refused unless you pass `--allow-dirty`, which records that the commit
does not describe the bytes.

## What it costs

Nothing at runtime that you do not write. `parse` allocates the two
trimmed strings and the `Pair`; `parse_all` allocates one list plus a
pair per line it keeps. No dependency, no effect but the ones your own
code declares — the three functions here carry none, so they run on
every tier and inside a pure caller.

## What it does not do

- **It is not a project scaffold.** `novo pkg init` scaffolds a program
  in place; this is a repository you clone when the thing you are
  writing will be published.
- **It does not choose your category.** The list is short and flat and
  the guidance is on one page, because a category assigned by a script
  is a category nobody read.
- **It does not vendor the shard audit.** The audit is one script in the
  toolchain's repository, fetched at CI time; a copy in every package
  repository would be a copy that drifts.

## Tests

```bash
novo test tests            # every @test module under tests/
novo test tests/template_tests.nv
```

## Licence

Apache-2.0. See `LICENSE`.
