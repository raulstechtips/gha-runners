# gha-runners

## Purpose

This chart installs two GitHub Actions runner scale sets through the unchanged
`gha-runner-scale-set` dependency, version `0.14.2`, from
`oci://ghcr.io/actions/actions-runner-controller-charts`.
The wrapper version starts at `1.0.0`. CI publishes it to GHCR.

| Application | Values file in cluster-config | Runner scale set | Extra objects |
| --- | --- | --- | --- |
| gha-runners | apps/dev/gha-runners.yaml | self-hosted-k3s | GitHub App StaticSecret and LimitRange |
| gha-runners-packer | apps/dev/gha-runners-packer.yaml | self-hosted-packer | packer-http LoadBalancer Service |

Both applications use namespace `arc-runners`. Their Helm release names are the
Application names. The old Applications `gha-runner-scale-set` and
`gha-runner-scale-set-packer` were deleted and these releases reinstalled.
Keep the scale set names: GitHub sends jobs by these names. Scale set names make the
upstream object names different. Only the k3s application owns the shared Secret
configuration and LimitRange. Packer reads the same `arc-github-app` Secret.

## Before first install

1. Make sure the ARC controller and its CRDs are installed. This dependency does
   not contain a `crds/` folder. Neither live application tracks CRDs, so there is
   no `skipCrds` setting to add.
2. Make sure the Infisical operator supports `InfisicalStaticSecret` v1beta1 and
   that `default/infisical-prod-universal-auth` is ready.
3. Keep the Infisical keys `APP_ID`, `INSTALLATION_ID`, and `PRIVATE_KEY` in project
   `k3s-ntpo`, environment `prod`, path `/github/arc`. The recursive lookup remains
   enabled. Do not put key values in chart values.
4. Make sure the controller ServiceAccount is
   `arc-systems/gha-runners-controller-gha-rs-controller`.
5. Make sure the selected nodes exist. k3s uses `dev-k3s-node-3`; Packer uses
   `dev-k3s-node-2`. Keep the MetalLB address `10.29.30.181` available for Packer.
6. Publish the chart. Replace `PENDING_CI_VERSION` in both Argo files with the
   first CI version, for example `1.0.1-main-<sha>-<run>`.
7. Complete the Secret adoption step below before the first manual sync.

## Values reference

`charts/values.yaml` holds shared runtime settings. The upstream values remain
under `gha-runner-scale-set`; upstream templates are not copied or changed.
The schema rejects unknown wrapper keys and leaves upstream values open.

| Value | Default | Use |
| --- | --- | --- |
| infisical.authRef | Empty name and namespace | Shared InfisicalAuth reference |
| infisical.projectSlug / environmentSlug | Empty | Environment connection |
| githubAppSecret.enabled | false | Enable only in the k3s application |
| githubAppSecret.secretName | arc-github-app | StaticSecret and target Secret name |
| githubAppSecret.secretPath | /github/arc | Infisical path |
| githubAppSecret.recursive | true | Keep the live recursive lookup |
| limitRange.enabled | false | Enable once in the shared namespace |
| packerService.enabled | false | Enable only in the Packer application |
| packerService.loadBalancerIP | Empty | Environment-specific MetalLB address |
| packerService.port | Empty | Service port, set in the Packer values file |
| packerService.targetPort | Empty | Runner pod port, set in the Packer values file |
| gha-runner-scale-set | Shared live tuning | Upstream values passed through unchanged |

The defaults keep the GitHub organization URL, existing Secret reference, one
maximum runner, zero idle runners, default runner group, runner image and command,
resource settings, controller account, and anti-affinity rule.
Only the k3s values file sets `containerMode.type: dind`.
The Packer values file selects its scale set and lowers runner memory to `1Gi`
instead of `3Gi`. These are release-specific exceptions to the environment-only
values split because one chart serves both releases. Helm replaces container
lists, so Packer repeats the complete runner entry to change its memory limit.

The LimitRange keeps the original comments, limits (`500m`, `1Gi`), and requests
(`10m`, `256Mi`). The Packer values file sets port and target port `8802`. Packer HTTP keeps TCP, traffic policy
`Cluster`, and the `actions.github.com/scale-set-name: self-hosted-packer` selector.

## Upgrades

Use `/opt/homebrew/bin/helm`. Set `TMPDIR` to a private directory under
`/tmp/ulw-wrappers/gha-runners/` before local checks:

```sh
export TMPDIR=/tmp/ulw-wrappers/gha-runners/tmp
mkdir -p "$TMPDIR"
bash ../.omo/wrapper-charts/tools/checks.sh "$PWD" /tmp/ulw-wrappers/gha-runners
```

The checks build the locked dependency, lint defaults and CI values, run unit
tests, and validate both renders with kubeconform. The CI skip list excludes the
Infisical StaticSecret. AutoscalingRunnerSet uses the datree CRD schema and is
not skipped.

Change the upstream version in `charts/Chart.yaml`, rebuild `Chart.lock`, and
repeat the checks and both live parity comparisons before an upgrade.
The shared library `argoDeploy` step only bumps `argo/dev/gha-runners.yaml`.
It does not bump `argo/dev/gha-runners-packer.yaml`, although that file uses chart
`gha-runners` too. Update its target revision by hand until DEVSECOPS-180 provides
multi-target bumps. Sync each application manually.

## Adoption notes

Read-only verification on October 30, 2026 found six tracked objects for k3s and
five for Packer. Both applications had no sync options or ignore differences.
The new definitions keep manual sync and empty `syncPolicy`, use project
`platform`, and protect the Application with `Delete=false`.

Before the first sync, the operator must orphan-delete the old Secret sync object:

```sh
kubectl delete infisicalsecret infisical-secret-arc-github-app \
  -n arc-runners --cascade=orphan
```

This is a future operator step. It was not run during this work. The existing
`arc-github-app` Secret must remain. Sync the k3s application first, then check
that `InfisicalStaticSecret/arc-github-app` is ready before syncing Packer.
The new object uses `creationPolicy: Orphan`, Opaque type, 60-second refresh,
the same path and recursive lookup, and these exact target keys:
`github_app_id <- APP_ID`, `github_app_installation_id <- INSTALLATION_ID`,
`github_app_private_key <- PRIVATE_KEY`.

Expected render differences, also recorded in
`../.omo/evidence/wrappers/arc-runners/notes.md`:

- k3s `NEW InfisicalStaticSecret/arc-github-app` and
  `PRUNE InfisicalSecret/infisical-secret-arc-github-app`: the required API
  migration. The target Secret name and key mapping do not change.
- k3s `CHANGED LimitRange/runner-defaults`: adds the wrapper's name, instance,
  and managed-by labels. Its spec is unchanged.
- Packer `CHANGED Service/packer-http`: adds the same wrapper labels.
  Its rendered service settings match live. API-assigned fields are not pinned.
- Both `CHANGED AutoscalingRunnerSet` objects: only
  `actions.github.com/values-hash` changes. Helm injects `global: {}` into
  dependency values; upstream hashes all values. A direct upstream render with
  the old values reproduces each live hash. Adding only `global: {}` reproduces
  each wrapper hash. This difference cannot be removed through runtime tuning
  without changing upstream templates. Keep the upstream hash calculation;
  review its effect during the manual sync. Both runner specs match live.

All six upstream ServiceAccount, Role, and RoleBinding objects are `SAME`.
There are no unknown kinds. No other prune or new objects are reported.
The upstream runner image remains `latest` for parity; pinning it is outside this
adoption. Packer deliberately keeps `runner-scale-set: self-hosted-k3s` in both
the pod label and anti-affinity selector. It looks copied from k3s but changing it
would change scheduling during adoption.

Final evidence contains passing checks, both full renders, both parity reports,
and the hash comparison. The initial unit test assumed a modern Kubernetes
capability that helm-unittest does not use by default. The suite now sets version
1.33 explicitly to check the native dind sidecar branch. No cluster writes or
Secret value reads were performed. YAML language-server diagnostics were not
available because `yaml-language-server` is not installed; Helm, unit tests,
kubeconform, and yq provide the local validation.
