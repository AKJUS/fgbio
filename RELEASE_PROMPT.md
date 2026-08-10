# fgbio Release Guide

## Important Notes for AI-Assisted Releases

1. **No AI references in commits**: Commit messages must NOT include any references to Claude, AI assistants, or "Generated with" footers. Use clean, standard commit messages.

2. **User confirmation required**: Before executing ANY action (edits, git commands, etc.), prompt the user for explicit confirmation to proceed. Do not auto-execute release steps.

---

## Quick Reference

| Field | Current Value |
|-------|---------------|
| **Next Release Version** | 4.1.2 |
| **Last Release** | 4.1.1 (August 10, 2026) |
| **Version File** | `version.sbt` |

---

## Version Bump Convention

| Bump Type | When to Use | Example |
|-----------|-------------|---------|
| **Major (X.0.0)** | Breaking changes, API changes, removed features | 2.5.21 → 3.0.0 |
| **Minor (X.Y.0)** | New features, significant functionality | 3.0.0 → 3.1.0 |
| **Patch (X.Y.Z)** | Bug fixes, small improvements | 3.1.0 → 3.1.1 |

---

## Prerequisites Checklist

### GitHub Secrets (Required)

These must be configured in `Settings > Secrets and variables > Actions`:

| Secret | Description |
|--------|-------------|
| `SONATYPE_USER` | Sonatype Central username |
| `SONATYPE_PASS` | Sonatype Central password/token |
| `PGP_PASSPHRASE` | GPG key passphrase |
| `PGP_SECRET` | Base64-encoded GPG private key |

### Pre-Release Verification

- [ ] GPG key has not expired
- [ ] Push access to `main` branch
- [ ] Permission to create and push tags
- [ ] Tests pass locally (optional): `sbt clean test`

---

## Release Steps

### Step 1: Switch to Release Version

Edit `version.sbt`:

```scala
// Change FROM (snapshot mode):
ThisBuild / version := s"${_version}-${gitHeadCommitSha.value}-SNAPSHOT"

// Change TO (release mode):
ThisBuild / version := _version
```

### Step 2: Commit and Tag

```bash
git commit -a -m "chore: release X.Y.Z"
git tag X.Y.Z
```

### Step 3: Push to Trigger Release Workflow

```bash
git push -f origin main && git push origin X.Y.Z
```

**Note**: The `-f` flag is needed to bypass branch protection rules for direct pushes to main.

The GitHub Actions workflow will automatically:
1. Verify tag is on main branch
2. Run full test suite (Java 8, 11, 17, 21, 22)
3. Generate tool and metrics documentation
4. Build assembly JAR
5. Sign and publish to Sonatype (`sbt +publishSigned`)
6. Release to Maven Central (`sbt sonaRelease`)
7. Publish docs to gh-pages branch
8. Generate changelog using git-cliff
9. Create GitHub release with JAR artifact

### Step 4: Bump to Next Snapshot Version

After workflow succeeds, edit `version.sbt`:

```scala
val _version = "X.Y.{Z+1}"  // Increment version
ThisBuild / version := s"${_version}-${gitHeadCommitSha.value}-SNAPSHOT"
```

### Step 5: Commit and Push Snapshot

```bash
git commit -a -m "chore: bump to X.Y.{Z+1}-SNAPSHOT"
git push -f origin main
```

### Step 6: Post-Release Verification

1. **GitHub Release**: https://github.com/fulcrumgenomics/fgbio/releases
   - Review auto-generated changelog
   - Add any additional context
   - Verify JAR artifact is attached

2. **Maven Central**: https://central.sonatype.com (may take ~30 minutes)
   - Search for `com.fulcrumgenomics:fgbio_2.13:X.Y.Z`

3. **Bioconda**: Auto-detects version changes (monitor bioconda-recipes repo)

### Step 7: Update This Release Guide

Update `RELEASE_PROMPT.md` with:
- New "Next Release Version" (increment from just-released version)
- New "Last Release" entry with version and date
- Add entry to Release History table

---

## Troubleshooting

### Release Workflow Fails

1. Check GitHub Actions logs for specific failure
2. Common issues:
   - **GPG key expired**: Generate new key, update `PGP_SECRET` and `PGP_PASSPHRASE`
   - **Sonatype auth failed**: Regenerate token, update `SONATYPE_USER`/`SONATYPE_PASS`
   - **Test failures**: Fix tests before re-releasing

### Re-releasing (if needed)

```bash
# Delete tag locally and remotely
git tag -d X.Y.Z
git push origin :refs/tags/X.Y.Z

# Fix the issue, then re-tag and push
git tag X.Y.Z
git push -f origin main && git push origin X.Y.Z
```

---

## Key Files

| File | Purpose |
|------|---------|
| `version.sbt` | Version definition and snapshot/release toggle |
| `build.sbt` | Build config, publishing settings, Sonatype config |
| `project/plugins.sbt` | sbt plugins (sbt-pgp, sbt-release, etc.) |
| `.github/workflows/release.yaml` | Release CI/CD workflow |
| `Contributing.md` | Original release documentation |

---

## Release History

| Version | Date | Key Changes |
|---------|------|-------------|
| 4.1.1 | 2026-08-10 | Deterministic consensus read downsampling, fork-PR docs job fix |
| 4.1.0 | 2026-05-20 | htsjdk 5.0.0, libdeflate, non-terminal `+` in read structures, CODEC dovetail fix |
| 4.0.0 | 2026-03-30 | Java 17 minimum, htsjdk 4.2.0, Scala 2.13.16 |
| 3.1.2 | 2026-03-06 | Cell barcode sort support, clip overlapping reads fix, consensus stats fix |
| 3.1.1 | 2025-12-15 | CRAM support, consensus caller fixes, UMI assigner determinism |
| 3.1.0 | 2025-11-20 | Cell barcode support, streaming input, CorrectUmis speedup |
| 3.0.0 | - | Breaking changes: renamed options, removed deprecated code |
| 2.5.21 | - | Last 2.5.x release |

---

## Learnings & Notes

- **Force push required**: Branch protection on `main` requires `-f` flag for direct pushes during releases. The push output still prints `Changes must be made through a pull request` — that is the rejection being overridden, not a failure; confirm the ref actually moved.
- **Step-by-step confirmation**: When using AI assistance, confirm each action before execution to maintain control
- **Version already set**: The `_version` in `version.sbt` is typically pre-set to the next release version after each release
- **Don't skip Steps 4–7**: The 4.1.0 release skipped them, leaving `version.sbt` in *release* mode on `main` for ~3 months (so main built as a plain `4.1.0`, colliding with the published artifact) and this guide stale by two releases. Verify `version.sbt` is back in snapshot mode before calling a release done.
- **Where the secrets actually live**: `SONATYPE_USER`/`SONATYPE_PASS` are on the repo's `github-actions` **environment**; `PGP_SECRET`/`PGP_PASSPHRASE` are **org-level** (`fulcrumgenomics`, scoped to selected repos). Precedence is environment > repo > org, so the environment copies of the Sonatype pair are what the job receives. List them with `gh secret list --env github-actions` and `gh secret list --org fulcrumgenomics`.
- **Checking GPG expiry without the key locally**: the signing key is not in any local keyring, so recover its ID from a published signature and look it up on a keyserver:
  ```bash
  curl -sO https://repo1.maven.org/maven2/com/fulcrumgenomics/fgbio_2.13/<last-version>/fgbio_2.13-<last-version>.pom.asc
  gpg --list-packets fgbio_2.13-<last-version>.pom.asc     # → keyid
  curl -sS "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x<fingerprint>" | gpg --show-keys
  ```
  Current key: `74317092BC7A3C35` (ed25519, Nils Homer), **expires 2028-03-18**. It is published on `keyserver.ubuntu.com` but *not* `keys.openpgp.org` — check that first if Sonatype ever rejects a signature.
- **Timing**: 4.1.1 took ~15 min for the whole workflow (test matrix ~3.5 min per JDK, then publish). Maven Central sync was near-immediate, well under the ~30 min the guide suggests.
