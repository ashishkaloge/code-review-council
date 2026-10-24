# Releasing on npm

The `Prepare npm release` workflow validates and packs the exact commit selected
by a manual run on `main`. Its default is a dry run. Pushes, tags, pull requests
and GitHub releases do not publish a package.

The workflow installs no project dependencies, runs the test suite, checks the
CLI, audits the package file list and smoke-tests the packed archive. Only the
separate staging job can obtain an npm OIDC credential. It stages the same
archive with provenance; a maintainer must review and approve it on npm with
2FA before the version becomes public. No npm token belongs in GitHub secrets.

## Configure the publisher

In the npm package settings for `@ashishkaloge/code-review-council`, add this
trusted publisher:

| Field | Value |
| --- | --- |
| Provider | GitHub Actions |
| Organization or user | `ashishkaloge` |
| Repository | `code-review-council` |
| Workflow filename | `publish.yml` |
| Environment | Leave blank |
| Allowed actions | Stage packages only; disable direct publishing and dist-tag changes |

Create the npm connection when the next release is ready: a new connection
expires if it does not complete its first successful publish within two days.
If setup was done earlier, check its status and replace an expired connection
before staging. The workflow filename must match this value. Keep the existing
GitHub protection for `main`; the workflow itself rejects other branches.

After the first OIDC release has been verified, select **Require two-factor
authentication and disallow tokens** in the npm package's publishing access
settings. Verify the change and remove any obsolete automation tokens. OIDC
continues to work with that restriction; it does not require a stored token.

## Prepare and approve a release

1. Merge a reviewed version change into `main`. Keep `package.json` and any
   release documentation consistent. This workflow supports stable versions
   such as `0.1.1`; prereleases require a separately reviewed dist-tag policy.
2. Check that CI passed for that commit. Open **Actions → Prepare npm release →
   Run workflow**, select `main`, enter its package version and leave `dry_run`
   checked. The existing version can be used for a setup dry run.
3. Review the run's source commit, checks, archive SHA-256 and downloadable
   `npm-release-<commit>` artifact. No npm credential or stage is created by a
   dry run. Artifacts expire after seven days.
4. When the release is authorized, run the same workflow on the same `main`
   commit and version with `dry_run` unchecked. If `main` has moved, review and
   validate the new commit first. Staging rejects an already published version;
   npm also rejects a duplicate pending staged version.
5. In npm's **Staged Packages** tab, inspect the candidate, compare its archive
   SHA-256 with the workflow summary and verify its source/provenance. Approve
   the stage with 2FA only after review. Staging alone is not a release.
6. Verify the published version, `latest` tag and provenance link. Install the
   exact version in a fresh temporary directory and check `code-review-council
   --help`. This smoke test does not exercise a live agent or establish review
   quality.

The workflow cannot approve stages, change dist-tags, create Git tags or create
GitHub releases. Publish each release under a new version; never replace an
already published version.

References: [npm trusted publishers](https://docs.npmjs.com/trusted-publishers/),
[staged publishing](https://docs.npmjs.com/staged-publishing/) and
[`npm stage`](https://docs.npmjs.com/cli/v11/commands/npm-stage/).
