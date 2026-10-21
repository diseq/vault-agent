# vault-agent

Public Linux and Windows x86-64 downloads for vault-agent. Builds are pushed by
the private source repository; this repository does not need access to it.

## Install With mise

For normal archive releases, add:

```toml
[tools."github:diseq/vault-agent"]
version = "latest"
github_attestations = true

[tools."github:diseq/vault-agent".platforms]
linux-x64 = { asset_pattern = "vault-agent-linux-amd64.tar.gz" }
windows-x64 = { asset_pattern = "vault-agent-windows-amd64.zip" }
```

Run `mise install`. Manual tag/CI releases contain raw binaries instead; for
those versions, use `vault-agent` and `vault-agent.exe` as the respective asset
patterns.

mise verifies GitHub attestations against `diseq/vault-agent`, not the private
source repository. Verification is opportunistic without recorded provenance:
`github_attestations = true` alone does not require an attestation to exist.
After a verified install records `github-attestations` provenance in
`mise.lock`, retain that lockfile and use locked installs to enforce it. Do not
disable verification or replace the lockfile to bypass verification failures.

To verify an archive directly with the GitHub CLI:

```sh
gh attestation verify vault-agent-linux-amd64.tar.gz \
	--repo diseq/vault-agent \
	--signer-workflow diseq/vault-agent/.github/workflows/publish.yaml \
	--deny-self-hosted-runners
```

The attestations prove that this repository's workflow distributed the exact
downloaded bytes after checking the private publisher's supplied digests. They
do not independently verify the private build or its claimed source commit.
`SOURCE.json` records that distinction and the publisher-reported identity.

## Publishing

The private repository's GitHub App requires **Contents: write** on this
repository. It uploads both platform files into a draft, then sends a
`publish-release` repository dispatch with `tag`, `source_sha`, `asset_kind`,
`linux_sha256`, and `windows_sha256`. No private-source token or other secret
is required here.

The **Verify and Publish Distribution** workflow runs only from `main`, with
Contents, Attestations, and OIDC-token write permissions. It downloads files
only from this repository's draft, rejects unexpected files and digest
mismatches, attests the artifacts and metadata, and publishes only after both
platforms are ready. Attestation failure leaves the draft unpublished. Source
build files are not unpacked, executed, or repackaged.

Deploy this workflow to `main` before enabling the source repository's handoff.
Protect `main` and restrict repository write access: the authenticated dispatch
is the trust boundary, not the checksum file uploaded alongside the binaries.

For recovery, rerun the failed distribution workflow with its original event,
or use **Run workflow** on `main` with the original tag, source commit, asset
kind (`archives` or `binaries`), and both SHA-256 digests obtained from the
private publisher. Do not derive recovery digests from untrusted draft files.
Published releases are never overwritten by these workflows; changes require a
new version. Previously published releases are not retroactively attested.
