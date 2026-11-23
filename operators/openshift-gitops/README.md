# OpenShift GitOps operator

Install the Red Hat OpenShift GitOps Operator through classic OpenShift OLM.
This follows the repository's Namespace, OperatorGroup and Subscription pattern,
with version settings supplied by a Kustomize patch and manual InstallPlan approval.

The operator runs in `openshift-gitops-operator` and watches all namespaces.
Unlike the AAP OperatorGroup, its OperatorGroup has no `targetNamespaces` or
selector. With its default settings, the operator creates the Argo CD instance
named `openshift-gitops` in namespace `openshift-gitops`. This overlay does not
create another ArgoCD resource or override the default instance's configuration.
See [Red Hat's CLI installation procedure](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.21/html/installing_gitops/installing-openshift-gitops#installing-openshift-gitops-operator-using-cli_installing-openshift-gitops).

## Layout and version

```text
operators/openshift-gitops/
├── README.md
├── base
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── operatorgroup.yaml
│   └── subscription.yaml
└── overlays
    └── 1.22
        └── 1.22.3
            ├── kustomization.yaml
            └── subscription-patch.yaml
```

The overlay selects `gitops-1.22` and
`openshift-gitops-operator.v1.22.3`, observed together in a target cluster's
`redhat-operators` package catalog on **2026-11-23**. That observation does not
establish availability or compatibility on every cluster. Verify the catalog
and supported OpenShift versions for your intended installation.

Do not apply `base/` directly: its channel and starting CSV are placeholders.
To select another release, copy the overlay and update both fields in
`subscription-patch.yaml`. `startingCSV` selects the initial bundle; it does not
force upgrades, downgrades or reinstalls. Manual approval holds subsequent
upgrades unless another controller or administrator approves their plans.

## 1. Check the target cluster

Use an administrator account with operator-installation permissions. The cluster
needs classic OLM, the Red Hat catalog and access to the required images.
Run these commands from the repository root:

```sh
oc config current-context
oc get catalogsource redhat-operators -n openshift-marketplace
oc get packagemanifest openshift-gitops-operator \
  -n openshift-marketplace -o yaml
oc get subscriptions.operators.coreos.com -A
oc get operatorgroups.operators.coreos.com -A
oc get namespaces openshift-gitops-operator openshift-gitops
```

Missing namespaces are expected on a fresh cluster. Verify the package's catalog
source, `gitops-1.22` channel and selected bundle. A channel's `currentCSV` is
its head, not the complete list of available versions. Check available entries
or catalog bundle data if the head has advanced.

These instructions assume a fresh installation. If GitOps already exists, inspect
its Subscription, OperatorGroup, ownership and Argo CD configuration before
adopting these manifests. Do not create a second OperatorGroup in the same
namespace or a competing Subscription for the same operator. Applying this
overlay to an existing installation can change its channel and approval policy;
it does not migrate the installation or restore default instance settings.
Red Hat requires resolving a Community Argo CD Operator installation before
installing OpenShift GitOps; do not treat this overlay as a migration procedure.

## 2. Bootstrap the operator

Use a local checkout to bootstrap GitOps before it can manage Applications. For
a private repository, authenticate your workstation's Git client to clone it;
Argo CD repository credentials are configured separately after installation.

```sh
oc kustomize operators/openshift-gitops/overlays/1.22/1.22.3
oc apply -k operators/openshift-gitops/overlays/1.22/1.22.3
```

The overlay creates the operator Namespace, OperatorGroup and Subscription.
The namespace enables recommended cluster monitoring. Installation waits for
manual approval; applying these three resources does not mean Argo CD is ready.

For optional admission validation, use `oc apply --dry-run=server --validate=strict`
with the same `-k` path. On a fresh cluster, the Namespace must already exist
before namespaced resources can pass server dry-run: a dry-run Namespace is not
persisted for the other requests. Do not mistake that limitation for a manifest
error. Server dry-run validates API admission, not OLM resolution or readiness.

## 3. Review and approve the exact InstallPlan

```sh
oc get subscription openshift-gitops-operator \
  -n openshift-gitops-operator -o yaml
oc get installplans -n openshift-gitops-operator
```

Wait for the Subscription's `status.installPlanRef.name`. Substitute that name
for `install-REPLACE-ME` and inspect the plan:

```sh
oc get installplan install-REPLACE-ME -n openshift-gitops-operator -o yaml
```

Confirm `spec.clusterServiceVersionNames` includes
`openshift-gitops-operator.v1.22.3`, review dependency CSVs and resources, and
confirm this is the intended initial installation. If it selects a different
GitOps version, inspect Subscription conditions and catalog availability before
proceeding. After reviewing, approve only that plan:

```sh
oc patch installplan install-REPLACE-ME \
  -n openshift-gitops-operator --type=merge \
  -p '{"spec":{"approved":true}}'
```

Leave future upgrade plans unapproved until their contents have been reviewed.
Do not bulk-approve unrelated plans. See the
[OLM specific-version procedure](https://olm.operatorframework.io/docs/tasks/install-operator-with-olm/#example-install-a-specific-version-of-an-operator).

## 4. Verify readiness

Once the selected CSV exists, wait for it to succeed and the ArgoCD CRD to be
established:

```sh
oc wait --for=jsonpath='{.status.phase}'=Succeeded \
  csv/openshift-gitops-operator.v1.22.3 \
  -n openshift-gitops-operator --timeout=600s
oc get subscription openshift-gitops-operator \
  -n openshift-gitops-operator \
  -o jsonpath='{.status.installedCSV}{"\n"}'
oc wait --for=condition=Established crd/argocds.argoproj.io --timeout=120s
oc get pods -n openshift-gitops-operator
oc get argocd openshift-gitops -n openshift-gitops -o yaml
oc get pods -n openshift-gitops
oc get route openshift-gitops-server -n openshift-gitops
```

The installed CSV must match the selected release. Allow reconciliation to
create the default instance, its pods and Route; repeat the inspection if they
do not exist yet. Confirm the operator and instance pods become Ready, review
the ArgoCD status for errors, and verify the Route is admitted and its UI is
reachable. Log in through the configured OpenShift authentication. A successful
CSV alone does not prove instance readiness or repository access. If installation
stalls, inspect Subscription conditions, InstallPlan status, CSV status and
namespace events before approving any additional plan.

## 5. Configure repository access and deploy AAP

For a private repository, register credentials with the Argo CD instance before
creating the AAP Applications. Repository Secrets belong in `openshift-gitops`,
not the operator namespace. Use a read-only SSH deploy key, HTTPS access token,
or GitHub App with repository Contents read access. Register through Argo CD's
**Settings → Repositories** or its supported repository Secret configuration.
The repository URL must match the Applications' `spec.source.repoURL`; SSH
credentials require an SSH URL. Preserve SSH host-key or HTTPS certificate
verification. Keep actual tokens, private keys and populated Secrets out of Git.

See [Argo CD private repository authentication](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
and [repository Secret examples](https://argo-cd.readthedocs.io/en/stable/operator-manual/argocd-repositories-yaml/).
Confirm the repository connection succeeds before refreshing the Applications.
Provision credentials separately in each cluster's GitOps installation.

Continue with the [QA Application guide](../../examples/argocd/qa/README.md),
or the [generic Application guide](../../examples/argocd/README.md).
Their AppProject and controller permissions still need to allow the AAP resources;
repository authentication alone does not grant deployment permissions. The AAP
Route's manually created TLS Secret is separate from repository credentials.

This bootstrap overlay does not create AAP resources, repository credentials,
additional RBAC or an Application managing GitOps itself. Validate the installation
before continuing with the existing manual AAP operator and instance sync steps.
