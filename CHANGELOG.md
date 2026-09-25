# Changelog


## [0.3.3] - 2026-09-25

### Changed
- Updated redis-exporter from v1.91.1 to v1.92.0


## [0.3.2] - 2026-09-12

### Fixed
- The bounded wait for a reachable master now measures wall clock instead of counting pauses, in both the master and the replica startup loops. Each iteration also runs `discover_master` and `sentinel_master`, each with its own connect timeouts and a scan of the headless service, so an iteration costs several times its two-second pause: the counter reached 30 while the clock was at 97 seconds, and a 70-second limit would not have fired for about four minutes. Both loops now set `SECONDS=0` and compare against it, so the limit means what it says.

### Why
Observed in production on the 0.3.0 → 0.3.1 upgrade. The deployment was left with every pod replicating an address that no longer existed, the replica sat in `Waiting for a reachable master...` past its limit, and it had to be resolved by hand. The bounded wait introduced in 0.3.1 was correct in shape and wrong in its unit; the master loop had carried the same flaw since the limit was introduced.


## [0.3.1] - 2026-09-12

### Fixed
- A chart version bump no longer restarts any pod. The replica and sentinel pod templates carried the full label set, which includes `helm.sh/chart` and `app.kubernetes.io/version`, and both ConfigMaps did too — the StatefulSets hash those rendered files into `checksum/configmap` and `checksum/health`, so the hash changed on every release even when the configuration was byte-identical. All four now use the stable selector labels. The checksums still change whenever the configuration actually changes, which is what they are for. Pods now restart only when the image or the configuration changes.
- A replica no longer waits forever for a master. Its startup loop asks the sentinels for the master and retried indefinitely, so when every pod was replicating an address that no longer existed — sentinels included — nothing could break the tie and the deployment stayed with no master until someone promoted a pod by hand. It now gives up after `2 × sentinel.downAfterMilliseconds + 10s` and starts as master, the same bounded wait the master pod already had, giving the sentinels a live candidate to converge on.

### Why this matters together with 0.3.0
0.3.0 stopped upgrades from *failing*. This release stops them from *restarting everything at once*: with the master, both replicas and all three sentinels rolling simultaneously, the sentinels could promote a replica that was itself about to restart, and the deployment ended up with no master. Observed in production on the 0.2.10 → 0.3.0 upgrade.


## [0.3.0] - 2026-09-12

### Fixed
- `volumeClaimTemplates` now use the stable selector labels (`app.kubernetes.io/name`, `/instance`, `/component`) instead of the full label set. The full set includes `helm.sh/chart` and `app.kubernetes.io/version`, which change with every release, and `volumeClaimTemplates` is immutable: every upgrade therefore failed with `Forbidden: updates to statefulset spec` unless the StatefulSets were deleted and recreated first. Nothing immutable changes between releases from now on.

### Changed
- The pre-upgrade hook is disabled by default (`preUpgradeHook.enabled: false`). It deleted the master, replica and sentinel StatefulSets with `--cascade=orphan` on every upgrade so Helm would recreate them, which is only needed when an immutable StatefulSet field changes — the problem fixed above. It is also harmful in sentinel mode: deleting all three at once restarts the sentinels together with the data pods, and the sentinels can promote a replica that is itself about to restart, leaving the deployment with no master. The sentinel StatefulSet has no `volumeClaimTemplates` and never needed recreating at all. With the hook enabled, an upgrade also depends on pulling `cgr.dev/chainguard/kubectl:latest` and stays in `pending-upgrade` when that pull fails.

### Upgrading from 0.2.10 or earlier
This release changes `volumeClaimTemplates`, so it is the **last** upgrade that touches an immutable field. Run it once with `--set preUpgradeHook.enabled=true`, or delete the StatefulSets yourself with `--cascade=orphan`. In sentinel mode, delete `-master` and `-replica` one at a time and **leave `-sentinel` running**. Existing PVCs keep their old labels; that is cosmetic and needs no action.


## [0.2.10] - 2026-09-11

### Fixed
- Sentinel mode did not recover after a node or cluster restart. Containers restart with a new pod IP while `/tmp` survives, so every sentinel reused `/tmp/sentinel.conf` pointing at the old master and at the old IPs of its peers: the master stayed `s_down`, no quorum could form, no failover happened, and replicas waited forever for a reachable master. A sentinel now reuses its saved config only while the monitored master still answers `ROLE` master; otherwise it discards it and discovers the current master as on a fresh install. After a full restart the master pod becomes master again even if a replica was master before, so writes made after the last failover are lost.
- Sentinel mode: a replica could stay attached to a master that no longer exists. During an upgrade the pre-upgrade hook recreates every StatefulSet, so the master, a replica and a sentinel restart at the same time; a promoted replica can be replaced seconds later and a pod that had just started as its replica keeps replicating the dead address. New sentinels start with a clean config and only learn replicas from the current master, so nobody reconfigures it. Replicas, and the master pod in sentinel mode, now use a liveness check that fails only when the replication link is down and the sentinels report a different master that answers as master; the pod restarts and discovers the current master.


## [0.2.9] - 2026-09-11

### Changed
- Updated Valkey from 9.0.3 to 9.1.2 (from Chainguard latest image)
- Updated redis-exporter from v1.82.0 to v1.91.1


## [0.2.8] - 2026-09-11

### Fixed
- Sentinel mode: replicas could not replicate with auth enabled. Master and replicas now set both `requirepass` and `masterauth`, so a former master can rejoin as a replica after a failover.
- Sentinel mode: `sentinel.conf` was rendered only when `sentinel.enabled` was true, independently of `architecture: sentinel`; every sentinel restart appended another `monitor` line and looped on `Duplicate master name`. The file is now rendered for `architecture: sentinel`, and a restarted sentinel reuses `/tmp/sentinel.conf` when it already monitors the master.
- Sentinel mode: `sentinel monitor` pointed at the master Service IP, which always selects the original master pod, so clients kept going to a demoted node after a failover. Every pod now asks the sentinels for the current master, accepts the answer only if that address replies `ROLE` master, and on a fresh install finds the master through the headless Service.
- Sentinel mode: a restarted master pod could stay a replica of its own dead IP forever, and a starting sentinel could wait indefinitely on a dead master. Both cases are now handled by the reachability check and a headless scan.
- Sentinel mode: startup scripts waiting for a master were killed by the liveness and readiness probes before their wait ended, deadlocking the whole StatefulSet. Added `startupProbe` to master, replica and sentinel (5 minutes by default).
- `replica.affinity` referenced a helper that does not exist and broke rendering; it now uses `toYaml` like master and sentinel.
- Standalone with auth: the startup script used `cp`, which the Chainguard image does not ship, so Valkey started with a config containing only `requirepass` (`maxmemory-policy noeviction` instead of `allkeys-lru`). Replaced with a bash builtin.

### Changed
- Sentinel defaults: `parallelSyncs` 1 → 10 and `failoverTimeout` 180000 → 30000. Every pod restart leaves an `s_down` replica entry with the old IP in the sentinels. With `parallel-syncs 1` the failover leader can spend its only reconfiguration slot on one of those addresses, and the per-replica timeout is only evaluated while a slot is free, so the failover lasts the whole `failover-timeout`; meanwhile the leader still reports the old master in `SENTINEL MASTERS`. Measured: ~3.5 minutes without a master for clients querying the leader, ~31 seconds with the new defaults.


## [0.2.7] - 2026-03-09

### Changed
- Updated redis-exporter from v1.81.0 to v1.82.0


## [0.2.6] - 2026-03-02

### Changed
- Updated redis-exporter from  to v1.81.0


## [0.2.5] - 2026-02-25

### Changed
- Updated Valkey from 9.0.2 to 9.0.3 (from Chainguard latest image)
- Updated redis-exporter from  to v1.81.0


## [0.2.4] - 2026-02-23

### Changed
- Updated redis-exporter from  to v1.81.0

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.3] - 2026-02-15

### Changed
- Update redis-exporter from `v1.80.0` to `v1.81.0` (latest stable release)
- Remove bash dependency from metrics exporter in replica and sentinel templates
- Metrics exporter now uses native entrypoint instead of bash wrapper (cleaner, more portable)
- Simplified metrics configuration across all deployment modes

### Added
- Automated weekly version checking workflow for Valkey
- Documentation for image versioning strategy
- Complete CHANGELOG.md with release history

### Fixed
- Remove redundant password configuration in metrics exporter (was configured twice)

### Note
- Chainguard `prometheus-redis-exporter` image requires authentication (not in public free tier)
- Using `oliver006/redis_exporter:v1.81.0` for public accessibility
- Template improvements support both oliver006 and distroless images

## [0.2.1] - 2026-02-15

### Changed
- Updated Valkey to version 9.0.2 (from Chainguard latest image)

## [0.2.0] - 2025-02-14

### Changed
- Switch to Chainguard zero-CVE images for enhanced security (valkey, kubectl, wolfi-base)
- Update container user from 999 to 65532 (Chainguard default)
- Simplify health check scripts for distroless compatibility
- Update pre-upgrade hook to work without shell

### Added
- Automated version checking workflow (runs weekly)
- Documentation for image versioning strategy
- CHANGELOG.md for tracking releases

### Security
- Migration to Chainguard images with zero known CVEs
- Enhanced security with distroless base images

## [0.1.0] - 2024

### Added
- Initial release of Valkey Helm Chart
- Standalone mode support
- Sentinel mode for high availability
- Authentication and security features
- Persistence configuration
- Prometheus metrics exporter
- TLS support
- Pre-upgrade hooks for zero-downtime migrations
- Network policies and RBAC
- Comprehensive documentation

---

**Note**: Starting from v0.2.0, this chart uses `cgr.dev/chainguard/valkey:latest` and the `appVersion` is automatically updated weekly via GitHub Actions when new Valkey versions are released.
