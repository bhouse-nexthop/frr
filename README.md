# FRR for Ubuntu noble, with zebra accepting lower EVPN sequence numbers from BGP

This branch builds Ubuntu 24.04 (noble)'s `frr` source package with extra patches.
Everything else is Ubuntu's package, unchanged.

- `UBUNTU_VERSION`: the Ubuntu source version to build, fetched from Launchpad.
- `LOCAL_SUFFIX`: appended to that version (`8.4.4-1.1ubuntu6.8` + `+nh2`).
- `patches/`: DEP-3 patches appended to Ubuntu's `debian/patches/series`.
- `build.sh`: the build, run in an `ubuntu:24.04` container.
- `.github/workflows/build.yml`: builds on every push to `noble-nexthop` (packages as a workflow
  artifact) and publishes a GitHub Release for tags named `noble/<version>`.

## The problem

zebra in FRR 8.4.4 ignores a remote MAC or neighbor update from bgpd when the update's
MAC-mobility sequence number is lower than the one zebra already holds
([FRRouting/frr#10468](https://github.com/FRRouting/frr/issues/10468)). After a VM moves,
bgpd can correctly select a route with a lower sequence number than one zebra installed
earlier. zebra then keeps forwarding to the old VTEP indefinitely, while
`show bgp l2vpn evpn route` shows the right next hop:

    show evpn mac vni 67413 mac 02:01:01:1a:00:1e
     Remote VTEP: 172.16.0.104   Remote Seq: 4       <- zebra and the kernel fdb
    show bgp l2vpn evpn route vni 67413 mac 02:01:01:1a:00:1e
     ... 172.16.0.102 ... MM:2, best                 <- bgpd

On 2026-11-27 four such entries had been left for five days by a rolling hypervisor
maintenance; each one blackholed traffic from that host to one VM.

## The patch

`zebra-evpn-accept-lower-bgp-seq.patch` backports the three zebra commits of
[FRRouting/frr#12081](https://github.com/FRRouting/frr/pull/12081) (first released in 8.5):
zebra accepts bgpd's sequence number for a remote entry. A local entry that is ready for
BGP keeps the higher one, as before. The hidden config command `no evpn accept-bgp-seq`
restores 8.4.4's behaviour at runtime.

The bgpd rework in the same pull request is not backported. It matters when a moved MAC
also has MAC+IP routes; on the hypervisors every MAC that moves is MAC-only (the MAC+IP
routes are each host's own bridge addresses).

Until this build is installed, a stale entry can be cleared without a restart by
re-advertising the MAC from the host that has it, one above the stale sequence number:

    vtysh -c 'configure terminal' -c 'router bgp' \
          -c 'bgp local-mac vni <vni> mac <mac> seq <n>'

## Rebasing on a newer Ubuntu update

Set `UBUNTU_VERSION` to the new version, push, and check that the patches still apply.
If Ubuntu's version moves past ours, bump `LOCAL_SUFFIX` or hold/pin `frr` on hosts that
run this build until Ubuntu ships the fix.
