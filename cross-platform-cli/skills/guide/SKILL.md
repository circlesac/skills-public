---
name: cross-platform-cli
description: Guide for building and distributing compiled CLIs across npm, Homebrew, and GitHub Releases — use when scaffolding a new CLI project or setting up release pipelines
---

# Cross-Platform CLI Distribution

How to structure a compiled CLI so it can be installed via `npm install -g`, `brew install`, and direct download — all from the same repo, using GitHub Actions for automated releases.

Works with any language that produces standalone binaries: Rust, Go, Zig, Bun, etc.

Real example: [oneup](https://github.com/circlesac/oneup) (CalVer version management CLI, built with Rust).

## Artifact Naming Convention

Use simple `{name}-{os}-{arch}` names for release artifacts:

| Platform | Artifact name | Archive |
|----------|--------------|---------|
| macOS ARM | `my-cli-darwin-arm64` | `.tar.gz` |
| macOS Intel | `my-cli-darwin-amd64` | `.tar.gz` |
| Linux x64 | `my-cli-linux-amd64` | `.tar.gz` |
| Linux ARM | `my-cli-linux-arm64` | `.tar.gz` |
| Windows x64 | `my-cli-windows-amd64` | `.zip` |

The binary inside the archive is always named `my-cli` (or `my-cli.exe` on Windows). This convention is language-agnostic and matches what Go, Zig, and container tooling use.

## Project Structure

```
my-cli/
├── src/                    # Source code (language-specific)
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

Plus language-specific files at the root (e.g. `Cargo.toml`, `go.mod`, `build.zig`, `tsconfig.json`).

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

No `"version"` field — oneup writes it before `npm publish`. `files` controls what gets published to npm. Only `bin/` is included — source stays out.

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
  "darwin-x64":   { artifact: "my-cli-darwin-amd64",   ext: ".tar.gz" },
  "darwin-arm64":  { artifact: "my-cli-darwin-arm64",   ext: ".tar.gz" },
  "linux-x64":    { artifact: "my-cli-linux-amd64",    ext: ".tar.gz" },
  "linux-arm64":   { artifact: "my-cli-linux-arm64",    ext: ".tar.gz" },
  "win32-x64":    { artifact: "my-cli-windows-amd64",  ext: ".zip"    },
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

case "$ARCH" in
  x86_64)  ARCH="amd64" ;;
  aarch64) ARCH="arm64" ;;
esac

case "$OS-$ARCH" in
  darwin-arm64|darwin-amd64|linux-amd64|linux-arm64) ;;
  *) echo "Unsupported platform: $OS-$ARCH"; exit 1 ;;
esac

VERSION=$(curl -fsSL "https://api.github.com/repos/$REPO/releases/latest" | grep '"tag_name"' | cut -d'"' -f4)
URL="https://github.com/$REPO/releases/download/$VERSION/my-cli-$OS-$ARCH.tar.gz"

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
      url "https://github.com/org/my-cli/releases/download/v#{version}/my-cli-darwin-arm64.tar.gz"
      sha256 "SHA_DARWIN_ARM64"
    end
    on_intel do
      url "https://github.com/org/my-cli/releases/download/v#{version}/my-cli-darwin-amd64.tar.gz"
      sha256 "SHA_DARWIN_AMD64"
    end
  end

  on_linux do
    on_arm do
      url "https://github.com/org/my-cli/releases/download/v#{version}/my-cli-linux-arm64.tar.gz"
      sha256 "SHA_LINUX_ARM64"
    end
    on_intel do
      url "https://github.com/org/my-cli/releases/download/v#{version}/my-cli-linux-amd64.tar.gz"
      sha256 "SHA_LINUX_AMD64"
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
version → build (matrix) → github-release → publish-npm
                                           → publish-homebrew
                                           → publish-registry (language-specific, optional)
```

`softprops/action-gh-release` creates the tag via `tag_name`, so no separate tag job is needed.

### Full workflow

The build job uses a placeholder `BUILD STEPS GO HERE` — see [Language-Specific Build Steps](#language-specific-build-steps) for the actual build commands per language.

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
          VERSION=$(npx --yes @circlesac/oneup version --target package.json | tail -1)
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"

      - uses: actions/upload-artifact@v4
        with:
          name: versioned-files
          path: package.json   # Add other version files as needed (Cargo.toml, etc.)

  build:
    needs: version
    strategy:
      matrix:
        include:
          - os: macos-latest
            target_os: darwin
            target_arch: arm64
            name: my-cli-darwin-arm64
          - os: macos-latest
            target_os: darwin
            target_arch: amd64
            name: my-cli-darwin-amd64
          - os: ubuntu-latest
            target_os: linux
            target_arch: amd64
            name: my-cli-linux-amd64
          - os: ubuntu-latest
            target_os: linux
            target_arch: arm64
            name: my-cli-linux-arm64
          - os: windows-latest
            target_os: windows
            target_arch: amd64
            name: my-cli-windows-amd64

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/download-artifact@v4
        with:
          name: versioned-files

      # ── BUILD STEPS GO HERE ──
      # See "Language-Specific Build Steps" section.
      # Must produce a binary at: ./my-cli (or ./my-cli.exe on Windows)

      - name: Package (Unix)
        if: matrix.os != 'windows-latest'
        run: tar czf ${{ matrix.name }}.tar.gz my-cli

      - name: Package (Windows)
        if: matrix.os == 'windows-latest'
        run: 7z a ${{ matrix.name }}.zip my-cli.exe
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
          SHA_DARWIN_ARM64=$(shasum -a 256 artifacts/my-cli-darwin-arm64/*.tar.gz | cut -d' ' -f1)
          SHA_DARWIN_AMD64=$(shasum -a 256 artifacts/my-cli-darwin-amd64/*.tar.gz | cut -d' ' -f1)
          SHA_LINUX_ARM64=$(shasum -a 256 artifacts/my-cli-linux-arm64/*.tar.gz | cut -d' ' -f1)
          SHA_LINUX_AMD64=$(shasum -a 256 artifacts/my-cli-linux-amd64/*.tar.gz | cut -d' ' -f1)

          git clone https://x-access-token:${GH_TOKEN}@github.com/org/homebrew-tap.git
          cd homebrew-tap

          cp ../homebrew/my-cli.rb.template Formula/my-cli.rb
          sed -i "s/VERSION_PLACEHOLDER/${VERSION}/g" Formula/my-cli.rb
          sed -i "s/SHA_DARWIN_ARM64/${SHA_DARWIN_ARM64}/g" Formula/my-cli.rb
          sed -i "s/SHA_DARWIN_AMD64/${SHA_DARWIN_AMD64}/g" Formula/my-cli.rb
          sed -i "s/SHA_LINUX_ARM64/${SHA_LINUX_ARM64}/g" Formula/my-cli.rb
          sed -i "s/SHA_LINUX_AMD64/${SHA_LINUX_AMD64}/g" Formula/my-cli.rb

          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add Formula/my-cli.rb
          git commit -m "Update my-cli to ${VERSION}"
          git push
```

## Language-Specific Build Steps

Replace the `BUILD STEPS GO HERE` placeholder in the build job with one of these.

### Rust

Add `Cargo.toml` as a oneup `--target` and to the versioned-files artifact.

Build matrix needs Rust-specific target triples. Replace the matrix with:

```yaml
    strategy:
      matrix:
        include:
          - os: macos-latest
            rust_target: aarch64-apple-darwin
            name: my-cli-darwin-arm64
          - os: macos-latest
            rust_target: x86_64-apple-darwin
            name: my-cli-darwin-amd64
          - os: ubuntu-latest
            rust_target: x86_64-unknown-linux-gnu
            name: my-cli-linux-amd64
          - os: ubuntu-latest
            rust_target: aarch64-unknown-linux-gnu
            name: my-cli-linux-arm64
          - os: windows-latest
            rust_target: x86_64-pc-windows-msvc
            name: my-cli-windows-amd64
```

Build steps:

```yaml
      - name: Install cross-compilation tools
        if: matrix.rust_target == 'aarch64-unknown-linux-gnu'
        run: sudo apt-get install -y gcc-aarch64-linux-gnu

      - name: Add Rust target
        run: rustup target add ${{ matrix.rust_target }}

      - name: Build
        run: cargo build --release --target ${{ matrix.rust_target }}
        env:
          CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER: aarch64-linux-gnu-gcc

      - name: Copy binary
        run: cp target/${{ matrix.rust_target }}/release/my-cli${{ matrix.os == 'windows-latest' && '.exe' || '' }} .
        shell: bash
```

Optional: add a `publish-crates` job after `github-release`:

```yaml
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
```

### Go

```yaml
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod

      - name: Build
        run: go build -o my-cli${{ matrix.os == 'windows-latest' && '.exe' || '' }} .
        env:
          GOOS: ${{ matrix.target_os }}
          GOARCH: ${{ matrix.target_arch }}
          CGO_ENABLED: "0"
```

Go cross-compiles natively — no extra toolchains needed. All matrix entries run on any OS, but building on the target OS is simpler for CGO-dependent projects.

### Zig

```yaml
      - uses: mlugg/setup-zig@v2

      - name: Build
        run: |
          zig build -Doptimize=ReleaseSafe -Dtarget=${{ matrix.zig_target }}
          cp zig-out/bin/my-cli${{ matrix.os == 'windows-latest' && '.exe' || '' }} .
```

Add `zig_target` to the matrix (e.g. `aarch64-macos`, `x86_64-linux-gnu`, `x86_64-windows`). Zig cross-compiles natively like Go.

### Bun

```yaml
      - uses: oven-sh/setup-bun@v2

      - name: Build
        run: bun build --compile --target=bun-${{ matrix.target_os }}-${{ matrix.target_arch }} src/index.ts --outfile my-cli
```

`bun build --compile` produces standalone single-file executables. Cross-compilation is built in via `--target`. No extra toolchains needed.

Note: Bun's compile targets use the same `{os}-{arch}` convention: `bun-darwin-arm64`, `bun-linux-x64`, `bun-windows-x64`. Map `amd64` to `x64` in the matrix if needed:

```yaml
        include:
          - os: macos-latest
            target_os: darwin
            target_arch: arm64
            bun_target: bun-darwin-arm64
            name: my-cli-darwin-arm64
          - os: macos-latest
            target_os: darwin
            target_arch: amd64
            bun_target: bun-darwin-x64
            name: my-cli-darwin-amd64
          # ... etc
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

Standard targets for CLI distribution:

| Platform | OS | Arch | Runner | Notes |
|----------|-----|------|--------|-------|
| macOS ARM | darwin | arm64 | macos-latest | Apple Silicon |
| macOS Intel | darwin | amd64 | macos-latest | |
| Linux x64 | linux | amd64 | ubuntu-latest | |
| Linux ARM | linux | arm64 | ubuntu-latest | May need cross-compilation toolchain |
| Windows x64 | windows | amd64 | windows-latest | Packaged as .zip |

macOS builds (both Intel and ARM) run on `macos-latest`. Linux ARM may require cross-compilation depending on the language (Rust needs `gcc-aarch64-linux-gnu`; Go and Zig cross-compile natively).

## Required GitHub Secrets

| Secret | Used by | Purpose |
|--------|---------|---------|
| `HOMEBREW_TAP_TOKEN` | publish-homebrew | PAT with write access to the homebrew-tap repo |

npm uses OIDC Trusted Publishing — no secret needed. See "npm Trusted Publishing" section above.

Language-specific registries may need additional secrets (e.g. `CARGO_REGISTRY_TOKEN` for crates.io).

## Triggering a Release

```bash
gh workflow run release.yml
gh run watch
```

The version bump is fully automated by oneup — no manual version editing needed.
