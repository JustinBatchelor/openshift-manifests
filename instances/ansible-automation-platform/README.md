# Ansible Automation Platform instance

Creates one `aap.ansible.com/v1alpha1` `AnsibleAutomationPlatform` named **`aap`**
in `ansible-automation-platform`, the namespace watched by this repository's
operator installation. It enables platform gateway, automation controller,
Automation Hub, and Event-Driven Ansible. Lightspeed and MCP are disabled.

All commands below run from the repository root. Install the operator and approve
its initial InstallPlan using the [repository README](../../README.md) first.
These instance overlays do not create a Namespace, Subscription or OperatorGroup.
They inherit the operator's component versions; the Subscription selects the CSV.

## Choose an overlay

| Overlay under `instances/ansible-automation-platform/overlays/` | Networking | Certificate source |
| --- | --- | --- |
| `2.7` | Operator-generated hostname and default TLS settings | Cluster's default ingress certificate |
| `2.7-custom-hostname` | Custom OpenShift Route with edge TLS | TLS Secret created manually from PEM files |
| `2.7-custom-hostname-cert-manager` | Same custom Route | Optional cert-manager Certificate using an existing issuer |

Choose **one** overlay for the instance. They all render the same resource name
and namespace. The default option does not require cert-manager or a custom Secret.
Do not apply the individual base, patch or certificate files directly; Kustomize
combines them into complete resources.

## Instance name and existing deployments

The previous `nameSuffix: -jbatchel` has been removed. The base and all patches now
render `aap`, without a suffix. Check existing instances before applying:

```sh
oc get ansibleautomationplatform -n ansible-automation-platform
```

If `aap-jbatchel` already exists, applying `aap` creates a different instance; it
does not rename the old one or migrate data. Preserve the old deployment until
you have an agreed backup, migration and cutover procedure. An existing `aap` in
this namespace will instead be updated. An `aap` in a different namespace is a
separate resource. This repository change does not change any live instance.

## Customize through patches

Keep reusable component defaults in `base/`. Edit these files for the environment:

| File | Settings |
| --- | --- |
| [`overlays/2.7/platform-patch.yaml`](overlays/2.7/platform-patch.yaml) | Database size, Hub size; add approved storage classes or component overrides here |
| [`overlays/2.7-custom-hostname/platform-patch.yaml`](overlays/2.7-custom-hostname/platform-patch.yaml) | `hostname`, `route_host`, `route_tls_secret`, ingress and TLS mode |
| [`overlays/2.7-custom-hostname-cert-manager/certificate-patch.yaml`](overlays/2.7-custom-hostname-cert-manager/certificate-patch.yaml) | Certificate `issuerRef` and optional issuance settings |

The custom-hostname overlays inherit the `2.7` storage patch. The cert-manager
overlay additionally inherits the custom-hostname patch, then copies the rendered
`route_host` into `Certificate.spec.dnsNames[0]` and `route_tls_secret` into
`Certificate.spec.secretName`. Change the hostname and Secret reference in the
AAP patch; the certificate values follow automatically when rendered.
For independent environments, copy the overlays and update their relative
resource paths so they inherit the intended environment's patches.

### Hostname and TLS settings

The custom-hostname patch contains these generic values:

```yaml
spec:
  hostname: ansible.qa.example.com
  route_host: ansible.qa.example.com
  service_type: ClusterIP
  ingress_type: Route
  route_tls_termination_mechanism: Edge
  gateway_tls_enabled: false
  route_tls_secret: aap-route-tls
```

Replace **both** hostname values with the customer's FQDN, without `https://`, a
port or a path. This repository uses a reserved example domain; replace it
before issuance or deployment.

With `ingress_type: Route`, **`route_host` controls the OpenShift Route host**.
In the tested AAP 2.7 operator, `hostname` alone did not change the generated Route.
Keep them aligned. `public_base_url` is deliberately omitted so the operator can
derive the gateway URL from the admitted Route. Check for an old override when
modifying an existing instance.

This example terminates TLS at the OpenShift router; router-to-gateway traffic is
HTTP. The certificate comes from `route_tls_secret`, in the AAP namespace. It does
not use the passthrough/gateway certificate field `gateway_tls_secret_name`.
If the customer requires encrypted backend traffic, select and validate a supported
reencrypt or passthrough configuration separately; changing only the termination
field is insufficient. See the [Red Hat deployment configuration reference](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/install-configure_your_ansible_automation_platform_deployment).

### DNS is a separate requirement

Create an A/AAAA record to the **OpenShift ingress endpoint**, or a CNAME to the
appropriate ingress load-balancer hostname. Do not use a pod, ClusterIP Service
or API-server address. Internal clients may need an internal DNS override;
external clients need externally reachable ingress and appropriate DNS.

The AAP hostname fields do not create DNS records. Certificate issuance also does
not create the application's DNS record. All intended clients must resolve the
custom hostname and reach ingress TCP/443. Check from an actual client:

```sh
dig +short ansible.qa.example.com
```

## Primary option: manually supplied certificate

Obtain a certificate whose SAN covers the chosen hostname, its intermediate
certificate chain, and the matching private key. Clients must trust the issuing
CA. Create the TLS Secret in the operator's namespace **before** applying the
custom-hostname overlay:

```sh
oc create secret tls aap-route-tls \
  -n ansible-automation-platform \
  --cert=/secure/path/fullchain.pem \
  --key=/secure/path/tls.key
```

Use the Secret name from `spec.route_tls_secret` if you changed it. An existing
appropriate `kubernetes.io/tls` Secret can be reused. Keep PEM files, keys and
Secret payloads outside Git. This path requires no cert-manager installation.

Then use the render, dry-run and apply sequence below with
`2.7-custom-hostname` selected.

## Optional certificate issuance with cert-manager

Use this option only if cert-manager and its `Certificate` CRD are installed.
Configure an existing Ready issuer authorized to issue for the customer's domain
in `certificate-patch.yaml`. The included `customer-production-issuer` is a
placeholder, not an issuer created by this repository. Use `kind: Issuer` for a
namespaced issuer in `ansible-automation-platform`, or `ClusterIssuer` for a
cluster-scoped issuer. Configure the issuer's CA/DNS challenge credentials through
the customer's secret-management process.

Use a CA trusted by clients; ACME staging certificates are not browser-trusted.
Do not manually create or rotate a Secret that cert-manager manages. The
[cert-manager Certificate documentation](https://cert-manager.io/docs/usage/certificate/)
describes issuance, renewal and the resulting Secret.

The Certificate has a dedicated label so it can be issued before creating AAP:

```sh
AAP_OVERLAY=instances/ansible-automation-platform/overlays/2.7-custom-hostname-cert-manager
oc kustomize "$AAP_OVERLAY"
oc apply --dry-run=server -k "$AAP_OVERLAY"

# Apply only the Certificate, with hostname and Secret replacements resolved.
oc apply -k "$AAP_OVERLAY" -l app.kubernetes.io/component=route-certificate
oc wait certificate/aap-route -n ansible-automation-platform \
  --for=condition=Ready --timeout=300s

# After issuance succeeds, apply the complete overlay, including AAP.
oc apply -k "$AAP_OVERLAY"
```

If issuance fails, describe the Certificate and inspect issuer/challenge status:

```sh
oc describe certificate aap-route -n ansible-automation-platform
```

The issuer must be able to validate the domain or sign the request; a DNS override alone does not
satisfy a public CA's ownership validation.

## Storage and component settings

The `2.7` patch requests a 20Gi PostgreSQL volume and 10Gi of shared Hub file
storage. These are small-lab starting sizes, not a production capacity or
availability design. Adjust them in `overlays/2.7/platform-patch.yaml`.

Storage-class fields are omitted so generated PVCs use the cluster's default
StorageClass. Do not replace omissions with empty strings: an explicit
`storageClassName: ""` on a PVC disables default-class selection. Hub file storage
requires **ReadWriteMany (RWX)**. A default block-storage class supporting only
ReadWriteOnce is insufficient for Hub. Set `spec.hub.file_storage_storage_class`
to an approved RWX class in the platform patch, or configure supported object
storage. Verify database storage capabilities and available capacity too.

```sh
oc get storageclass
```

See the [Hub custom resource reference](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/reference-automation_hub_custom_resources).
If changing the namespace, update the operator namespace/watch scope and every
instance, Certificate and patch namespace together.

## Render, validate and deploy

Confirm the operator CSV is `Succeeded`, image pulls are authorized, and the
cluster has capacity and suitable storage. Select one overlay below; for the
default generated hostname, change the value to the `2.7` path instead.

```sh
oc config current-context
oc get csv -n ansible-automation-platform
oc wait --for=condition=Established \
  crd/ansibleautomationplatforms.aap.ansible.com --timeout=120s

AAP_OVERLAY=instances/ansible-automation-platform/overlays/2.7-custom-hostname
oc kustomize "$AAP_OVERLAY"
oc apply --dry-run=server -k "$AAP_OVERLAY"
oc apply -k "$AAP_OVERLAY"
```

Server dry-run validates API schema/admission, not DNS, Secret contents, issuer
readiness, storage provisioning or operator reconciliation. Existing-instance
changes also require reviewing old networking fields, component URLs, OAuth/SSO
redirect URLs and cutover impacts. Do not manage the same CR from competing
GitOps sources or apply multiple overlays in succession.

## Verify the result

```sh
oc get ansibleautomationplatform aap -n ansible-automation-platform \
  -o jsonpath='{.status.conditions}{"\n"}'
oc get pods,pvc,route -n ansible-automation-platform
oc get route -n ansible-automation-platform \
  -o custom-columns=NAME:.metadata.name,HOST:.spec.host,TERMINATION:.spec.tls.termination

# Use the customer hostname and client trust store; do not disable TLS checking.
curl -I https://ansible.qa.example.com/
```

Inspect the gateway Route and confirm its host matches the patch, it is admitted,
and it presents the intended certificate. In a browser, verify the certificate
SAN/chain, login, authenticated gateway/controller access, and logout. Confirm
redirects remain on the custom hostname. Obtain the generated admin password
through the OpenShift console and activate AAP with the customer's subscription.

A forced-resolution check such as `curl --resolve` can isolate routing and TLS,
but it is **not** proof of working DNS. Acceptance includes ordinary DNS lookup
and HTTPS access from the intended client network. For Pending claims, inspect
PVC events and the selected storage class.

## Certificate replacement and renewal

For manually supplied certificates, create a new versioned TLS Secret, change
`route_tls_secret` in `platform-patch.yaml`, and reapply the selected overlay.
After reconciliation, verify the served certificate before retiring the old
Secret. Keep the previous Secret available until rollback is no longer needed.

cert-manager renews its Secret, but do not assume the AAP operator immediately
copies renewed material into the Route. In the tested build, a Secret renewal
needed an explicit spec change to refresh the Route. Add or update this inert
marker in the custom-hostname platform patch, preserving any other annotations:

```yaml
spec:
  route_annotations: |
    example.com/certificate-refresh: "renewal-002"
```

`route_annotations` is a **string containing a YAML mapping** on the tested AAP
2.7 CRD. Change the marker for each refresh and reapply the selected overlay.
An arbitrary annotation on the AAP CR metadata did not trigger the same refresh.
Compare the certificate serial/fingerprint in the Secret with the one served to
clients. Cert-manager automation alone was not validated as unattended
end-to-end Route certificate rotation.

The [customer walkthrough](https://github.com/JustinBatchelor/aap-custom-hostname)
contains detailed TLS, DNS, rotation and rollback procedures, with
[validation notes](https://github.com/JustinBatchelor/aap-custom-hostname/blob/main/docs/validation-notes.md)
for the original custom-hostname test. On a new installation, validate the fields
against the installed operator's CRD and repeat the acceptance checks above.

## Manifest validation

Checked November 21, 2026 (Eastern), using `oc` 4.22.8 and Kustomize 5.7.1:
all instance and operator overlays render, the default instance retains its
component/storage configuration with the suffix removed, and edits to the
hostname, TLS Secret and issuer patches propagate into the certificate as intended.
All three instance overlays passed strict server dry-run against installed AAP 2.7
and cert-manager CRDs. The certificate-only apply selector excluded AAP.

The target namespace was absent in the validation cluster, so server dry-runs
used temporary resource names in an existing namespace. This checked schema and
admission there, not the target namespace's policies or a new deployment. No
resources were persisted. Repeat the dry-run in the customer's namespace, then
verify DNS, TLS and application behavior after deployment.
