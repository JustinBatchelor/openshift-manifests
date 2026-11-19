# Ansible Automation Platform instance

Creates one `aap.ansible.com/v1alpha1` `AnsibleAutomationPlatform` named `aap`
in `ansible-automation-platform`, the namespace watched by this repository's
operator installation. It enables platform gateway, automation controller,
Automation Hub, and Event-Driven Ansible. Lightspeed and MCP are disabled.

The `overlays/2.7` overlay uses operator-managed PostgreSQL with a 100Gi database
volume request and 10Gi of shared Hub file storage. These are starting sizes,
not a production capacity or availability design. Adjust sizes in
`overlays/2.7/platform-patch.yaml` before deploying.

This overlay works with either included AAP 2.7 operator patch release. The
operator Subscription selects the CSV; this instance does not independently pin
component versions or override the operator's bundled images.

## Default storage

Storage-class fields are deliberately omitted. The generated PVCs can use the
cluster's default StorageClass. Do not replace the omitted fields with empty
strings: an explicit `storageClassName: ""` on a PVC disables default-class
selection.

The default class must support the access modes requested by each component,
including **ReadWriteMany (RWX)** for Hub file storage and database storage
requirements. Having a default class does not prove that it supports RWX.
Check the provisioner's capabilities; many block-storage classes support only
ReadWriteOnce (RWO). If the default is RWO-only, set
`spec.hub.file_storage_storage_class` to a suitable RWX class in
`overlays/2.7/platform-patch.yaml`, or configure supported object storage for Hub.

```sh
oc get storageclass
```

Red Hat's [Hub reference](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/reference-automation_hub_custom_resources)
documents the RWX requirement. The upstream
[Hub/Galaxy PVC template](https://github.com/ansible/galaxy-operator/blob/main/roles/galaxy-content/templates/galaxy-file-storage.pvc.yaml.j2)
omits `storageClassName` when the Hub storage class is unspecified, allowing
[Kubernetes default provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/).
Confirm the actual PVCs use the expected class on your installed operator build.

## Deploy

Run from the repository root. First install the operator, approve its initial
InstallPlan, and confirm the selected CSV is `Succeeded` as described in the
[repository README](../../../README.md). This instance configuration does not
create another Namespace, Subscription, or OperatorGroup.

```sh
oc get csv -n ansible-automation-platform
oc wait --for=condition=Established \
  crd/ansibleautomationplatforms.aap.ansible.com --timeout=120s

# Render locally.
oc kustomize manifests/instances/ansible-automation-platform/overlays/2.7

# Validate against the installed CRD and admission policies, without persisting.
oc apply --dry-run=server \
  -k manifests/instances/ansible-automation-platform/overlays/2.7

# Create the instance after validation succeeds.
oc apply -k manifests/instances/ansible-automation-platform/overlays/2.7
```

The cluster must be able to pull the required Red Hat images and have sufficient
capacity for the selected components. This config uses the operator's route/TLS
defaults; inspect the resulting route and certificate for your environment.
No passwords, pull-secret payloads, or other credential values are stored here.
Keep credentials in cluster Secrets or your external secret-management system.

## Check the deployment

```sh
oc get ansibleautomationplatform aap -n ansible-automation-platform
oc get ansibleautomationplatform aap -n ansible-automation-platform \
  -o jsonpath='{.status.conditions}{"\n"}'
oc get pods,pvc,route -n ansible-automation-platform
```

Use the gateway route to access AAP. Obtain the operator-generated admin password
from the cluster Secret through the OpenShift console; do not export Secret
contents into this repository. Activate AAP with your subscription through the UI.

If storage claims remain Pending, inspect them and their events:

```sh
oc describe pvc -n ansible-automation-platform
oc get events -n ansible-automation-platform --sort-by=.lastTimestamp
```

This configuration was checked against the
[AAP 2.7 deployment example](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/install-proc_operator_link_components)
and [CR reference](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/reference-ansibleautomationplatform__aap_ansible_com_v1alpha1_).
Local Kustomize rendering was validated. Live CRD/admission validation and
provisioning could not be checked because the configured cluster hostname did not
resolve during preparation; run the server dry-run above on your target cluster.
