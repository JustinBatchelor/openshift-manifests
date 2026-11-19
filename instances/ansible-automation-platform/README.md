# Ansible Automation Platform instance

Creates one `aap.ansible.com/v1alpha1` `AnsibleAutomationPlatform` named `aap-jbatchel`
in `ansible-automation-platform`, the namespace watched by this repository's
operator installation. It enables platform gateway, automation controller,
Automation Hub, and Event-Driven Ansible. Lightspeed and MCP are disabled.

The `overlays/2.7` overlay uses operator-managed PostgreSQL with a 20Gi database
volume request and 10Gi of shared Hub file storage. These are starting sizes for
a small lab, not a production capacity or availability design. Adjust sizes in
`overlays/2.7/platform-patch.yaml` before deploying.

This overlay works with either included AAP 2.7 operator patch release. The
operator Subscription selects the CSV; this instance does not independently pin
component versions or override the operator's bundled images.

## Testing instance name

The base resource and patch use the name `aap`. The testing overlay adds
`nameSuffix: -jbatchel` in `overlays/2.7/kustomization.yaml`, producing the final
name `aap-jbatchel`. Change that suffix to customize the testing instance name.
The namespace remains `ansible-automation-platform`.

Applying this name creates a separate instance if `aap` already exists; it does
not rename or migrate an existing deployment or its data.

## Default hostname and optional customization

Custom hostname and TLS fields are commented out in the base, so the operator
uses its default networking configuration. No custom TLS Secret is required.
Find the generated hostname with `oc get route -n ansible-automation-platform`.

To enable a custom hostname later, uncomment and edit `spec.hostname` in
[base/ansibleautomationplatform.yaml](base/ansibleautomationplatform.yaml):

```yaml
spec:
  hostname: aap.apps.your-cluster.example.com
```

Replace this placeholder with the DNS hostname for your platform gateway, without
`https://`, a port, or a URL path. The `overlays/2.7` Kustomization includes this
base, so the hostname is carried into the rendered `AnsibleAutomationPlatform`.
An environment-specific overlay can override `spec.hostname` in its platform
patch if needed.

The platform-level `hostname` field is also used in Red Hat's
[AAP on OpenShift example](https://cloud.redhat.com/experts/aro/aap/).

Ensure the hostname resolves to the OpenShift ingress endpoint. Setting this
value does not create DNS records or certificates.

Verify the field against the installed operator before applying:

```sh
oc explain ansibleautomationplatform.spec.hostname
```

## Optional custom gateway TLS

To enable custom gateway TLS, uncomment the five networking settings following
`hostname` in the base: `gateway_tls_enabled`, `gateway_tls_secret_name`,
`service_type`, `ingress_type`, and `route_tls_termination_mechanism`. They configure
a passthrough Route and reference `aap-gateway-tls`. Create that Secret in
`ansible-automation-platform` after the namespace exists and before applying the
custom TLS configuration:

```sh
oc create secret tls aap-gateway-tls \
  -n ansible-automation-platform \
  --cert=/secure/path/fullchain.pem \
  --key=/secure/path/tls.key
```

Use your certificate chain and matching private key; the certificate's SAN must
cover `spec.hostname`. This command creates the `tls.crt` and `tls.key` entries.
For an existing Secret, change `spec.gateway_tls_secret_name` in the base to its
name. Keep certificate/key files and Secret payloads outside Git.

Clients must use HTTPS. The gateway serves this certificate; the router passes
TLS through. After rotating the Secret, restart the gateway:

```sh
oc rollout restart deployment/aap-jbatchel-gateway -n ansible-automation-platform
```

Verify configuration after reconciliation:

```sh
oc get ansibleautomationplatform aap-jbatchel -n ansible-automation-platform \
  -o jsonpath='{.status.conditions[?(@.type=="ConfigurationValid")]}{"\n"}'
```

Expect `True`; a missing Secret can cause `False`. See
[Red Hat gateway TLS configuration](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/install-configure_your_ansible_automation_platform_deployment).

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
[repository README](../../README.md). This instance configuration does not
create another Namespace, Subscription, or OperatorGroup.

```sh
oc get csv -n ansible-automation-platform
oc wait --for=condition=Established \
  crd/ansibleautomationplatforms.aap.ansible.com --timeout=120s

# Render locally.
oc kustomize instances/ansible-automation-platform/overlays/2.7

# Validate against the installed CRD and admission policies, without persisting.
oc apply --dry-run=server \
  -k instances/ansible-automation-platform/overlays/2.7

# Create the instance after validation succeeds.
oc apply -k instances/ansible-automation-platform/overlays/2.7
```

The cluster must be able to pull the required Red Hat images and have sufficient
capacity for the selected components. The default configuration needs no custom
hostname or TLS Secret; inspect the generated route and certificate for your
environment. Follow the optional sections above only when enabling custom TLS.
No passwords, pull-secret payloads, or other credential values are stored here.
Keep credentials in cluster Secrets or your external secret-management system.

## Check the deployment

```sh
oc get ansibleautomationplatform aap-jbatchel -n ansible-automation-platform
oc get ansibleautomationplatform aap-jbatchel -n ansible-automation-platform \
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
