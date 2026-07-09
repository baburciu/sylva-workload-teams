# sylva-workload-teams

[![My GitFut card](https://gitfut.com/baburciu.png)](https://gitfut.com/baburciu)

Workload-teams repo to test the **Git-driven** Sylva workload-cluster path against the `rke2-capo-baburciu-rocket` management cluster (CAPO).

## What's in here

```
kustomization.yaml          # generates ConfigMap "baburciu" (the team)  ← workload-teams-repo unit syncs this
team-baburciu.yaml          # team def: points the team GitRepository at cluster-definitions/
cluster-definitions/
  kustomization.yaml
  wc-rocket.yaml            # the SylvaWorkloadCluster (CAPO) ← workload-cluster-operator reconciles this
```

Flow: `workload-teams-repo` unit syncs the root → ConfigMap aggregated into `workload-team-defs`
→ chart creates namespace `swct-baburciu`, preset `default`, and a GitRepository/Kustomization
that applies `cluster-definitions/` → `workload-cluster-operator` reconciles the SWC →
`SylvaUnitsRelease` → Flux `sylva-units` HelmRelease → CAPO provisions the cluster.

## Step 0 — set your repo URL, then push

```shell
cd sylva-workload-teams
git init && git add -A && git commit -m "init"
git branch -M main
git remote add origin https://github.com/baburciu/sylva-workload-teams.git
git push -u origin main
```

## Step 1 — preconditions on the management cluster

```shell
export KUBECONFIG=management-cluster-kubeconfig   # must be reachable
# workload-cluster-operator already Ready (you confirmed)
# workload-team-defs unit hard-depends on these — confirm they're enabled/Ready:
kubectl -n sylva-system get hr | grep -E 'vault|vault-config-operator|external-secrets|sylva-units-operator'
kubectl -n sylva-system get ks sylva-units-operator'
```

If `vault` / `vault-config-operator` / `external-secrets-operator` are NOT enabled, enable them
(the `workload-team-defs` unit will not deploy without them).

## Step 2 — enable the teams machinery in the mgmt env-values

Add to `sylva-core/environment-values/rke2-capo-baburciu-rocket/values.yaml`:

```yaml
units:
  workload-cluster-operator:
    enabled: true
  workload-team-defs:
    enabled: true
  workload-teams-repo:
    enabled: true

# where the workload-teams-repo unit pulls THIS repo from
workload_teams_repo:
  repo_url: https://github.com/baburciu/sylva-workload-teams.git
  ref_type: branch
  ref_value: master
  repo_path: "."

# values passed to the workload-team-defs Helm chart
workload_team_defs:
  # Override the base preset so its sylva-core source is pinned to a real ref.
  # Use the ref your mgmt cluster runs; chart/mgmt skew causes failures.
  sylva_units_release_presets:
    default:
      annotations:
        workload-cluster-operator.sylva-project.org/preset-base: "true"
      spec:
        sylvaUnitsSource:
          type: git
          url: https://gitlab.com/sylva-projects/sylva-core.git
          branch: bb/issue-3809                              # clone hint
          commit: adcb7ec52e6302d5b3458afc347b73710de4c3fc   # exact pin = same commit as the mgmt cluster
        valuesFrom:
        - name: managed-workload-clusters-settings
          type: ConfigMap
          valuesKey: values

  # NOTE: do NOT set `managed_clusters_settings` here. It defaults to the rendered
  # `mgmt_cluster_state_values` (mgmt_cluster_ip, Thanos/Loki URLs, neuvector token).
  # An explicit map would REPLACE that default and the workload cluster would lose
  # all mgmt-state propagation. The rocket-specific proxy / registry-mirror / ntp /
  # storageClass settings live in the SylvaWorkloadCluster's spec.overridingValues
  # instead (see cluster-definitions/wc-rocket.yaml).
```

Apply the management cluster units:

```shell
cd sylva-core
./apply.sh environment-values/rke2-capo-baburciu-rocket
```

## Step 3 — drop in the OpenStack credentials (out-of-band, not in git)

Once `swct-baburciu` exists:

```shell
kubectl get ns swct-baburciu
kubectl apply -f ../capo-clouds-yaml.SECRET.local.yaml
```

## Step 4 — watch the Git-driven reconciliation

```shell
# teams layer synced?
kubectl -n workload-teams-repo get gitrepository,kustomization
kubectl -n sylva-system get cm workload-team-defs -o yaml | head
kubectl get surp -n swct-baburciu                 # preset "default" present
kubectl -n swct-baburciu get gitrepository,kustomization   # team "baburciu" syncing cluster-definitions/

# the SWC and what the operator generates
kubectl -n swct-baburciu get swc wc-rocket -w
kubectl get sur,hr -n swct-baburciu--wc-rocket
sylvactl watch -n swct-baburciu--wc-rocket Kustomization/swct-baburciu--wc-rocket/sylva-units-status

# CAPO provisioning
kubectl -n swct-baburciu--wc-rocket get cluster,openstackcluster,machine,openstackmachine
```

## Teardown

```shell
# remove the SWC from git (or kubectl delete it) -> ordered pre-delete hook -> CAPO deprovision
git rm cluster-definitions/wc-rocket.yaml && git commit -m "remove wc" && git push
# or:
kubectl delete swc wc-rocket -n swct-baburciu
```
