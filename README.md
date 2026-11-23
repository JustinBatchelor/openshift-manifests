# OpenShift manifests

Kustomize manifests for installing and configuring workloads on OpenShift.
Includes OpenShift GitOps operator installation and Ansible Automation Platform
operator and instance deployments.

## OpenShift GitOps

Use the [GitOps operator guide](operators/openshift-gitops/README.md) to bootstrap
OpenShift GitOps before deploying the AAP Argo CD Application examples. The
Kustomize base and version overlay install the operator in
`openshift-gitops-operator` with all-namespace scope and manual OLM approval.
The operator creates the default Argo CD instance in `openshift-gitops`.

The included `operators/openshift-gitops/overlays/1.22/1.22.3` overlay selects
channel `gitops-1.22` and starting CSV `openshift-gitops-operator.v1.22.3`.
Render it from the repository root:

```sh
oc kustomize operators/openshift-gitops/overlays/1.22/1.22.3
```

Follow the guide for prerequisites, installation, readiness checks and separate
private-repository authentication. Inspect existing installations before applying;
the example does not migrate an existing GitOps deployment.

## Ansible Automation Platform

Installs the namespace-scoped `ansible-automation-platform-operator` through
classic OpenShift OLM into the dedicated `ansible-automation-platform` namespace.
The operator manifests create a Namespace, OperatorGroup, and Subscription.
After the operator is installed, deploy the
[platform instance](instances/ansible-automation-platform/README.md)
to enable Controller, Automation Hub, and Event-Driven Ansible using the cluster's
default storage class (RWX support is required for Hub).

For QA GitOps deployment, use the [QA Argo CD Application examples](examples/argocd/qa/README.md).
They reference the operator and instance Kustomize overlays separately and walk
through OLM installation, manual TLS Secret creation and instance synchronization.

## Version options

The included overlays select these AAP 2.7 patch releases from Red Hat's documentation,
checked on November 18, 2026. Both use the `stable-2.7` channel and
namespace-scoped CSVs. Catalog availability on the target cluster must be checked
before installation; its API hostname did not resolve during preparation.

| Overlay | Release date | Exact starting CSV |
| --- | --- | --- |
| `overlays/2.7/12.3` | November 11, 2026 | `aap-operator.v2.7.0-0.1794269743` |
| `overlays/2.7/12.2` | November 2, 2026 | `aap-operator.v2.7.0-0.1793374941` |

Release sources: [2.7-12.3](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-aap_2026_11_11),
[2.7-12.2](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/whats_new-aap_2026_11_02).

## Repository layout

Operator installation:

```text
operators/ansible-automation-platform/
├── base
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── operatorgroup.yaml
│   └── subscription.yaml
└── overlays
    └── 2.7
        ├── 12.2
        │   ├── kustomization.yaml
        │   └── subscription-patch.yaml
        └── 12.3
            ├── kustomization.yaml
            └── subscription-patch.yaml
```

Platform instance:

```text
instances/ansible-automation-platform/
├── README.md
├── base
│   ├── kustomization.yaml
│   └── ansibleautomationplatform.yaml
└── overlays
    ├── 2.7
    │   ├── kustomization.yaml
    │   └── platform-patch.yaml
    ├── 2.7-custom-hostname
    │   ├── kustomization.yaml
    │   └── platform-patch.yaml
    ├── qa
    │   ├── kustomization.yaml
    │   └── platform-patch.yaml
    └── prod
        └── .gitkeep
```

QA is the configured environment example. It enables Controller, Hub and EDA,
requests `100G` of managed database storage and `10Gi` of Hub RWX file storage,
and uses a custom Route with a manually created TLS Secret. Edit QA's
[`platform-patch.yaml`](instances/ansible-automation-platform/overlays/qa/platform-patch.yaml)
for hostname, TLS Secret reference and storage sizes. Render it from the repository
root:

```sh
oc kustomize instances/ansible-automation-platform/overlays/qa
```

Follow the [instance guide](instances/ansible-automation-platform/README.md)
for operator installation prerequisites, DNS, manual TLS Secret preparation and
apply/verification steps, or use the [QA Argo CD guide](examples/argocd/qa/README.md).
QA and future PROD are intended for separate clusters, each with its own GitOps
installation. PROD is reserved with an empty `.gitkeep` file only; it has no
Kustomization, deployable manifests or Argo CD Application.

The existing `2.7` (generated hostname) and `2.7-custom-hostname` overlays and
generic Argo CD examples remain available with their original settings. Every
deployable instance overlay renders `aap` without a suffix. Select one overlay
and one Application pair per cluster; these configurations manage the same
resource identities in `ansible-automation-platform`.

**Name change:** older revisions rendered `aap-jbatchel`. Applying this revision
does not rename that resource or migrate its data; inspect existing instances
before deploying. See the instance guide for transition considerations.

## How pinning works

`startingCSV` selects the initial version for a new Subscription.
`installPlanApproval: Manual` prevents both the initial installation and subsequent
upgrades until their InstallPlans are approved. Approve the initial plan for the
chosen CSV, then leave newer plans unapproved to retain that version. This follows
the [OLM specific-version installation procedure](https://olm.operatorframework.io/docs/tasks/install-operator-with-olm/#example-install-a-specific-version-of-an-operator).

Changing `startingCSV` on an existing installation does not force an upgrade or
downgrade. An upgrade must follow the catalog's supported upgrade path and requires
reviewing and approving the appropriate plan. Any external controller that approves
InstallPlans can defeat this manual hold.

## Check the target cluster

Log in to the intended OpenShift cluster with permission to install operators.
It must have OLM, the `redhat-operators` catalog, support for the chosen AAP release,
and access to the required Red Hat images.

```sh
oc config current-context
oc get catalogsource redhat-operators -n openshift-marketplace
oc get packagemanifest ansible-automation-platform-operator \
  -n openshift-marketplace -o yaml
```

Check `status.catalogSource`, the `stable-2.7` channel, and its `entries` (when
provided) for the chosen CSV. `currentCSV` is only the channel head, not a full
version list. If the selected bundle is absent, use a catalog containing that
bundle or select an available version. Review existing OperatorGroups and
Subscriptions before installing into an already-used namespace; this layout
assumes a fresh dedicated namespace.

## Render and install one option

```sh
# From the repository root:
cd operators/ansible-automation-platform

# Preview 2.7-12.3; this does not contact the cluster.
oc kustomize overlays/2.7/12.3

# Create the three resources. Installation waits for manual approval.
oc apply -k overlays/2.7/12.3
```

To choose an older option for a fresh install, use
`oc apply -k overlays/2.7/12.2` instead.
Apply only one option. Do not apply `base/` directly: its version fields are
deliberate placeholders. To use another CSV, copy an overlay and edit its
`subscription-patch.yaml` channel and startingCSV together.

## Review and approve the initial InstallPlan

Wait for OLM to resolve the Subscription and populate `status.installPlanRef`:

```sh
oc get subscription ansible-automation-platform \
  -n ansible-automation-platform -o yaml
oc get installplans -n ansible-automation-platform
```

Copy the plan name from that Subscription's `status.installPlanRef.name`.
Replace `install-REPLACE-ME` below with that exact name and inspect the plan:

```sh
oc get installplan install-REPLACE-ME \
  -n ansible-automation-platform -o yaml
```

Confirm `spec.clusterServiceVersionNames` includes the selected starting CSV,
review any dependency CSVs and resources in the plan, and confirm it is the
intended initial install. If it names a different AAP version, do not approve it;
inspect the Subscription's conditions and catalog availability instead.

After reviewing, approve that specific plan:

```sh
oc patch installplan install-REPLACE-ME \
  -n ansible-automation-platform --type=merge \
  -p '{"spec":{"approved":true}}'

oc get csv -n ansible-automation-platform
oc get subscription ansible-automation-platform \
  -n ansible-automation-platform \
  -o jsonpath='{.status.installedCSV}{"\n"}'
```

The selected CSV should reach `Succeeded`, and `status.installedCSV` should match
the requested version. Keep future upgrade plans unapproved to retain the pin.

Namespace and OperatorGroup layout follows the
[Red Hat CLI installation instructions](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/install-assembly_installing_aap_operator_cli).
If changing the namespace, update the Namespace name, both resource namespaces,
the OperatorGroup's `targetNamespaces`, and the overlay patch namespaces together.
