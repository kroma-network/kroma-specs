<div align="center">
  <br />
  <br />
  <a href="https://kroma.network"><img alt="Kroma network" src="https://raw.githubusercontent.com/kroma-network/kroma-brand-kit/main/assets/images/signature/Kroma-signature.svg" width=600></a>
  <br />
  <h3><a href="https://kroma.network">Kroma network</a> aims to be a New Universal ZK Rollup on Ethereum.</h3>
  <br />
</div>

## Kroma network Specification

This repository contains the [Specs Book](https://kroma-specs.kroma.network).

## Contributing

### Dependencies

**Rust Toolchain**

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

**`mdbook` + plugins**

```sh
cargo install mdbook mdbook-katex mdbook-linkcheck mdbook-mermaid
```

### Serving the book locally

```sh
just serve
```

### Linting

`doctoc` is used to automatically add a table of contents.

```sh
just lint-specs-toc-check
```

To fix markdown linting errors:

```sh
just lint-specs-md-fix
```

See the [markdownlint rule reference](https://github.com/DavidAnson/markdownlint/blob/main/doc/Rules.md)
and an example [config file](https://github.com/DavidAnson/markdownlint/blob/main/schema/.markdownlint.jsonc).

Justification for linting rules in
[.markdownlint.json](https://github.com/ethereum-optimism/specs/blob/main/.markdownlint.json):

- _line_length_ (`!strict && stern`): don't trip up on url lines
- _no-blanks-blockquote_: enable multiple consecutive blockquotes separated by white lines
- _single-title_: enable reusing `<h1>` for content
- _no-emphasis-as-heading_: enable emphasized paragraphs

To lint links:

```sh
just lint-links
```

[lychee][lychee] is used for linting links.
