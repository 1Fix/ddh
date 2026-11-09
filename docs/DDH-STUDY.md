<!-- Written by the abliteration.ai model "abliterated-model" on 2026-11-09 from the repository's own documents. Unreviewed model output: check it before acting on it. -->



# DDH Fork Engineering Report

## 1. How it works

Obscura (and by extension DDH) is a Rust workspace composed of ten specialized crates. The architecture separates concerns strictly to maintain performance and safety, particularly around the V8 runtime.

### Crate Responsibilities

| Crate | Role | Key Files/Modules |
| :--- | :--- | :--- |
| **`obscura-cli`** | Entry point for `fetch`, `serve`, `scrape`, `mcp`. Manages binary spawning. | `src/main.rs`, `src/worker.rs` |
| **`obscura-cdp`** | Chrome DevTools Protocol server. Handles WebSocket connections, routing, and CDP domain logic. | `server.rs`, `dispatch.rs`, `domains/` |
| **`obscura-browser`** | High-level `Page` logic. Navigation, lifecycle events, resource loading coordination. | `page.rs`, `lifecycle.rs` |
| **`obscura-js`** | V8 runtime wrapper (`deno_core`). Manages the Isolate, JS bindings, and `bootstrap.js`. | `runtime.rs`, `ops.rs`, `js/bootstrap.js` |
| **`obscura-dom`** | DOM tree implementation. Handles parsing, mutation, and tree traversal. | `tree.rs` |
| **`obscura-net`** | HTTP client logic. Handles cookies, robots.txt, and the stealth transport (`wreq`). | `client.rs`, `wreq_client.rs`, `cookies.rs` |
| **`obscura-ssrf`** | URL/IP validation for SSRF protection (private network blocking). | `lib.rs` |
| **`obscura-mcp`** | Model Context Protocol server for AI agents. | `lib.rs` |
| **`obscura-render`** | CSS layout, text shaping, and CPU-based paint (screenshots/PDF). | `layout/`, `paint/`, `style/` |
| **`obscura`** | Embeddable Rust library API (not the CLI). | `lib.rs` |

### Page Load Flow

1.  **Entry:** A request enters via **CLI** (`fetch`), **CDP** (`Page.navigate`), or **MCP** (`browser_navigate`).
2.  **Dispatch:**
    *   **CLI:** `obscura-cli` parses args and invokes `obscura-browser` directly.
    *   **CDP/MCP:** `obscura-cdp` or `obscura-mcp` receives the message, acquires the V8 lock, and delegates to `obscura-browser`.
3.  **Navigation:** `obscura-browser::Page` initiates navigation.
4.  **Network:** `obscura-net` fetches the document.
    *   If `--stealth` is active, it uses `wreq_client.rs` (BoringSSL) for TLS fingerprinting.
    *   Cookies and headers are managed here.
5.  **Parsing:** `obscura-dom` parses the HTML into a tree.
6.  **Scripting:** `obscura-js` runs inline scripts.
    *   `bootstrap.js` provides browser globals (`window`, `document`).
    *   `ops.rs` bridges JS calls to Rust (e.g., `fetch`, DOM mutation).
7.  **Rendering (Optional):** If the `render` feature is enabled:
    *   `obscura-render` calculates layout (Taffy) and paint (tiny-skia).
    *   Screenshots/PDFs are generated here.
8.  **Response:** The result (HTML, JSON, Screenshot) is sent back to the CLI/CDP/MCP client.

---

## 2. Stage 2 Rename Plan

To minimize merge conflicts while establishing the DDH identity, we will **rename binaries and public strings** but **keep internal crate names** (`obscura-*`). This ensures `use obscura_cdp::...` imports remain identical to upstream, reducing diff noise.

### Execution Order (Safe & Reversible)

#### 1. Cargo Binaries (The Foundation)
*   **File:** `crates/obscura-cli/Cargo.toml`
*   **Change:**
    ```toml
    [[bin]]
    name = "ddh"  # was "obscura"
    path = "src/main.rs"

    [[bin]]
    name = "ddh-worker"  # was "obscura-worker"
    path = "src/worker.rs"
    ```
*   **Reason:** This changes the output executable names.

#### 2. Worker Lookup Logic
*   **File:** `crates/obscura-cli/src/main.rs` (and potentially `src/worker.rs`)
*   **Action:** Search for hardcoded `"obscura-worker"`.
    *   *Inference:* The `scrape` command likely spawns the worker. If it uses `Command::new("obscura-worker")`, change to `"ddh-worker"`.
    *   *Better Approach:* Use `std::env::current_exe().with_file_name("ddh-worker")` to locate the sibling binary in the same directory, ensuring it works regardless of installation path.
*   **Reason:** The main binary must find the worker binary.

#### 3. Environment Variables (Breaking Change)
*   **Scope:** All docs, source code, and CI.
*   **Action:** Replace `OBSCURA_` with `DDH_`.
    *   `OBSCURA_ALLOW_PRIVATE_NETWORK` → `DDH_ALLOW_PRIVATE_NETWORK`
    *   `OBSCURA_CDP_TOKEN` → `DDH_CDP_TOKEN`
    *   `OBSCURA_TIMEZONE` → `DDH_TIMEZONE`
*   **Mitigation:** In `src/main.rs` or config parsing, check `DDH_` first, then fallback to `OBSCURA_` for one minor version to allow users to transition.
*   **Files:** `docs/Environment-variables.md`, `crates/obscura-cli/src/`, `crates/obscura-net/src/`.

#### 4. Release & Packaging Workflows
*   **File:** `.github/workflows/release.yml`
*   **Changes:**
    *   Update `matrix.name` variables (e.g., `ddh-x86_64-linux`).
    *   Update `dist/` directory paths in packaging steps.
    *   Update `actions/upload-artifact` names.
*   **File:** `.github/workflows/windows-package.yml`
*   **Changes:** Update fixture names and hash checks for `ddh-x86_64-windows`.
*   **File:** `scripts/ci/package_windows.ps1`
*   **Changes:** Update script logic to handle `ddh` binary names.

#### 5. Docker
*   **File:** `Dockerfile` (if exists) or `docs/OBSCURA-README.md` Docker section.
*   **Change:** Update image tag from `h4ckf0r0day/obscura` to `1Fix/ddh`.
*   **Action:** Create a new Docker Hub repo `1Fix/ddh`.

#### 6. Scoop Manifest
*   **File:** `scoop/obscura.json`
*   **Action:** Rename to `scoop/ddh.json`.
*   **Changes:** Update `version`, `url`, `hash`, and `bin` entries to point to `ddh` and `ddh-worker`.

#### 7. MCP Configuration
*   **File:** `docs/Use-the-MCP-server.md`
*   **Change:** Update example config:
    ```json
    {
      "mcpServers": {
        "ddh": { ... }
      }
    }
    ```
*   **File:** `crates/obscura-mcp/src/`
*   **Change:** Update any hardcoded server names or help text.

#### 8. Documentation & Strings
*   **Scope:** All `.md` files.
*   **Action:** Search/Replace "Obscura" with "DDH" (except where referring to upstream history).
*   **Action:** Update `--version` output in `crates/obscura-cli/src/main.rs`.

### What to Leave Alone (For Mergeability)
*   **Crate Names:** Keep `obscura-cli`, `obscura-cdp`, etc. in `Cargo.toml`.
*   **Module Imports:** Do not rename `use obscura_cdp::...` to `use ddh_cdp::...`.
*   **Internal Struct Names:** Keep `ObscuraState`, `ObscuraJsRuntime` unless they are public API.
*   **License:** Keep Apache-2.0 text (see Section 4).

---

## 3. Upstream Cleanup (Remove/Replace)

These items belong to the original Obscura project and should be removed or replaced to avoid confusion.

| Item | Location | Action |
| :--- | :--- | :--- |
| **Sponsors** | `README.md` (NodeMaven table) | Remove or replace with DDH sponsors. |
| **Cloudflare Story** | `README.md` (Kitesurf section) | Remove or footnote as "Inspired by". |
| **Obscura Cloud** | `README.md` (Hosted version) | Remove. |
| **Website Links** | `README.md` (obscura.sh) | Replace with `github.com/1Fix/ddh` or DDH site. |
| **Social Links** | `README.md` (Twitter/X, Cal) | Replace with DDH socials or remove. |
| **Badges** | `README.md` (Trendshift, Vercel OSS) | Remove or replace with DDH badges. |
| **Icon** | `README.md` (assets/icon.png) | Replace with Dodecahedron icon. |
| **Docker Image** | `README.md` (Docker section) | Update to `1Fix/ddh`. |
| **Scoop Manifest** | `scoop/obscura.json` | Rename to `ddh.json` and update URL. |
| **MCP Config** | `docs/Use-the-MCP-server.md` | Change server name to `ddh`. |
| **Env Vars** | `docs/Environment-variables.md` | Prefix with `DDH_`. |

---

## 4. Apache-2.0 Obligations

Since Obscura is Apache-2.0, DDH must comply with the following:

1.  **License Text:** Include the full Apache-2.0 license text in a `LICENSE` file.
2.  **Copyright Notice:** Retain the original copyright notice in source files (e.g., `// Copyright 2024 Obscura Authors`). You may add `// Copyright 2024 DDH Contributors` alongside it.
3.  **NOTICE File:** If the upstream has a `NOTICE` file, include its attributions in your `NOTICE` file. (The provided docs mention a `NOTICE` file in Stage 1, so ensure it is preserved).
4.  **State Changes:** If you modify files, add a notice in the file header stating that changes were made.
5.  **Trademarks:** You cannot use the "Obscura" trademark for your project name (DDH is fine). You must not imply endorsement by the original authors.

---

## 5. Upstream Merge Strategy

To keep pulling changes with minimal conflicts:

1.  **Add Upstream Remote:**
    ```bash
    git remote add upstream https://github.com/h4ckf0r0day/obscura.git
    ```
2.  **Keep Crate Names:** As noted in Section 2, do not rename crates (`obscura-*`). This ensures `use` statements match exactly.
3.  **Rebase, Don't Merge:**
    ```bash
    git fetch upstream
    git rebase upstream/main
    ```
    This keeps history linear.
4.  **Isolate Rename Commits:** Perform all DDH renaming in a dedicated branch (e.g., `ddh-identity`). Merge this into `main` once. Future rebases will only touch code logic, not identity strings.
5.  **Avoid Renaming Internal Types:** If you rename `ObscuraState` to `DdhState`, every upstream commit referencing it will conflict. Keep internal names unless they are part of the public API exposed to users.
6.  **Env Var Fallback:** Support both `DDH_` and `OBSCURA_` env vars during the transition period to avoid breaking user scripts that might copy upstream docs.

---

## 6. Risks and Unknowns

| Risk | Impact | Mitigation |
| :--- | :--- | :--- |
| **Worker Binary Hardcoding** | `scrape` command fails to find `ddh-worker`. | Check `crates/obscura-cli/src/` for `Command::new("obscura-worker")`. Prefer `current_exe().with_file_name()`. |
| **V8 Snapshot Strings** | V8 snapshot might contain "Obscura" strings. | Check `obscura-js` snapshot generation. If hardcoded, update or rebuild snapshot. |
| **Env Var Breaking** | Existing scripts using `OBSCURA_` break. | Support dual prefixes (`DDH_` preferred, `OBSCURA_` fallback) for one release cycle. |
| **Docker Layer Caching** | Renaming binaries breaks Docker layer cache. | Accept the rebuild cost; it's a one-time cost for the fork. |
| **Scoop Auto-Update** | Scoop users might get confused by version skew. | Bump version to `0.2.0` or `1.0.0` to signal a fork divergence. |
| **MCP Client Config** | Users have `obscura` in their MCP config. | Document the change in release notes; provide a migration snippet. |
| **Inferred:** | **Windows Runtime Dependencies** | The `windows-package.yml` script bundles DLLs. Ensure the script handles `ddh.exe` names correctly. |

---

## 7. Five Improvements to Set DDH Apart

1.  **Automatic System Font Loading**
    *   *Current State:* Docs say "System fonts are not scanned automatically." Users must pass `--font-dir`.
    *   *Improvement:* Auto-detect system font directories (Windows Fonts, macOS `/System/Library/Fonts`, Linux `/usr/share/fonts`) on startup.
    *   *Value:* Immediate "it just works" experience for screenshots/PDFs without manual config.

2.  **WASM Support**
    *   *Current State:* Not explicitly mentioned in features. V8 supports it, but browser shim might not.
    *   *Improvement:* Enable WebAssembly execution in `bootstrap.js`.
    *   *Value:* Enables running modern JS-heavy sites that rely on WASM for rendering or logic, increasing compatibility.

3.  **Native Proxy Rotation**
    *   *Current State:* Docs mention sticky sessions via proxy username.
    *   *Improvement:* Add a `--proxy-rotation` flag that rotates IPs per request or per page automatically using a configured pool.
    *   *Value:* Huge advantage for scraping high-volume targets without external proxy manager.

4.  **Headless Chrome Compatibility Patch**
    *   *Current State:* "Drop-in replacement" claims are ambitious.
    *   *Improvement:* Add a `--compat-chrome` flag that patches specific CDP responses to match Chrome's quirks exactly (e.g., `navigator.webdriver`, specific error codes).
    *   *Value:* Reduces friction for Puppeteer/Playwright users migrating from Chrome.

5.  **Distributed Scraping Mesh**
    *   *Current State:* `scrape` uses local workers.
    *   *Improvement:* Extend `ddh-worker` to support a `--mesh` mode where workers can be on different machines, coordinated by a central `ddh` master.
    *   *Value:* Scales scraping across a fleet of machines easily, fitting the "Dodecahedron" (multi-faceted) theme.
