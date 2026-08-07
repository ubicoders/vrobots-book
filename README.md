# VRobots Book

Source for **The VRobots SDK Book**, the tutorial and reference for driving Ubicoders
virtual robots from code.

**Read it online: <https://ubicoders.github.io/vrobots-book/>**

## About

`vrobots_sdk` is the Rust SDK for controlling Ubicoders virtual robots running in a Unity
simulator. It speaks zenoh for state, commands and services, and iceoryx2 for camera
frames, with FlatBuffers on the wire in both directions. The C++ and Python SDKs are thin
bindings over the same Rust crate, so all three surfaces share one lifecycle, one set of
snapshots, and one set of stable error codes.

This repository contains only the book. It is built with
[mdBook](https://rust-lang.github.io/mdBook/) and published to GitHub Pages.

The current text documents SDK 0.1.4 against simulator v3.0.0.

## Contents

| Part | Chapters |
|---|---|
| Tutorial | 1 Getting started, 2 Concepts |
| The API in four slices | 3 Reading state, 4 Sending commands, 5 Cameras and images, 6 Services and configuration |
| Reference | 7 Supported virtual robots |
| Diagnostics | 8 Tooling and diagnostics |
| Appendices | A Topics, B Commands, C Errors, D Glossary |

Chapters 3 to 6 are independent of each other, but all of them lean on the five rules set
out in chapter 2.

## Building locally

Install mdBook, then build or serve:

```sh
cargo install mdbook

mdbook build          # writes the static site to ./book
mdbook serve --open   # live-reloading preview on http://localhost:3000
```

The generated `book/` directory is not tracked in git.

## Repository layout

```
book.toml                  mdBook configuration
src/SUMMARY.md             table of contents; a page is only built if listed here
src/introduction.md        preface
src/ch01-getting-started/  ... through ch08-tooling/
src/appendix-*.md          lookup tables
.github/workflows/         build and deploy to GitHub Pages
```

## Contributing

1. Add or edit Markdown under `src/`.
2. Register any new page in `src/SUMMARY.md`. mdBook ignores files that are not listed.
3. Run `mdbook build` and confirm it completes without warnings.
4. Open a pull request against `main`.

Code blocks in the book are copied from the SDK examples or from signatures in the SDK
source, so that everything shown compiles as written. Please keep that property when
editing.

## Deployment

Pushing to `main` triggers
[.github/workflows/mdbook.yml](.github/workflows/mdbook.yml), which builds the book and
deploys it to GitHub Pages. The workflow pins mdBook to the version named in its
`MDBOOK_VERSION` environment variable; if your local mdBook is newer, treat CI as the
authority.

## Versioning

Tags mark book releases against the simulator version they document.

| Tag | Documents |
|---|---|
| `v3.0.0` | simulator v3.0.0, SDK 0.1.4 |
| `v2.0` | the previous edition |
