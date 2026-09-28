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

## Source only

The fork holds source and tags only. It has no CI (upstream's `.github/workflows/` is removed
on these branches) and publishes no release artifacts or images.

Consumers pin a tag and the commit SHA it points at. They fetch the tag, check that it resolves
to the pinned commit, and build the plugin with Go 1.26.8 to match the EKS build, then record
the output's sha256:

```sh
git fetch origin tag <tag>
test "$(git rev-parse '<tag>^{commit}')" = "<pinned commit SHA>"
git checkout --detach <pinned commit SHA>
GOTOOLCHAIN=go1.26.8 go version   # record the toolchain used
GOTOOLCHAIN=go1.26.8 CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -trimpath -buildvcs=false -buildmode=pie -ldflags "-s -w" \
  -o aws-cni ./cmd/routed-eni-cni-plugin
sha256sum aws-cni           # record this
```

`-trimpath` drops local paths and `-buildvcs=false` turns off VCS stamping, so the binary
depends only on the source and the toolchain. Building by tag or by commit, from any checkout
location, gives the same sha256, and anyone can rebuild it to check the recorded value.

Delivery is moving to a container image (an installer DaemonSet) built from this tag.

> **Warning: do not set `-X main.version`, or any `-X` flag, on the plugin.** The plugin sends
> `main.version` to ipamd as `ClientVersion` on every Add/Del
> (`cmd/routed-eni-cni-plugin/cni.go`). ipamd rejects any request whose `ClientVersion` differs
> from its own version (`validateVersion` in `pkg/ipamd/rpc_handler.go`). The EKS-managed ipamd
> we run beside this plugin reports an empty version. A stamped plugin therefore fails every
> pod's network setup on the node.

## Branches and tags

- **Branches** are `scale/vX.Y.Z`, one per upstream base, cut from upstream tag `vX.Y.Z`. Each
  one carries exactly one functional patch (the derived-MAC fix) plus fork-housekeeping commits
  (this file, removing upstream CI). It must not diverge beyond that. Any other change goes
  upstream, not here. To move to a new upstream release, cut a new branch from that tag and
  reapply the patch; don't rebase or merge an existing branch.
- **Tags** are `vX.Y.Z-scale.N`: the upstream base plus a Scale revision counter starting at 1.
  Tags are never moved or reused.

## When to drop it

Drop the fork branch, and move back to the stock plugin, once upstream fixes #3576 in a
release.
