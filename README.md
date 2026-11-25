# `billionaire` — agent-friendly CLI for the billionaire platform

Built after [`order3000`](../../order3000/cli): the same command
conventions (citty), the same output contract, the same exit codes.

Output adapts automatically: text for people when stdout is a terminal,
JSON otherwise — an agent or a script gets JSON without a flag. Force
either with `--json` / `--pretty`.

Exit codes: `0` ok · `1` user error · `99` unexpected. Branch on the exit
code, never on stderr text.

---

## Install via Homebrew (macOS — Apple Silicon or Intel)

This is the recommended path. The Homebrew binary bundles the runtime and
the chart data, so you don't need to install Node yourself.

```sh
# 1. Add the billionaire tap (one-time setup)
brew tap billionaire-sh/tap

# 2. Install the CLI
brew install billionaire

# 3. Verify
billionaire --version
```

That's it. From here, `billionaire` is on your `$PATH` like any other
Homebrew binary. To upgrade to the latest release later:
`brew upgrade billionaire`.

> **Linux Homebrew users**: the same two commands work.

Or via npm:

```sh
pnpm i -g @billionaire-sh/cli
```

---

## From the checkout: always the code as it stands

Homebrew and npm install a build, and a build is behind the checkout: a
command added today is not in the binary of last week. Whoever works in the
repository links the starter into a folder of the `PATH` once:

```sh
ln -s ~/Projects/kadal/apps/billionaire/cli/bin/billionaire.mjs /opt/homebrew/bin/billionaire
billionaire --version
```

From then on `billionaire` runs `src/cli.ts` of that checkout — no build,
no release, nothing to upgrade. It needs the checkout installed
(`pnpm install`) and nothing else. The commands that work on the files of
the chart editions (`charts lock`, `charts seal`, `charts parse`,
`charts check`) look at the folder you call them from, so call them inside
the checkout whose files you mean.

The starter is three lines (`bin/billionaire.mjs`); the mechanism is
`@kolaveri/dev-tools/cli-from-source`, shared with the `order3000` CLI.

---

## Shell completion (tab autocomplete)

`billionaire completion <shell>` prints a completion script for your
shell. Pipe it to your shell's completions directory once and tab
autocomplete works on every `billionaire …` invocation thereafter.

```sh
# zsh — Homebrew on macOS
billionaire completion zsh > "$(brew --prefix)/share/zsh/site-functions/_billionaire"
autoload -Uz compinit && compinit

# bash
billionaire completion bash > /usr/local/etc/bash_completion.d/billionaire

# fish
billionaire completion fish > ~/.config/fish/completions/billionaire.fish
```

---

## From source (this monorepo)

```sh
# Build the CLI from a local checkout
pnpm --filter @billionaire-sh/cli build

# Run the just-built binary
node apps/billionaire/cli/dist/cli.js --help

# Or compile to a single-file native binary (this Mac)
pnpm --filter @billionaire-sh/cli build:binaries darwin-arm64
./apps/billionaire/cli/dist/binaries/billionaire-darwin-arm64 --help
```

---

## `charts` — the reference for DATEV account numbers

DATEV reserves numbers and drops accounts from one edition to the next. A
number from memory or from a web page is not a source. **Ask this before a
DATEV number goes into code, a spec, a fixture or a booking.**

```sh
# What is on these numbers?
billionaire charts account skr-03 4930 1579 1330

# Which account is it? Words of the name, German or English.
billionaire charts account skr-04 "innergemeinschaftlichem Erwerb"

# For scripts and hooks: exit 1 unless every number is an active account.
billionaire charts account skr-03 4930 1574 --check

# A chart is published anew from time to time. Without more, the newest
# edition answers; ask a named edition, or the one valid on a day — the
# edition a booking of that day stands on. A day before a chart's first
# edition answers with an error: a later edition does not stand in.
billionaire charts account skr-03 4930 --edition 2026
billionaire charts account skr-03 4930 --on 2026-03-31

# Which charts, which editions, valid from when, from which publication?
billionaire charts list
```

An edition's data is produced from its source documents by a parser
(`libs/billionaire/charts/python`). Inside a checkout of the repository:

```sh
# Write the edition's printed.json from the documents in its source/ folder.
billionaire charts parse skr-03 2026

# Exit 1 unless the checked-in printed.json is exactly what the parser gives.
billionaire charts check skr-03 2026
```

Both run the parser package in the repository's uv workspace; the installed
CLI outside a checkout answers that the documents are in the repository.

What follows from the editions' files is derived and checked in as a lock
(`libs/billionaire/charts/README.md`, "Succession" and "Mappings"): what
becomes of each account from one edition to the next, the counterpart of
each account on the other chart line, its place in each taxonomy edition,
and the rules its purposes stand for. A change of an account in
`meaning.json` ends with `lock`, and the diff of the locks is part of it:

```sh
# Write every lock and each chart line's CHANGES.md.
billionaire charts lock

# Exit 1 unless every checked-in lock is exactly what the derivation gives.
billionaire charts lock --check
```

The chart data is sealed by the owner (`libs/billionaire/charts/README.md`,
"The seal"): every file of the editions has its hash in `seal.json`, signed
with a key whose passphrase only the owner knows. A change of the chart
data — and of the locks that follow from it — waits for that seal:

```sh
# Exit 1 unless every file of the chart data is what the owner sealed.
# Needs no key.
billionaire charts seal --check

# The owner's: show what changed since the last seal — data as a table
# (account, field, was, is), locks as pairs of lines — then sign.
# ssh-keygen asks for the passphrase of the sealing key on the terminal.
# A seal that verifies writes its changeset (.changeset/chart-seal-…md).
billionaire charts seal                          # the key at ~/.ssh/billionaire/chart_seal
billionaire charts seal --key /path/to/key       # a key file somewhere else
billionaire charts seal --refs BLNR-1937         # stories the changeset names beside the ones git gives

# The changeset of the seal that stands, for a seal made without one.
# Signs nothing, needs no key.
billionaire charts seal --changeset [--refs BLNR-1937,BLNR-1943]
```

A number answers with one of:

| `status` | Meaning |
|---|---|
| `account` | The chart keeps an active account there — the only answer that may be booked on. Comes with name, account type, the booking purposes it answers (with their conditions) and taxonomy position. |
| `inactive` | The platform keeps the account, but it is no longer booked on. |
| `reserved` | DATEV reserves the number (with the block and the page of the publication). Not an account. |
| `unknown` | The chart does not know the number. An account of an entity's own may take it. |

### Where the data comes from

The charts are inlined at build time from the ONE place the platform keeps
them — the library `libs/billionaire/charts` (`editions/<provider>/<chart>/<edition>/`: what the source prints and what
we decide, beside the source PDFs), taken from the DATEV Kontenrahmen named in
each file's `edition`. The CLI holds no copy and no number of its own.

The CLI has no database and no network, so it loads none: a database is
loaded from the same files by the app, which holds the connection
(`apps/billionaire/billionaire/scripts/typescript/apply-chart-edition.ts`),
and held equal to them by `check-chart-editions-against-database.ts` —
`libs/billionaire/charts/README.md`, "Loading an edition into a database".
