# Proposal: Position-anchored tags for GitHub Actions

## Summary

Append the first-parent commit count and short hash to every action version tag, giving humans a cheap smoke test and machines a verifiable anchor:

```
v0.18.0-r75-a1b2c3d
```

## Motivation

The [Trivy supply chain attack](https://snyk.io/articles/trivy-github-actions-supply-chain-compromise/) (March 19, 2026) demonstrated a fundamental weakness in how GitHub Actions are referenced. An attacker force-pushed 75 of 76 version tags in `aquasecurity/trivy-action`, silently repointing them to malicious commits. Every CI pipeline referencing those tags by name pulled the compromised code.

The root cause is simple: **tags are mutable refs, but users treat them as stable identifiers.** Git's actual immutability guarantee lives in the object store (commit hashes), not in the naming layer. The standard convention `uses: org/action@v1.2.3` relies entirely on the mutable layer.

Pinning to full SHAs (`@a1b2c3d4e5f6...`) solves this cryptographically but fails the human layer — nobody reads, remembers, or compares hex strings during review.

## Proposal

Extend the tagging convention to include two pieces of information derived from the commit DAG:

```
v{SEMVER}-r{POSITION}-{SHORT_HASH}
```

| Component | Example | Purpose |
|---|---|---|
| Semantic version | `v0.18.0` | Human-readable release identity |
| Commit position | `r75` | First-parent commit count from root — slow-moving, easy to remember |
| Short hash | `a1b2c3d` | Content-addressed anchor for machine verification |

### Generation

```bash
VERSION="0.18.0"
POS=$(git rev-list --first-parent --count HEAD)
HASH=$(git rev-parse --short HEAD)
TAG="v${VERSION}-r${POS}-${HASH}"

git tag "$TAG"
```

### Verification

```bash
# Extract expected position and hash from tag name
TAG="v0.18.0-r75-a1b2c3d"
EXPECTED_POS=$(echo "$TAG" | sed 's/.*-r\([0-9]*\)-.*/\1/')
EXPECTED_HASH=$(echo "$TAG" | sed 's/.*-\([a-f0-9]*\)$/\1/')

# Verify against actual DAG
ACTUAL_POS=$(git rev-list --first-parent --count "$TAG")
ACTUAL_HASH=$(git rev-parse --short "$TAG")

if [ "$EXPECTED_POS" != "$ACTUAL_POS" ] || [ "$ACTUAL_HASH" != "$EXPECTED_HASH" ]; then
  echo "TAG MISMATCH — possible tampering"
fi
```

## Why this works

The proposal exploits an asymmetry in human cognition:

- **Humans are bad at:** comparing hex strings (`a1b2c3d` vs `b3f8e2a`)
- **Humans are good at:** noticing a number that's "off" (75 → 3)

The commit position is a slow-moving, monotonically increasing integer. If a tag that said `r75` last week suddenly says `r3`, a reviewer will notice immediately — even without tooling. The short hash then provides cryptographic backing for machine verification.

In the Trivy attack, every rewritten tag pointed to a commit whose tree was just `master` HEAD. The position numbers would have been wildly inconsistent with what anyone familiar with the project expected.

## Properties

- **Deterministic** — derived from the DAG, not assigned manually
- **Human-scannable** — a changing position number is instantly suspicious
- **Machine-verifiable** — tooling can validate position + hash against the actual DAG
- **Backward-compatible** — extends existing semver tags, doesn't replace them
- **Zero infrastructure** — no signing keys, transparency logs, or external services required

## Limitations

- The commit position depends on the first-parent chain, which can differ after certain rebase/merge strategies. Projects should document which counting method they use.
- Short hashes can collide in very large repositories. The position number mitigates this since both must match.
- This is a **detection** mechanism, not a **prevention** mechanism. It makes tampering visible, not impossible.
- Adoption requires convention change. Tooling (e.g., a GitHub Action or Dependabot integration) that auto-verifies position-anchored tags would accelerate this.

## Suggested next steps

1. Standardize the tag format `v{SEMVER}-r{POS}-{HASH}` as a recommended convention for GitHub Actions.
2. Add optional verification in the Actions runner: if a tag matches the position-anchored pattern, check position and hash against the DAG before execution.
3. Surface the position number in the GitHub UI alongside tags to make visual review trivial.

## Context

This proposal was motivated by the Trivy compromise, but the underlying problem — mutable refs masquerading as stable identifiers — applies broadly to any ecosystem that resolves dependencies by name rather than by content hash.

## Getting involved

- **Have feedback?** Open an [issue](../../issues) to discuss.
- **Want to implement this?** See [IMPLEMENTATION.md](./IMPLEMENTATION.md) for reference tooling.
- **Using this convention?** Add your project to [ADOPTION.md](./ADOPTION.md).