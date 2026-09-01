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

### OpenLDAP + Keycloak (SSO -- phase 1, directory + IdP only)
Goal: one identity for ArgoCD, Grafana, and SSH to all three NUCs. That needs two pieces, not one --
Keycloak alone (its bundled Postgres is just Keycloak's own storage) has no LDAP server interface for
anything else to bind against, and SSH doesn't speak OIDC. So:
1. **OpenLDAP** (`jp-gouin/helm-openldap` community chart -- Bitnami dropped their own openldap chart
   from the catalog) is the actual directory, source of truth for accounts.
2. **Keycloak** (bitnami/keycloak + bundled Postgres) federates to that LDAP and is what ArgoCD/Grafana
   will authenticate against via OIDC.
3. SSSD/PAM on each NUC will authenticate SSH directly against the same LDAP.

This phase deploys and verifies **only #1 and #2** -- directory up, one test user seeded, Keycloak
console reachable. ArgoCD/Grafana OIDC wiring and the NUC SSSD/PAM piece (Ansible) are deliberately
separate follow-ups, not done here.

Base DN `dc=homelab,dc=local`. Both charts have credentials pinned via a SealedSecret
(`openldap-secrets`/`keycloak-secrets` apps) from the start, rather than letting either chart
auto-generate its own -- Grafana and MinIO both had their generated passwords silently rotate on
every ArgoCD sync (selfHeal "correcting" the chart's own random-password template re-rendering
differently each comparison), and this sidesteps that class of bug entirely instead of discovering it
again later.

`apps/openldap.yaml` seeds `ou=people`, `ou=groups`, and one `testuser` account (with
`posixAccount`/`posixGroup` attributes already in place for the later SSSD phase) via
`customLdifFiles`. That LDIF is a plain ConfigMap, not sealed, so it intentionally carries no
`userPassword` -- set one after deploy with `ldappasswd`, the same pattern already used for ArgoCD's
own initial admin password (fetched once via `kubectl`, never stored in git).

Both LDAP and Keycloak run plaintext (no TLS) -- traffic stays inside the cluster network, same risk
posture already accepted for Grafana/MinIO's ingress. Worth revisiting before SSSD is wired up on the
NUCs, since that traffic would be leaving the cluster.

**One gotcha caught before it shipped**: the openldap chart defaults `replication.enabled: true`
regardless of `replicaCount`. At `replicaCount: 1` that renders a syncrepl config with the pod
pointed at itself (confirmed via `helm template` -- `LDAP_REPLICATION_HOSTS` listed only the pod's
own headless DNS name). Explicitly disabled since there's nothing to replicate to.

**Status: #1 and #2 are both deployed and verified.** `ldapsearch` against the live pod (bound as
`cn=admin,dc=homelab,dc=local`, using the sealed credential) returns the seeded `testuser` entry with
all its attributes intact. Keycloak returns `200` on `/realms/master` with a clean startup log. Three
more bugs turned up getting there, all unrelated to the directory/IdP design itself:

1. **`openldap`'s `phpldapadmin` Ingress is stuck on `extensions/v1beta1`**, an API group removed from
   Kubernetes years ago. The `openldap` chart line is frozen at `2.0.4` -- upstream moved on to a
   differently-named successor chart (`openldap-stack-ha`) instead of fixing this one. ArgoCD fails
   the *entire* Application sync atomically on one invalid resource, so nothing in it applied at all
   until this was found. Fixed by disabling just that ingress (`phpldapadmin.ingress.enabled: false`);
   `phpldapadmin` itself still deploys fine, reachable via `kubectl port-forward` for now.

2. **Both `bitnami/keycloak` and its bundled `bitnami/postgresql` 404'd on `docker.io`.** Bitnami moved
   pinned version tags behind a paid tier in 2025, leaving only `latest` free on `docker.io/bitnami/*`
   -- older tags are mirrored to a separate `bitnamilegacy` org instead (confirmed both images exist
   there via `docker manifest inspect`). Both `apps/keycloak.yaml`'s top-level `image.repository` and
   `postgresql.image.repository` now point at `bitnamilegacy/*`.

3. **Keycloak OOMKilled itself on first boot.** The chart's default `resourcesPreset: small` caps
   memory at 768Mi, which isn't enough for Keycloak 26's real first-boot footprint (confirmed via
   `containerStatuses.lastState`: exitCode 137, reason `OOMKilled`, mid-Liquibase schema migration).
   The node had 15.7Gi allocatable at 4% usage, so this was the container's own limit, not real node
   pressure. Fixed with an explicit `resources` block (1536Mi limit) overriding the preset.

### SSO phase 2: ArgoCD + Grafana login via Keycloak
Goal from phase 1: verify LDAP + Keycloak core, defer wiring up the actual apps. This phase does that
wiring for ArgoCD and Grafana -- both now log real users in through Keycloak, federated back to the
same LDAP directory. NUC SSSD/PAM (SSH against the same LDAP) is still a separate follow-up.

**ArgoCD** is configured directly in `homelab`'s Terraform (`main.tf`), not here -- Terraform already
owns the `argo-cd` Helm release, and `argocd-cm`/`argocd-secret` are objects that release renders.
An ArgoCD Application patching those same objects would fight this Helm release on every
reconciliation, the exact Terraform/ArgoCD collision this repo split exists to avoid. OIDC is wired
directly (dex disabled) since ArgoCD talks to a provider natively and dex would be a redundant hop.
The client secret is a `sensitive` Terraform variable (`TF_VAR_argocd_oidc_client_secret`, stored in
1Password, never committed), matching the existing MinIO backend credential pattern.

**Grafana** OIDC lives here as usual, in `apps/kube-prometheus-stack.yaml` -- client secret pinned via
SealedSecret (`kube-prometheus-stack-secrets`), injected as `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET`.

**Status: both verified with a real end-to-end login** -- not just "pods are up." Simulated the full
browser OAuth dance with `curl` (authorize redirect -> Keycloak login form -> callback -> real session
token/cookie), then confirmed actual authorized access (a real ArgoCD API call succeeding, Grafana's
`/api/user` returning the right identity with the right org role) rather than stopping at "got
redirected to a login page." That rigor is what surfaced every bug below -- none of them would show up
just from checking that pods are `Running`.

**Real bugs hit, all fixed:**

1. **ArgoCD login silently didn't work at all -- zero errors, zero debug output.** The Keycloak LDAP
   federation component (from phase 1) was created with `parentId: "homelab"` -- the realm's *name*,
   which is what the REST API's URL path uses (`/admin/realms/homelab/...`). But a component's
   `parentId` field needs the realm's actual internal ID, a separate random UUID (confirmed via
   `LOGIN_ERROR` events in the server log: `realmId="4bc22b35-..."`, not `"homelab"`). Silently wrong
   for every purpose that mattered -- `testAuthentication` still passed (that's a raw bind test, not
   realm-scoped), so nothing about setup looked broken. Fixed by updating `parentId` to the real UUID.

2. **ArgoCD RBAC denied access even with a policy that should work by the book.** `g,
   testuser@homelab.local, role:admin` is the textbook ArgoCD RBAC example for granting an SSO user
   admin -- and it didn't work, denied with no further detail. Never conclusively identified which
   claim ArgoCD's enforcer actually matches against in this version; rather than keep guessing,
   granted explicit wildcard `p,` policies plus `g,` role-grants against all three plausible subject
   identifiers (email, the `sub` UUID, `preferred_username`) at once, which resolved it. Worth
   revisiting to narrow down the real cause if it ever matters (e.g. adding a second real user).

3. **Grafana's OAuth redirect_uri used its internal container port.** Came back as
   `http://grafana:3000/login/generic_oauth` (confirmed by capturing the actual authorize request),
   not the real ingress URL -- Grafana's default `root_url` template combines `[server] domain`
   (correctly `grafana`, from `ingress.hosts`) with the default `http_port` (3000). Keycloak rejects
   it since only the real public URL is a registered redirect URI on the client. Fixed with an
   explicit `root_url: http://grafana/`.

4. **Grafana's `role_attribute_path` evaluated to nothing.** Set to the JMESPath literal `'Admin'`
   (the standard pattern for "everyone gets the same role") in `grafana.ini`, but debug logs
   (`org.sync`) showed the resolved role as completely empty, falling back to `Viewer`. Root cause:
   Grafana's own INI-value parser strips matching leading/trailing quote characters as normal INI
   quoting convention, silently turning the quoted literal into the bare word `Admin` -- which
   JMESPath then reads as a field lookup (finds nothing) instead of a string literal. Env vars don't
   go through that parser, so `role_attribute_path` is set via
   `GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH` in `env` instead, everything else stays in `grafana.ini`.

5. **Grafana didn't sync the (correctly-evaluating) role at all, even after fixing #4.** Grafana 13's
   `generic_oauth` provider defaults to *not* syncing org role from the IdP on every login unless
   `skip_org_role_sync` is explicitly set to `false`. Without it, role is only set once at account
   creation from `auto_assign_org_role` (default `Viewer`) and never updated again after that.

**Also learned along the way, not a bug**: ArgoCD Applications sourced from this repo -- both the
Helm-values kind (`kube-prometheus-stack`, `openldap`) and the plain-manifest kind (`*-secrets`) --
can lag behind a fresh push by a full poll cycle even after `app-of-apps` itself shows `Synced`. A
`kubectl annotate application <name> argocd.argoproj.io/refresh=hard` on `app-of-apps` first, *then*
on the child Application, reliably pulls in a just-pushed commit immediately instead of waiting on
the default poll interval -- useful when iterating on a fix and re-verifying it live.

## Bootstrapping onto a fresh cluster
```
kubectl apply -f bootstrap/app-of-apps.yaml
```
That's the only manual `kubectl apply` this repo ever needs -- ArgoCD takes it from there.
