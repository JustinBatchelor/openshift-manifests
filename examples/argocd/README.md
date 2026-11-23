# Deploy AAP with Argo CD

For the configured QA environment, use the [QA Application examples](qa/README.md).
They select the QA overlay with `100G` database storage and `10Gi` Hub storage.
The generic examples below retain their original paths and sizes. Use one pair
per cluster; both pairs would manage the same underlying resources. PROD has no
Application or deployable configuration yet.

These two `Application` examples reference this repository's Kustomize overlays:

| Application | Source path | Managed resources |
| --- | --- | --- |
| [`aap-operator`](operator-application.yaml) | `operators/ansible-automation-platform/overlays/2.7/12.3` | Namespace, OperatorGroup and Subscription |
| [`aap-instance`](instance-application.yaml) | `instances/ansible-automation-platform/overlays/2.7-custom-hostname` | AnsibleAutomationPlatform named `aap` |

Argo CD detects `kustomization.yaml` in each source path and renders that overlay.
The operator then creates the platform's workloads. See the official
[Kustomize integration guide](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/).

The Applications use manual sync. Install the operator first, approve its OLM
InstallPlan, wait for it to become ready, create the TLS Secret, and then sync
the instance. Independent Applications do not establish this dependency merely
by being created together. Skipping dry-run for a missing CRD does not install
the operator or make it ready.

## Prerequisites and fields to change

Use an existing Argo CD or OpenShift GitOps installation and log in with its CLI
or UI. For a new installation, first follow the
[GitOps operator bootstrap guide](../../operators/openshift-gitops/README.md).
These Application examples do not install GitOps, create an AppProject or grant RBAC.
Run commands below from this repository's root using the intended cluster context.

| Field in both Applications | Example value | When to change it |
| --- | --- | --- |
| `metadata.namespace` | `openshift-gitops` | Set to the namespace of your Argo CD instance, such as `argocd` |
| `spec.project` | `default` | Select an existing AppProject allowed to manage these resources |
| `spec.source.repoURL` | This repository's HTTPS URL | Set to your fork or configuration repository when customizing patches |
| `spec.source.targetRevision` | Full Git commit SHA | Pin the reviewed commit containing your configuration |
| `spec.source.path` | Operator or instance overlay | Select the desired release and instance configuration |
| `spec.destination.server` | `https://kubernetes.default.svc` | Set to a registered destination cluster if deploying remotely |
| `spec.destination.namespace` | `ansible-automation-platform` | Coordinate with namespace fields and operator watch scope in the source manifests |

The example revision `53b286e28d25c03ed9a9445019ede26d857044dd` is an existing
merged commit with both referenced overlays. Its custom-hostname patch contains
`ansible.qa.example.com`, so **customize and commit the patch before deployment**.
Changing a local file does not change what Argo CD renders from a pinned revision.
Update `repoURL` and `targetRevision` to reference the committed changes in Git.
See the [Application specification](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/).

The selected AppProject must permit the source repository, destination and these
resource kinds. Separately, Argo CD's destination credentials/controller service
account need Kubernetes permissions to manage the cluster-scoped Namespace and
the namespaced OperatorGroup, Subscription and AnsibleAutomationPlatform. Follow
your GitOps installation's namespace-management and RBAC configuration. Project
permissions alone do not grant Kubernetes access.

The operator Application owns namespace creation through its source manifest;
the instance Application uses that namespace. Changing only the Application's
destination namespace does not rewrite explicit namespaces in source manifests.
Review existing installations before adopting resources already managed elsewhere.

## 1. Prepare the source configuration

Edit the [instance platform patch](../../instances/ansible-automation-platform/overlays/2.7-custom-hostname/platform-patch.yaml):
set `hostname` and `route_host` to the intended FQDN and `route_tls_secret` to the
manually provisioned Secret's name. Review storage sizing/classes and enabled
components in the [instance guide](../../instances/ansible-automation-platform/README.md).
Commit and push those changes, then update the Application source fields above.

For the operator-generated hostname and default ingress certificate instead,
set the instance Application's source path to
`instances/ansible-automation-platform/overlays/2.7`. That option needs no custom
TLS Secret. Choose one instance overlay for the `aap` resource.

For another operator patch release, change the operator Application's source path
to the appropriate version overlay. Review catalog availability and OLM pinning
in the [repository guide](../../README.md#how-pinning-works).

## 2. Sync the operator

After editing the Application fields:

```sh
oc apply --dry-run=server -f examples/argocd/operator-application.yaml
oc apply -f examples/argocd/operator-application.yaml
argocd app get aap-operator
argocd app diff aap-operator
argocd app sync aap-operator
```

The Argo CD UI's **Sync** action can be used instead of the CLI. These commands
assume the CLI is already connected to the Argo CD instance managing these
Applications. Applying the Application object alone does not start a manual sync.

The Subscription has `installPlanApproval: Manual`. Inspect its InstallPlan,
verify the expected CSV and approve that exact plan using the
[InstallPlan procedure](../../README.md#review-and-approve-the-initial-installplan).
For the included `2.7/12.3` operator overlay, the expected CSV is
`aap-operator.v2.7.0-0.1794269743`. Then verify installation:

```sh
oc get subscription ansible-automation-platform -n ansible-automation-platform \
  -o jsonpath='{.status.installedCSV}{"\n"}'
oc get csv -n ansible-automation-platform
oc wait --for=condition=Established \
  crd/ansibleautomationplatforms.aap.ansible.com --timeout=120s
```

Wait for the expected CSV to show `Succeeded` before proceeding. An Argo CD
Application showing `Synced` means its manifests match Git; it is not proof that
OLM has finished installing the operator. Keep subsequent InstallPlans unapproved
until you intend to upgrade.

## 3. Prepare DNS and the TLS Secret

For the custom-hostname overlay, make the configured FQDN resolve to OpenShift
ingress. Obtain a certificate covering that hostname and a matching private key.
Create the Secret in the AAP namespace using the name in `route_tls_secret`:

```sh
oc create secret tls aap-route-tls \
  -n ansible-automation-platform \
  --cert=/secure/path/fullchain.pem \
  --key=/secure/path/tls.key
```

An appropriate existing Secret can be reused. Keep private keys and Secret
payloads outside Git. The Applications do not create or renew this Secret.
TLS terminates at the router in this example; router-to-gateway traffic is HTTP.
See the [manual TLS instructions](../../instances/ansible-automation-platform/README.md#create-the-tls-secret-manually)
for certificate requirements and replacement.

## 4. Sync and verify the instance

Once the operator is ready and the Secret exists:

```sh
oc apply --dry-run=server -f examples/argocd/instance-application.yaml
oc apply -f examples/argocd/instance-application.yaml
argocd app get aap-instance
argocd app diff aap-instance
argocd app sync aap-instance

oc get ansibleautomationplatform aap -n ansible-automation-platform \
  -o jsonpath='{.status.conditions}{"\n"}'
oc get pods,pvc,route -n ansible-automation-platform
```

Allow the operator to reconcile and complete the
[DNS, TLS and application checks](../../instances/ansible-automation-platform/README.md#verify-the-result).
Application-CR server dry-run only validates the Application resource; it does
not fetch Git, render the source or check the controller's destination permissions.

## Subsequent changes and lifecycle

Commit configuration changes to the source repository, update the relevant
Application's `targetRevision`, reapply the Application and review its diff before
syncing. Manage the AAP CR through this Application after adoption so direct
`oc apply -k` changes do not compete with Git. For certificate replacement,
manually create the replacement Secret, commit its reference in the instance
patch, update the instance Application revision, sync and verify the served cert.

Automated sync/pruning and cascading-deletion finalizers are omitted. Removing
an Application object with `oc delete` therefore leaves its managed resources in
place; deliberate teardown is a separate operation. Pruning the operator's
Namespace or deleting the AAP CR can remove the running platform and its data.
Do not use Git revision rollback as an operator downgrade or database restore;
follow the supported platform recovery process for those operations.

## Validation

Checked November 23, 2026: both Application YAML files passed strict server
dry-run against an installed Argo CD CRD in `openshift-gitops`. The exact pinned
Git source paths rendered the expected resource kinds with `oc kustomize` and
matched the current deployment manifests. YAML parsing, local documentation
links and all four deployment overlay renders passed.

No Applications were persisted or synced. Git access from the Argo CD server,
AppProject/controller permissions, OLM installation and platform provisioning
remain deployment-time checks.
