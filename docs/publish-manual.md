# Manual publish under the @dkhokhlov scope

The unscoped package `open-codebase-index` is owned by upstream (`helweg`).
The CI workflow publishes that package on GitHub release. Do not publish it
manually.

The `@dkhokhlov/open-codebase-index` package is a fork package. It is published
manually from a Linux x64 host. The published tarball ships only the Linux x64
native binary. The package is published as `latest`. It provides two binaries:
`open-codebase-index-mcp` and `opencode-codebase-index-mcp`.

The scope rename is a temporary edit in the throwaway staging directory. The
repository, the identity catalog, and the committed `package.json` stay
`open-codebase-index`. Do not commit the scoped name.

## Prerequisites

Run on a Linux x64 host with Node.js 20 or newer and an npm login that can
publish to the `@dkhokhlov` scope. Confirm the login:

```bash
npm whoami
```

The answer must be `dkhokhlov`.

## Steps

### 1. Bump the version

Edit `package.json` and `package-lock.json`. Change the root `version` and the
`packages[""].version` from the previous release to the new release. Example:
`0.23.2` to `0.23.3`. Do not change `native/Cargo.lock`; the `0.23.x` string
there belongs to an unrelated transitive crate.

Commit the bump on the release branch:

```bash
git add package.json package-lock.json
git commit -m "chore(release): prepare v0.23.3"
```

### 2. Build

Build the TypeScript bundle and the host native binary:

```bash
npm run build:ts
npm run build:native
```

`build:native` builds only the host platform. On a Linux x64 host it produces
`native/codebase-index-native.linux-x64-gnu.node`. This is the only native
binary in the published tarball.

### 3. Stage the future identity

Stage the package with the future identity. The staging script rewrites the
package name to `open-codebase-index`, sets both binaries, and patches the host
manifests. It validates the repository URL against the identity catalog. Use
the Helweg repository URL:

```bash
node scripts/prepare-package-metadata.mjs \
  --package-name open-codebase-index \
  --project-root . \
  --output-dir /tmp/staging-dkhokhlov \
  --repository-url https://github.com/Helweg/open-codebase-index
```

### 4. Apply the @dkhokhlov scope as a temporary edit

The staging script accepts only catalog names (`open-codebase-index` or
`opencode-codebase-index`). Apply the scoped name as a temporary edit in the
staging directory only. Do not change the repository.

In `/tmp/staging-dkhokhlov/package.json`, change `name` from
`open-codebase-index` to `@dkhokhlov/open-codebase-index`.

In `/tmp/staging-dkhokhlov/package-lock.json`, change the root `name` and the
`packages[""].name` from `open-codebase-index` to
`@dkhokhlov/open-codebase-index`.

### 5. Publish

Publish from the staging directory as `latest`:

```bash
cd /tmp/staging-dkhokhlov
npm publish --ignore-scripts --access public
```

`--ignore-scripts` is mandatory. The staging directory has no dev dependencies,
so `prepublishOnly` must not run. `--access public` is required for scoped
packages. Do not pass `--tag`; the package publishes as `latest`. Do not pass
`--provenance`; provenance is CI only.

### 6. Verify

Confirm the new version is published as `latest`:

```bash
npm view @dkhokhlov/open-codebase-index@0.23.3 name version bin repository
npm view @dkhokhlov/open-codebase-index dist-tags
```

The `dist-tags` must show `latest: 0.23.3`. Confirm the tarball is Linux only:

```bash
npm pack @dkhokhlov/open-codebase-index@0.23.3 --dry-run
```

The tarball must contain `native/codebase-index-native.linux-x64-gnu.node`,
`dist/cli.js`, and both binaries. It must not contain native binaries for other
platforms.

### 7. Discard the staging directory

Remove the throwaway staging directory:

```bash
rm -rf /tmp/staging-dkhokhlov
```

The repository stays on `open-codebase-index`. The scoped name was a temporary
edit.

## Notes

- The `@dkhokhlov/open-codebase-index` package is Linux only by design. The
  cross-platform package is the unscoped `open-codebase-index`, published by CI.
- A version is consumed once it is published. Do not republish a version.
- If `npm publish` returns a `403` error, stop. Report the error before you
  retry.