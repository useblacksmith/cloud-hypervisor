# Claude Code Context

This is a fork of [cloud-hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor)
with Blacksmith-specific patches, used to build the `cloud-hypervisor` binary that runs
Windows VMs on our hosts.

## Branch / remote structure

- `upstream` (a.k.a. `cloud-hypervisor/cloud-hypervisor`): the official project.
- `origin` = `useblacksmith/cloud-hypervisor`: this fork.
- **`main`**: a *pure fast-forward mirror* of upstream `main`. It carries **no** custom
  commits, so it can always be synced with GitHub's "Sync fork" / `merge-upstream`.
  **Do not commit to `main`** — doing so breaks the clean fast-forward sync.
- **`patchset`**: our Blacksmith patches as a clean, linear series **on top of an upstream
  release tag** (currently `v52.0`). This is the branch we build and release from.

## Golden rules for `patchset`

1. **`patchset` is based on a release *tag* (e.g. `v52.0`), never on `main`.** `main` is
   unreleased, moving, and untested as a base. Release tags are stable and give the binary a
   clean version identity (`cloud-hypervisor --version` → `v52.0`), which both our ansible and
   the agent's version-gating depend on.
2. **Never `git merge`, "Update branch", or rebase `patchset` onto `main`.** That pulls in
   hundreds of unreleased commits, adds merge commits, and destroys the linear series. If you
   see a merge commit on `patchset`, undo it (`git reset --hard` to the last patch + force-push).
3. **`patchset` is build-from-only — it is never merged into `main`.** Don't open a
   `patchset → main` PR (the tempting "Update branch" button is how rule 2 gets violated).
4. To pick up a newer cloud-hypervisor, **rebase onto the next release tag** (see below), or
   cherry-pick a *specific* needed upstream commit — not a wholesale move to `main`.

## The patches (current series on top of `v52.0`)

Both make Windows guest userspace logs reach the host `vm.log` over COM1 (→ Vector → Axiom),
matching the Linux `console=ttyS0` path:

1. `vmm: carve legacy COM1 I/O range out of PCI Root Complex _CRS` (`vmm/src/pci_segment.rs`)
   — frees COM1's `0x3f8-0x3ff` from the PCI root I/O window so Windows stops rejecting COM1
   with Code 12 (`STATUS_CONFLICTING_ADDRESSES`) and `serial.sys` binds.
2. `devices: serial: assert THRE when the TX interrupt is enabled` (`devices/src/legacy/serial.rs`)
   — asserts THRE on the IER-enable path so interrupt-driven TX (Windows `serial.sys`) doesn't
   deadlock. This is the key fix that actually gets bytes onto the wire.

(There is also unmerged nested-virtualization work — FEATURE_CONTROL / Hyper-V CPUID in
`arch/src/x86_64/mod.rs` — that is **not** in this series; it's blocked on guest Event 27.)

## Cutting a release

The build/publish workflow is upstream's own `.github/workflows/release.yaml`, inherited from
the `v52.0` base — **no custom workflow is needed**. It triggers on **tag creation**, builds the
`x86_64-unknown-linux-musl` static binary (`cloud-hypervisor-static`), and creates a **draft**
GitHub Release with it attached.

```bash
# 0. Be on a clean patchset (verify it's just our patches on the tag):
git checkout patchset
git log --oneline v52.0..patchset      # should show only our patches (+ this doc)

# 1. Tag the patchset HEAD. Scheme: patchset-v<CH_VERSION>-<UTC timestamp>
TAG="patchset-v52.0-$(date -u +%Y%m%d%H%M%S)"
git tag -a "$TAG" patchset -m "Blacksmith CH patchset on v52.0"
git push origin "$TAG"

# 2. Wait for the "Cloud Hypervisor Release" workflow to finish, then verify the asset:
gh run list  --repo useblacksmith/cloud-hypervisor --limit 3
gh release view "$TAG" --repo useblacksmith/cloud-hypervisor \
  --json isDraft,assets --jq '.isDraft, (.assets[].name)'   # expect cloud-hypervisor-static
```

The release is created as a **draft on purpose** — that is the manual "publish gate". Ansible's
download URL only resolves once the release is **published**, so:

```bash
gh release edit "$TAG" --repo useblacksmith/cloud-hypervisor --draft=false
```

## How the binary is consumed

The `fa` repo's ansible pulls the binary as a GitHub Release asset:
`fa/infra/ansible/linux_bootstrap/setup_cloudhypervisor_environment.yaml`

```yaml
ch_version: "patchset-v52.0-<timestamp>"
ch_binary_url: "https://github.com/useblacksmith/cloud-hypervisor/releases/download/{{ ch_version }}/cloud-hypervisor-static"
```

**Coupling to the agent:** the binary still reports `v52.0`, and the agent gates its CH command
args on the detected version (`fa/agent/vm/cloudhypervisor/manager.go`, `detectCHCapabilities`):
`nested=on` needs CH ≥ v50, `image_type=qcow2,backing_files=on` needs CH ≥ v51. Keep this in
mind if the patchset ever moves to a different base version.

## Forward-porting to a new cloud-hypervisor release

When a new upstream release (e.g. `v53.0`) ships and we want it:

```bash
git fetch upstream --tags
git branch patchset-backup-$(date -u +%Y%m%d)      # safety
# Rebase our patches from the old base tag onto the new one:
git rebase --onto v53.0 v52.0 patchset
# Resolve conflicts (our two files rarely change upstream), then:
git push origin patchset --force-with-lease
```

Then update the `v52.0` references in this file and in the ansible to the new version, cut a new
`patchset-v53.0-<timestamp>` release, and re-verify the agent version-gating thresholds still hold.
