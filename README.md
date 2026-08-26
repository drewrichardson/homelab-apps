# Homelab App Manifests (ArgoCD)

GitOps repo for everything ArgoCD deploys onto the [homelab](https://github.com/drewrichardson/homelab)
k3s cluster, kept deliberately separate from that infra-bootstrap repo -- this one changes on a much
faster, less careful cadence (app config, image bumps) than cluster bootstrap does.

## Structure (app-of-apps pattern)
- `bootstrap/app-of-apps.yaml` -- applied **once, manually**, via `kubectl apply -f`. Everything
  after that, ArgoCD manages by watching `apps/` in this same repo.
- `apps/` -- one ArgoCD `Application` per app, each pointing at that app's own directory here.
- `home-assistant/` -- the actual Kubernetes manifests per app.

## Apps

### Home Assistant
Migrated from a standalone HAOS install on a Raspberry Pi to the `homeassistant/home-assistant`
container image. Notable choices:
- `hostNetwork: true` -- required for mDNS/SSDP-based local device discovery (HomeKit, Cast, Roomba,
  etc.) to keep working; an isolated pod network can't see that LAN broadcast traffic.
- Pinned to a specific node (`nodeSelector`) on purpose -- with `hostNetwork`, the pod's network
  identity *is* the node's, so letting it float between nodes would change its IP on every
  reschedule.
- `strategy: Recreate`, not the default rolling update -- the pod owns a fixed host port and a
  `ReadWriteOnce` volume, so a rolling update would try to start the replacement before killing the
  original and collide on both.
- Lost with the move off HAOS: the Supervisor and its add-on store. Only two add-ons were in use --
  Matter Server (not currently deployed here -- wasn't actually in use, removed during migration; the
  stored Matter integration config still points at the old add-on hostname if it's ever added back)
  and Terminal & SSH (not needed; `kubectl exec` covers it).
- Known follow-ups from the migration, not yet done: sign back into Home Assistant Cloud (Nabu Casa)
  fresh via the UI -- the migrated config carried over some HAOS-specific assumptions that may need
  re-establishing; remove the now-permanently-broken `rpi_power` integration (Pi-specific hardware
  check, inapplicable off a Pi) via the UI.

**A debugging note worth keeping**: mid-migration this looked like a genuine Home Assistant boot-hang
bug in image `2026.8.2` -- the HTTP server never came up, across many restarts, with every integration
disabled, with and without `hostNetwork`, across multiple image versions. Real root cause: a flawed
`pgrep`-based PID lookup in the diagnostic script used to check whether the process was still making
progress, which produced false "0% CPU, frozen" readings. `2026.8.2` was working correctly the entire
time. Lesson: when a diagnostic result conflicts with other evidence (here, later confirmed by `curl`
returning real content from a pod that the same script called "frozen"), verify the diagnostic itself
before trusting hours of results built on it.

### kube-prometheus-stack (Prometheus + Grafana + node-exporter)
Grafana at `http://grafana` via Traefik ingress (k3s ships Traefik by default). node-exporter's
`hwmon` collector is explicitly enabled for hardware sensor metrics (CPU temperature) -- needs
`lm-sensors` + the `coretemp` kernel module on the host, handled by the Ansible bootstrap playbook
in the `homelab` repo. A "NUC Hardware" dashboard is auto-provisioned via a ConfigMap (see
`monitoring-dashboards/`) using real thresholds pulled from the CPU's own reported critical
temperature, not guessed values.

**Two real bugs hit getting this running, worth keeping as reference:**

1. **CRDs stuck permanently OutOfSync -> Prometheus pod never existed.** This chart's CRDs
   (`Prometheus`, `Alertmanager` especially) are large enough to exceed the size limit on
   `kubectl apply`'s client-side `last-applied-configuration` annotation, which is what ArgoCD's
   default sync strategy uses. They silently never applied -- `kubectl get prometheus` errored "the
   server doesn't have a resource type," meaning the whole monitoring stack looked like it deployed
   (Grafana, node-exporter, kube-state-metrics all came up fine) while its actual data source never
   existed. Fixed by adding `ServerSideApply=true` to this Application's `syncOptions` (doesn't use
   that annotation, no size limit) -- see `apps/kube-prometheus-stack.yaml`.

2. **The Prometheus Operator caches CRD availability at startup and never rechecks.** Even after the
   CRDs above were force-applied to unblock the immediate outage, the operator (already running,
   started before the CRDs existed) had logged `resource "prometheuses" ... not installed in the
   cluster` at boot and simply never watched that resource type again. Fix: restart the operator
   pod any time CRDs are added/fixed out from under it --
   `kubectl rollout restart deployment kube-prometheus-stack-operator -n monitoring`.

**Also expected, not a bug**: this Application will show `OutOfSync`/`Degraded` in ArgoCD as long as
greyskull-2/3 are offline. Its sync operation waits for the node-exporter DaemonSet to be fully
healthy before proceeding to later resources in the same sync wave, and a DaemonSet with pods
`Pending` on offline nodes never reaches that state. Self-resolves once those nodes are back --
no action needed.

## Bootstrapping onto a fresh cluster
```
kubectl apply -f bootstrap/app-of-apps.yaml
```
That's the only manual `kubectl apply` this repo ever needs -- ArgoCD takes it from there.
