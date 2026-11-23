# QA deployment with Argo CD

Deploy QA on its own OpenShift cluster using the cluster's local GitOps instance.
Controller, Hub and EDA are enabled. The AAP instance is named `aap` in
`ansible-automation-platform`, with a `100G` managed database request and `10Gi`
of Hub RWX file storage. PROD remains an empty placeholder with no Application.

| Application | Kustomize source path |
| --- | --- |
| [`aap-qa-operator`](operator-application.yaml) | `operators/ansible-automation-platform/overlays/2.7/12.3` |
| [`aap-qa-instance`](instance-application.yaml) | `instances/ansible-automation-platform/overlays/qa` |

Both examples pin source commit `2255d7c63b46715eae4938e572eb142476ba876c`, which
contains the operator and QA overlays. They use namespace `openshift-gitops`,
project `default` and destination `https://kubernetes.default.svc`. Adapt those
values for the QA GitOps installation. The Application names distinguish these
examples; they do not add a suffix to the AAP instance.

Use this pair instead of the generic Applications in the parent directory.
They manage the same Namespace, Subscription and AAP resource identities. Do not
sync both pairs to the same cluster. If adopting an existing Argo CD deployment,
update its source path/revision and retain its Application ownership rather than
creating a second Application for the same resources.

## Prepare the source and permissions

For a cluster without GitOps, first follow the
[GitOps operator bootstrap guide](../../../operators/openshift-gitops/README.md).
It creates the default Argo CD instance in `openshift-gitops`; configure private
repository access separately before creating these Applications.

Run commands from the repository root after selecting the QA OpenShift context
and logging the Argo CD CLI into QA's GitOps server. The selected AppProject and
controller must permit the repository, destination, Namespace, OperatorGroup,
Subscription and AAP CR. See the shared [permissions and source-field guide](../README.md#prerequisites-and-fields-to-change).

Edit the [QA platform patch](../../../instances/ansible-automation-platform/overlays/qa/platform-patch.yaml):
replace both hostname values with the QA FQDN, select the manual TLS Secret name,
and review the storage requests. Commit and push those changes. Then update
`repoURL` and `targetRevision` in the QA Applications to the repository and exact
commit containing that configuration. Local edits are not read by Argo CD.
The checked-in `ansible.qa.example.com` value is a placeholder to replace.

Render QA locally and review its configuration:

```sh
oc config current-context
oc kustomize instances/ansible-automation-platform/overlays/qa
```

The database request is 100 decimal GB, not 100Gi. Read the
[QA storage explanation](../../../instances/ansible-automation-platform/README.md#qa-settings)
for database limits, operating headroom, Hub RWX requirements and existing-volume
expansion considerations. The operator configures component database connections;
the example does not create separate database PVCs for each component.

## 1. Install and verify the operator

```sh
oc apply --dry-run=server -f examples/argocd/qa/operator-application.yaml
oc apply -f examples/argocd/qa/operator-application.yaml
argocd app get aap-qa-operator
argocd app diff aap-qa-operator
argocd app sync aap-qa-operator
```

Review and approve the exact initial OLM InstallPlan using the
[InstallPlan procedure](../../../README.md#review-and-approve-the-initial-installplan).
For the included release, verify CSV `aap-operator.v2.7.0-0.1794269743` reaches
`Succeeded`. Argo CD reporting `Synced` does not establish operator readiness.

```sh
oc get csv -n ansible-automation-platform
oc wait --for=condition=Established \
  crd/ansibleautomationplatforms.aap.ansible.com --timeout=120s
```

## 2. Prepare DNS and the TLS Secret

Point the QA FQDN to this cluster's ingress endpoint. Obtain a trusted certificate
covering the hostname and its matching private key. Create the Secret named by
QA's `route_tls_secret`, or reuse an appropriate existing Secret:

```sh
oc create secret tls aap-route-tls \
  -n ansible-automation-platform \
  --cert=/secure/path/fullchain.pem \
  --key=/secure/path/tls.key
```

Keep certificate/key files and Secret payloads outside Git. These manifests do
not issue certificates or create TLS Secrets. TLS terminates at the router;
router-to-gateway traffic is HTTP. See the [manual TLS procedure](../../../instances/ansible-automation-platform/README.md#create-the-tls-secret-manually).

## 3. Sync and verify QA

After the operator is ready and the TLS Secret exists:

```sh
oc apply --dry-run=server -k instances/ansible-automation-platform/overlays/qa
oc apply --dry-run=server -f examples/argocd/qa/instance-application.yaml
oc apply -f examples/argocd/qa/instance-application.yaml
argocd app get aap-qa-instance
argocd app diff aap-qa-instance
argocd app sync aap-qa-instance

oc get ansibleautomationplatform aap -n ansible-automation-platform \
  -o jsonpath='{.status.conditions}{"\n"}'
oc get automationcontroller,automationhub,eda -n ansible-automation-platform
oc get pods,pvc,route -n ansible-automation-platform
```

The local overlay dry-run must use the same source configuration pinned in the
Application. Check the generated component CRs and pods, PVC capacity/access modes,
Route host and certificate, DNS resolution, login, component access and logout.
Follow the [instance acceptance checks](../../../instances/ansible-automation-platform/README.md#verify-the-result).
Application-CR dry-run does not validate Git access or reconcile AAP workloads.

Both Applications use manual sync and omit automated pruning and cascading-delete
finalizers. For later configuration or certificate changes, commit the QA patch,
update the Application revision and sync. See the shared
[lifecycle guide](../README.md#subsequent-changes-and-lifecycle).

## Validation

Checked November 23, 2026: QA renders with Controller, Hub and EDA enabled,
the intended Route settings, a `100G` database request and `10Gi` Hub RWX storage.
All four existing deployment overlays still render identically to their prior
versions. Both QA Application source paths render at their exact pinned revision.
YAML, local documentation links and the empty PROD placeholder check passed.

QA passed strict server dry-run against the installed AAP CRD, retaining all
authored settings and accepting the CRD's additional defaults. The intended
namespace was absent, so that check used a temporary resource name in an existing
namespace. Both QA Application CRs passed strict server dry-run in
`openshift-gitops`. No resources were persisted or synced. These checks do not
validate target-namespace policies, GitOps permissions, storage provisioning,
DNS/TLS or full operator reconciliation in the QA cluster.
