# Scale fork of amazon-vpc-cni-k8s

This is `scaleapi/amazon-vpc-cni-k8s`, a temporary fork of
[aws/amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s). It is tracked in Linear
**MLI-9309**.

## Why it exists

Under a burst of pod starts on a fresh node, the CNI plugin fails pod network setup with
`failed to generate Unique MAC addr for host side veth: netlink operation interruption persisted
after 5 attempts` (upstream issue
[aws/amazon-vpc-cni-k8s#3576](https://github.com/aws/amazon-vpc-cni-k8s/issues/3576)). The
plugin picks a random host-veth MAC and checks it against a dump of every host link. Concurrent
veth churn keeps interrupting that dump until it gives up.

The patch on this branch derives the MAC from the sandbox identity instead
(`sha256(hostVethName + "\x00" + netnsPath)`, with the locally administered bit set and the
unicast bit kept), so the dump is no longer needed. On a 500-pod burst onto one fresh
c6a.48xlarge in staging, veth-MAC failures went from 758 to 0, CNI ADDs from 1,258 to 500, and
pod start p95 from 167 s to 93 s.

## Rules

- **Exactly one patch.** Each `scale/vX.Y.Z` branch is upstream tag `vX.Y.Z` plus the
  derived-MAC patch, this file, and the release workflow. It must not diverge beyond that. Any
  other change goes upstream, not here.
- **One branch per upstream base.** To move to a new upstream release, cut a new
  `scale/vX.Y.Z` branch from that tag and reapply the patch. Don't rebase or merge an existing
  branch.
- **Tags** are `vX.Y.Z-scale.N`: the upstream base plus a Scale revision counter starting at 1.
- **The only published artifact is the `aws-cni` plugin binary** (linux amd64 and arm64, plus
  `SHA256SUMS`), attached to the GitHub release for the tag by
  `.github/workflows/scale-release.yml`. It is overlaid on nodes that keep the EKS-managed
  VPC CNI addon. No container images are built or pushed from this fork.

## When to delete it

Drop this fork once upstream fixes #3576 in a release, and move back to the stock plugin.
