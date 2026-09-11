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
| the module's name | rename `src/template.nv`, and the `use template` in `tests/` |
| the licence holder | `LICENSE`, last section |
| the first entry | `CHANGELOG.md` |
| the ignored binary's name, if you make it a program | `.gitignore` |

`novo pkg categories` prints the categories you may choose from;
[Package categories](https://novo-lang.org/docs/registry/categories.html)
says what each one holds.

Two names, not one: the **package name** is what `novo pkg add` takes
and what the repository is called, and the **module name** is the
filename under `src/`, which is what `use` names. A port of a package
from another language keeps the upstream name with `-nv`; an original
takes a plain name.

## What it gives you

The API and the dependencies are on
[the package's page](https://novo-lang.org/packages), generated from
these sources at every publish: every `pub` declaration with its
signature, its effect row and the comment block written above it.

This README is the part a generator cannot write — what the package is
for, why it is shaped this way, and what it deliberately does not do.
A table of function names here would be a second original, and the
second original is the one that goes stale.

## The layout, and why it is this one

| Path | |
|---|---|
| `novo.toml` | the manifest. Required, and the only file the registry parses. It names **no** `main`, which is what makes this a library |
| `src/` | the sources. **Ships whole** — every module here is compiled into every consumer's program |
| `tests/` | `@test` modules. **Never ships**, and `novo test` still runs them as members of the package |
| `bugs/` | known defects, one Markdown file each, written by `novo bugs new` and checked by `novo bugs audit`. `novo bugs pull` puts what strangers filed about this package into `bugs/inbox/`, where it waits until you read it and `novo bugs accept` it, and `novo bugs sync` tells those reporters what you decided. **Never ships** |
| `README.md` | the front page: prose, and a link to the generated reference |
| `CHANGELOG.md` | what changed, per version — the only thing a consumer deciding whether to upgrade can read |
| `LICENSE` | Apache-2.0 |
| `.github/workflows/check.yml` | test on every push, publish on a `v*` tag |

**It is a library, not a program.** The manifest names no `main` and
there is no `src/main.nv`, so `novo pkg build` type-checks and
effect-checks every module and writes no binary, and `novo test` runs
the suite. A consumer gets the modules under `src/` and nothing else:
a dependency's entry file is never read and its `main` is never built
into anybody's program. To make it a program instead, write
`src/main.nv` and add `main = "src/main.nv"` to `[package]` — both
halves, because a manifest that names an entry it does not ship is an
error rather than a library.

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
3. `novo test tests` — every `@test` module under `tests/`, one
   summary, and a non-zero exit if any module failed;
4. `novo bugs audit` — the filings under `bugs/` are well formed. The
   toolchain owns that schema, so there is no script here to keep in
   step with it, and a repository with no filings passes;
5. `novo doc` — the reference is generated from the sources, and every
   ```novo block in a documentation comment is compiled against the
   package;
6. the **shard audit**, `--strict`, fetched from
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

## The reference, and the examples in it

The comment block directly above a `pub` declaration is that
declaration's documentation. This is Go's rule and there is no new
syntax to learn: no `///`, no `/** */`, no `@param` — the signature and
the effect row already say what a tag would restate.

```novo ignore
// Parse one `key=value` line.
//
// ```novo
// match template.parse("host = example.com")
//     Some(p) => println("${p.key} -> ${p.value}")
// ```
pub fn parse(line: Str) -> ?Pair
```

A blank line between the block and the declaration means the
declaration has no documentation. A fenced ```novo block inside one is
an EXAMPLE, and it is checked rather than trusted:

```bash
novo doc                       # write APIDOC.md; compile every example
novo test src/template.nv      # run them, one test per block
novo pkg publish               # refuses over a block that does not compile
```

An example with no `fn main` is wrapped in one and given a `use` of its
module, so the block is the lines a reader would type and nothing else;
one that needs an effect beyond `[io]` writes its own `main`. A block
that is illustration rather than code is fenced ```novo ignore, and
every run lists it as skipped.

`APIDOC.md` is generated, gitignored and never shipped as a source: it
travels to the registry as its own upload beside the archive, and
`novo-lang.org/packages/<name>` serves it above this README.

## Tests

```bash
novo test tests                       # every @test module under tests/
novo test tests/template_tests.nv     # one module
novo test src/template.nv             # the examples in the documentation
```

`novo test <dir>` walks the directory in a stable order, runs each `.nv`
file that carries an `@test` annotation, and prints one summary over the
modules; it exits non-zero if any of them failed. A directory with no
`@test` module at all is an error, so a suite that quietly stopped being
collected cannot pass as a green run. That is what the workflow runs.

## Licence

Apache-2.0. See `LICENSE`.
