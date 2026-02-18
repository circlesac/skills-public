---
name: cross-platform-cli
description: Guide for building and distributing Rust CLIs across npm, Homebrew, and GitHub Releases — use when scaffolding a new CLI project or setting up release pipelines
---

# Cross-Platform CLI Distribution

How to structure a Rust CLI so it can be installed via `cargo install`, `npm install -g`, and `brew install` — all from the same repo, using GitHub Actions for automated releases.

Real example: [oneup](https://github.com/circlesac/oneup) (CalVer version management CLI).

## Project Structure

```
my-cli/
├── Cargo.toml              # Rust crate metadata + dependencies (version = "0.0.0")
├── Cargo.lock
├── src/
│   └── main.rs             # CLI entry point (clap for arg parsing)
├── package.json            # npm wrapper — postinstall downloads the binary (versionless)
├── bin/
│   ├── my-cli              # Node shim that launches the native binary
│   ├── install.js          # Binary downloader (runs on npm install)
│   └── install.sh          # Standalone curl-pipe installer for direct install
├── homebrew/
│   └── my-cli.rb.template  # Homebrew formula with placeholder SHAs
├── .github/
│   └── workflows/
│       └── release.yml     # Multi-job release pipeline
└── .npmrc                  # Scoped registry config (if needed)
```

Three distribution channels from one repo. No versions in git — oneup fills them at release time.

`.npmrc` is needed when a scope is used across multiple registries (e.g. publishing `@org/my-cli` to the public npm registry while also using `@org/*` packages from a private registry). It pins the scope to the correct registry. Unscoped packages or scopes that only use one registry don't need it.

## npm Wrapper

The npm package doesn't bundle the binary — it downloads the platform-specific build from GitHub Releases on `postinstall`.

### package.json

```json
{
  "name": "@scope/my-cli",
  "description": "What my CLI does",
  "bin": { "my-cli": "bin/my-cli" },
  "files": ["bin"],
  "scripts": { "postinstall": "node bin/install.js" }
}
```

No `"version"` field — oneup writes it before `npm publish`. `files` controls what gets published to npm. Only `bin/` is included — Rust source stays out.

If the repo also ships skills (has `pi.skills`), add `"skills"` to `files` so the skill files get published to npm:

```json
{
  "files": ["bin", "skills"],
  "pi": { "skills": ["./skills"] }
}
```

### bin/my-cli (Node shim)

```js
#!/usr/bin/env node
const { spawnSync } = require("child_process");
const { existsSync } = require("fs");
const path = require("path");

const ext = process.platform === "win32" ? ".exe" : "";
const bin = path.join(__dirname, "native", `my-cli${ext}`);

(async () => {
  if (!existsSync(bin)) {
    await require("./install.js");
  }
  const result = spawnSync(bin, process.argv.slice(2), { stdio: "inherit" });
  process.exit(result.status ?? 1);
})();
```

On first run, if the native binary is missing, it triggers the installer. Otherwise it delegates directly to the native binary.

### bin/install.js (Binary downloader)

```js
const https = require("https");
const fs = require("fs");
const path = require("path");
const { execSync } = require("child_process");

const { version } = require("../package.json");
const REPO = "org/my-cli";

const PLATFORMS = {
  "darwin-x64":  { artifact: "my-cli-x86_64-apple-darwin",       ext: ".tar.gz" },
  "darwin-arm64": { artifact: "my-cli-aarch64-apple-darwin",      ext: ".tar.gz" },
  "linux-x64":   { artifact: "my-cli-x86_64-unknown-linux-gnu",  ext: ".tar.gz" },
  "linux-arm64":  { artifact: "my-cli-aarch64-unknown-linux-gnu", ext: ".tar.gz" },
  "win32-x64":   { artifact: "my-cli-x86_64-pc-windows-msvc",    ext: ".zip"    },
};

async function download(url) {
  return new Promise((resolve, reject) => {
    https.get(url, (res) => {
      if (res.statusCode === 302 || res.statusCode === 301) {
        return download(res.headers.location).then(resolve).catch(reject);
      }
      if (res.statusCode !== 200) return reject(new Error(`HTTP ${res.statusCode}`));
      const chunks = [];
      res.on("data", (c) => chunks.push(c));
      res.on("end", () => resolve(Buffer.concat(chunks)));
      res.on("error", reject);
    });
  });
}

async function main() {
  const platform = `${process.platform}-${process.arch}`;
  const info = PLATFORMS[platform];
  if (!info) {
    console.error(`Unsupported platform: ${platform}`);
    process.exit(1);
  }

  const { artifact, ext } = info;
  const url = `https://github.com/${REPO}/releases/download/v${version}/${artifact}${ext}`;
  const data = await download(url);
  const nativeDir = path.join(__dirname, "native");
  fs.mkdirSync(nativeDir, { recursive: true });

  const tmp = path.join(nativeDir, `tmp${ext}`);
  fs.writeFileSync(tmp, data);

  if (ext === ".zip") {
    execSync(`powershell -Command "Expand-Archive -Force '${tmp}' '${nativeDir}'"`, { cwd: nativeDir });
  } else {
    execSync(`tar xzf "${tmp}"`, { cwd: nativeDir });
  }
  fs.unlinkSync(tmp);

  if (process.platform !== "win32") {
    fs.chmodSync(path.join(nativeDir, "my-cli"), 0o755);
  }
}

module.exports = main();
```

Uses the version from package.json to construct the GitHub Release download URL. At install time, the published package.json already has the version (oneup wrote it before `npm publish`). No npm dependencies needed — just Node builtins.

### bin/install.sh (Standalone installer)

For users who don't use npm or Homebrew — a curl-pipe installer that downloads the latest release directly:

```bash
#!/bin/sh
set -e

REPO="org/my-cli"
INSTALL_DIR="${INSTALL_DIR:-/usr/local/bin}"

OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)

case "$OS-$ARCH" in
  darwin-arm64)  TARGET="aarch64-apple-darwin" ;;
  darwin-x86_64) TARGET="x86_64-apple-darwin" ;;
  linux-aarch64) TARGET="aarch64-unknown-linux-gnu" ;;
  linux-x86_64)  TARGET="x86_64-unknown-linux-gnu" ;;
  *) echo "Unsupported platform: $OS-$ARCH"; exit 1 ;;
esac

VERSION=$(curl -fsSL "https://api.github.com/repos/$REPO/releases/latest" | grep '"tag_name"' | cut -d'"' -f4)
URL="https://github.com/$REPO/releases/download/$VERSION/my-cli-$TARGET.tar.gz"

echo "Installing my-cli $VERSION..."
curl -fsSL "$URL" | tar xz -C "$INSTALL_DIR"
chmod +x "$INSTALL_DIR/my-cli"
echo "Installed to $INSTALL_DIR/my-cli"
```

The script is attached to each GitHub Release as an asset (see workflow below). Usage:

```bash
curl -fsSL https://github.com/org/my-cli/releases/latest/download/install.sh | sh
```

## Homebrew Formula Template

Lives at `homebrew/my-cli.rb.template`. The release workflow substitutes placeholders with real SHA256 checksums.

```ruby
class MyCli < Formula
  desc "What my CLI does"
  homepage "https://github.com/org/my-cli"
  version "VERSION_PLACEHOLDER"
  license "MIT"

  on_macos do
    on_arm do
      url "https://github.com/org/my-cli/releases/download/v#{version}/my-cli-aarch64-apple-darwin.tar.gz"
      sha256 "SHA_DARWIN_ARM64"
    end
    on_intel do
      url "https://github.com/org/my-cli/releases/download/v#{version}/my-cli-x86_64-apple-darwin.tar.gz"
      sha256 "SHA_DARWIN_X64"
    end
  end

  on_linux do
    on_arm do
      url "https://github.com/org/my-cli/releases/download/v#{version}/my-cli-aarch64-unknown-linux-gnu.tar.gz"
      sha256 "SHA_LINUX_ARM64"
    end
    on_intel do
      url "https://github.com/org/my-cli/releases/download/v#{version}/my-cli-x86_64-unknown-linux-gnu.tar.gz"
      sha256 "SHA_LINUX_X64"
    end
  end

  def install
    bin.install "my-cli"
  end

  test do
    system "#{bin}/my-cli", "--help"
  end
end
```

The formula is committed to a separate `homebrew-tap` repo (e.g. `org/homebrew-tap`). Users install via:

```bash
brew install org/tap/my-cli
```

## GitHub Actions Release Workflow

A multi-job pipeline triggered manually via `workflow_dispatch`. No version commits — the version job calculates the next version once, writes it to target files, and shares them via artifact upload. Downstream jobs download the versioned files instead of running oneup independently.

```
version → build (matrix) → github-release → publish-crates
                                           → publish-npm
                                           → publish-homebrew
```

`softprops/action-gh-release` creates the tag via `tag_name`, so no separate tag job is needed.

### Full workflow

```yaml
name: Release
on:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.bump.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Calculate version
        id: bump
        run: |
          VERSION=$(npx --yes @circlesac/oneup version --target Cargo.toml --target package.json | tail -1)
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      - uses: actions/upload-artifact@v4
        with:
          name: versioned-files
          path: |
            Cargo.toml
            package.json

  build:
    needs: version
    strategy:
      matrix:
        include:
          - target: x86_64-apple-darwin
            os: macos-latest
            name: my-cli-x86_64-apple-darwin
          - target: aarch64-apple-darwin
            os: macos-latest
            name: my-cli-aarch64-apple-darwin
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
            name: my-cli-x86_64-unknown-linux-gnu
          - target: aarch64-unknown-linux-gnu
            os: ubuntu-latest
            name: my-cli-aarch64-unknown-linux-gnu
          - target: x86_64-pc-windows-msvc
            os: windows-latest
            name: my-cli-x86_64-pc-windows-msvc

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          name: versioned-files

      - name: Install cross-compilation tools
        if: matrix.target == 'aarch64-unknown-linux-gnu'
        run: sudo apt-get install -y gcc-aarch64-linux-gnu

      - name: Add target
        run: rustup target add ${{ matrix.target }}

      - name: Build
        run: cargo build --release --target ${{ matrix.target }}
        env:
          CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER: aarch64-linux-gnu-gcc

      - name: Package (Unix)
        if: matrix.os != 'windows-latest'
        run: |
          cd target/${{ matrix.target }}/release
          tar czf ../../../${{ matrix.name }}.tar.gz my-cli
          cd ../../..

      - name: Package (Windows)
        if: matrix.os == 'windows-latest'
        run: |
          cd target/${{ matrix.target }}/release
          7z a ../../../${{ matrix.name }}.zip my-cli.exe
          cd ../../..
        shell: bash

      - uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.name }}
          path: ${{ matrix.name }}.*

  github-release:
    needs: [version, build]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          path: artifacts

      - uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ needs.version.outputs.version }}
          generate_release_notes: true
          files: |
            artifacts/**/*.tar.gz
            artifacts/**/*.zip
            bin/install.sh

  publish-crates:
    needs: [version, github-release]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          name: versioned-files

      - run: cargo publish --allow-dirty
        env:
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}

  publish-npm:
    needs: [version, github-release]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write    # Required for npm Trusted Publishing (OIDC)
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "lts/*"
          registry-url: https://registry.npmjs.org

      - uses: actions/download-artifact@v4
        with:
          name: versioned-files

      - run: npm publish

  publish-homebrew:
    needs: [version, github-release, build]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          path: artifacts

      - name: Update Homebrew formula
        env:
          GH_TOKEN: ${{ secrets.HOMEBREW_TAP_TOKEN }}
        run: |
          VERSION=${{ needs.version.outputs.version }}
          SHA_DARWIN_ARM64=$(shasum -a 256 artifacts/my-cli-aarch64-apple-darwin/*.tar.gz | cut -d' ' -f1)
          SHA_DARWIN_X64=$(shasum -a 256 artifacts/my-cli-x86_64-apple-darwin/*.tar.gz | cut -d' ' -f1)
          SHA_LINUX_ARM64=$(shasum -a 256 artifacts/my-cli-aarch64-unknown-linux-gnu/*.tar.gz | cut -d' ' -f1)
          SHA_LINUX_X64=$(shasum -a 256 artifacts/my-cli-x86_64-unknown-linux-gnu/*.tar.gz | cut -d' ' -f1)

          git clone https://x-access-token:${GH_TOKEN}@github.com/org/homebrew-tap.git
          cd homebrew-tap

          cp ../homebrew/my-cli.rb.template Formula/my-cli.rb
          sed -i "s/VERSION_PLACEHOLDER/${VERSION}/g" Formula/my-cli.rb
          sed -i "s/SHA_DARWIN_ARM64/${SHA_DARWIN_ARM64}/g" Formula/my-cli.rb
          sed -i "s/SHA_DARWIN_X64/${SHA_DARWIN_X64}/g" Formula/my-cli.rb
          sed -i "s/SHA_LINUX_ARM64/${SHA_LINUX_ARM64}/g" Formula/my-cli.rb
          sed -i "s/SHA_LINUX_X64/${SHA_LINUX_X64}/g" Formula/my-cli.rb

          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add Formula/my-cli.rb
          git commit -m "Update my-cli to ${VERSION}"
          git push

```

## npm Trusted Publishing (OIDC)

The `publish-npm` job uses npm's Trusted Publishing — no `NODE_AUTH_TOKEN` or npm secret needed. Authentication is handled via GitHub's OIDC token.

Requirements:
- `id-token: write` permission on the job
- npm >= 11.5.1 (Node.js >= 22.14.0)
- Trusted Publisher configured on npmjs.com (Settings → Trusted Publishers → add GitHub Actions)
- `repository` field in `package.json` matching the GitHub repo URL
- First publish must use a traditional token; OIDC works from the second publish onward

Provenance attestation (`--provenance`) is automatically generated with Trusted Publishing. The published package shows a "Built and signed on GitHub Actions" badge on npmjs.com.

For scoped packages (`@org/name`), `--access public` is needed on first publish only. Subsequent publishes retain the access level.

## Build Targets

Standard Rust cross-compilation targets for CLI distribution:

| Platform | Target | Runner | Notes |
|----------|--------|--------|-------|
| macOS Intel | x86_64-apple-darwin | macos-latest | |
| macOS ARM | aarch64-apple-darwin | macos-latest | |
| Linux x64 | x86_64-unknown-linux-gnu | ubuntu-latest | |
| Linux ARM | aarch64-unknown-linux-gnu | ubuntu-latest | Needs `gcc-aarch64-linux-gnu` |
| Windows x64 | x86_64-pc-windows-msvc | windows-latest | Packaged as .zip |

macOS builds (both Intel and ARM) run on `macos-latest`. Linux ARM cross-compiles on ubuntu with the aarch64 linker.

## Required GitHub Secrets

| Secret | Used by | Purpose |
|--------|---------|---------|
| `CARGO_REGISTRY_TOKEN` | publish-crates | crates.io API token |
| `HOMEBREW_TAP_TOKEN` | publish-homebrew | PAT with write access to the homebrew-tap repo |

npm uses OIDC Trusted Publishing — no secret needed. See "npm Trusted Publishing" section above.

## Triggering a Release

```bash
gh workflow run release.yml
gh run watch
```

The version bump is fully automated by oneup — no manual version editing needed.
