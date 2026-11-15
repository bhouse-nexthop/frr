# FRR for Ubuntu noble, with EVPN-MH remote MACs installed as static

This branch builds Ubuntu 24.04 (noble)'s `frr` source package with extra patches.
Everything else is Ubuntu's package, unchanged.

- `UBUNTU_VERSION`: the Ubuntu source version to build, fetched from Launchpad.
- `LOCAL_SUFFIX`: appended to that version (`8.4.4-1.1ubuntu6.8` + `+nh1`).
- `patches/`: DEP-3 patches appended to Ubuntu's `debian/patches/series`.
- `build.sh`: the build, run in an `ubuntu:24.04` container.
- `.github/workflows/build.yml`: builds on every push to `noble-nexthop` (packages as a workflow
  artifact) and publishes a GitHub Release for tags named `noble/<version>`.

## The problem

Kernels carrying the fix for CVE-2026-98023 ("vxlan: reject dynamic fdb entries that
reference a nexthop id") refuse a dynamic fdb entry that uses an nhid. That includes
Ubuntu noble's HWE 7.0 kernel (7.0.0-42 checked). zebra installs every remote MAC behind
an EVPN multihoming Ethernet Segment that way (`NUD_REACHABLE` + `NTF_EXT_LEARNED` +
nhid), so the installs fail:

    zebra: Extended Error: A nexthop fdb cannot be dynamic
    zebra: netlink-dp (NS 0) error: Invalid argument, type=RTM_NEWNEIGH(28), ...

Without those entries, unicast to MACs behind the ES falls through to head-end
replication. It goes to every VTEP of the ES and can arrive duplicated.

## The patch

`zebra-evpn-mh-remote-mac-nhid-static.patch` sets `NUD_NOARP` (static) on remote MACs
that reference a nexthop group. zebra owns those entries, and kernel readback of vxlan
fdb entries ignores `NUD_NOARP`, so only the installed state changes. It's an interim
local patch, to be replaced by the upstream FRR fix (in progress on the
[`bhouse.zebra-fdb-nhid-noarp`](https://github.com/bhouse-nexthop/frr/tree/bhouse.zebra-fdb-nhid-noarp)
branch of this fork) once it's published.

## Rebasing on a newer Ubuntu update

Set `UBUNTU_VERSION` to the new version, push, and check that the patches still apply.
If Ubuntu's version moves past ours, bump `LOCAL_SUFFIX` or hold/pin `frr` on hosts that
run this build until Ubuntu ships the fix.
