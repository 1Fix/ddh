# DDH (Dodecahedron)

DDH is our own headless browser for AI agents and web automation. It runs real JavaScript, keeps a real DOM,
renders pages itself (no Chromium needed) and speaks the Chrome DevTools Protocol, so Puppeteer and Playwright
can drive it. It also ships an MCP server, which is how the REA Browser agent controls it.

DDH is a fork of [Obscura](https://github.com/h4ckf0r0day/obscura) by h4ckf0r0day, used under the Apache-2.0
licence. See [NOTICE](NOTICE) for what we changed. The original project README is kept at
[docs/OBSCURA-README.md](docs/OBSCURA-README.md), and the guides in [docs/](docs/) still apply.

## Status

- **Stage 1 (done):** our own repository, name and front page.
- **Stage 2 (in progress):** the programs are now `ddh` and `ddh-worker`, and Windows builds come from this
  repository's own "DDH Windows build" workflow. Releases appear under
  [Releases](https://github.com/1Fix/ddh/releases) once a build has passed.
- Not renamed yet: the internal crate names (`obscura-*`), the `OBSCURA_*` environment variables, the guides in
  `docs/`, and the Docker and Scoop files. They still say Obscura, which keeps merging upstream updates easy.

## Use it

With the DDH Browser agent (github.com/1Fix/REA, folder `browser/`):

```
browse.cmd "Open https://example.com and tell me the page title"
```

Directly:

```
ddh fetch https://example.com --dump text
ddh serve --port 9222
ddh mcp
```

## Build from source

Needs Rust 1.75 or newer. The first build compiles V8 and takes several minutes.

```
cargo build --release -p obscura-cli --bins --features render
```

More detail: [docs/Build-from-source.md](docs/Build-from-source.md) and [AGENTS.md](AGENTS.md).

## Keeping up with upstream

```
git remote add upstream https://github.com/h4ckf0r0day/obscura.git
git fetch upstream
git merge upstream/main
```

## Licence

Apache-2.0, the same as Obscura. See [LICENSE](LICENSE).
