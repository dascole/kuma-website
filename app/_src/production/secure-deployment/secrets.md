---
title: Manage secrets
content_type: how-to
---

The `Secret` resource enables users to store sensitive data.
Sensitive information is anything a user considers non-public, e.g.:

- TLS keys
- tokens
- passwords

Secrets belong to a specific [`Mesh`](/docs/{{ page.release }}/production/mesh/) resource, and cannot be shared across different `Meshes`.
[Policies](/docs/{{ page.release }}/policies/introduction) use secrets at runtime.

{% tip %}
{{site.mesh_product_name}} leverages `Secret` resources internally for certain operations,
for example when storing auto-generated certificates and keys when Mutual TLS is enabled.
{% endtip %}

{% tabs %}

{% tab Kubernetes %}

On Kubernetes, {{site.mesh_product_name}} under the hood leverages the native [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) resource to store sensitive information.

{{site.mesh_product_name}} secrets are stored in the same namespace as the Control Plane with `type` set to `system.kuma.io/secret`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sample-secret
  namespace: {{site.mesh_namespace}} # {{site.mesh_product_name}} will only manage secrets in the same namespace as the CP
  labels:
    kuma.io/mesh: default # specify the Mesh scope of the secret
data:
  value: dGVzdAo= # Base64 encoded
type: system.kuma.io/secret # {{site.mesh_product_name}} will only manage secrets of this type
```

Use `kubectl` to manage secrets like any other Kubernetes resource.

```sh
echo "apiVersion: v1
kind: Secret
metadata:
  name: sample-secret
  namespace: {{site.mesh_namespace}}
  labels:
    kuma.io/mesh: default
data:
  value: dGVzdAo=
type: system.kuma.io/secret" | kubectl apply -f -

kubectl get secrets -n {{site.mesh_namespace}} --field-selector='type=system.kuma.io/secret'
# NAME            TYPE                    DATA   AGE
# sample-secret   system.kuma.io/secret   1      3m12s
```

Kubernetes Secrets are identified with the `name + namespace` format,
therefore **it is not possible** to have a `Secret` with the same name in multiple meshes.
Multiple `Meshes` always belong to one {{site.mesh_product_name}} CP that always runs in one Namespace.

In order to reassign a `Secret` from one `Mesh` to another `Mesh` you need to delete the `Secret` resource and create it in another `Mesh`.

{% endtab %}

{% tab Universal %}

A `Secret` is a simple resource that stores specific `data`:

```yaml
type: Secret
name: sample-secret
mesh: default
data: dGVzdAo= # Base64 encoded
```

Use `kumactl` to manage any `Secret` the same way you would do for other resources:

```sh
echo "type: Secret
mesh: default
name: sample-secret
data: dGVzdAo=" | kumactl apply -f -
```

{% endtab %}
{% endtabs %}


The `data` field of a {{site.mesh_product_name}} `Secret` is a Base64 encoded value.
Use the `base64` command in Linux or macOS to encode any value in Base64:

```sh
# Base64 encode a file
cat cert.pem | base64

# or Base64 encode a string
echo "value" | base64
```


### Access to the Secret HTTP API

Secret API requires authentication.
Consult [Accessing Admin Server from a different machine](/docs/{{ page.release }}/production/secure-deployment/certificates/#user-to-control-plane-communication) for how to configure remote access.

## Scope of the Secret

{{site.mesh_product_name}} provides two types of Secrets.

### Mesh-scoped Secrets

Mesh-scoped Secrets are bound to a given Mesh.
Only this kind of Secrets can be used in Mesh Policies like [Provided CA](/docs/{{ page.release }}/policies/mutual-tls#usage-of-provided-ca) or TLS setting in [External Service](/docs/{{ page.release }}/policies/external-services).

{% tabs %}
{% tab Kubernetes %}

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sample-secret
  namespace: {{site.mesh_namespace}}
  labels:
    kuma.io/mesh: default # specify the Mesh scope of the secret
data:
  value: dGVzdAo=
type: system.kuma.io/secret
```

{% endtab %}
{% tab Universal %}

```yaml
type: Secret
name: sample-secret
mesh: default # specify the Mesh scope of the secret
data: dGVzdAo=
```

{% endtab %}
{% endtabs %}

### Global-scoped Secrets

Global-scoped Secrets are not bound to a given Mesh and cannot be used in Mesh Policies.
Global-scoped Secrets are used for internal purposes.
You can manage them just like the regular secrets using `kumactl` or `kubectl`.

{% tabs %}
{% tab Kubernetes %}
Notice that the `type` is different and `kuma.io/mesh` label is not present.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sample-secret
  namespace: {{site.mesh_namespace}}
data:
  value: dGVzdAo=
type: system.kuma.io/global-secret
```

{% endtab %}
{% tab Universal %}
Notice that the `type` is different and `mesh` field is not present.

```yaml
type: GlobalSecret
name: sample-global-secret
data: dGVzdAo=
```

{% endtab %}
{% endtabs %}

## Behaviour in multi-zone deployment

The Secrets are synced from global to zones, not the other way around as this would risk exposing sensitive information.
{% if_version lte:2.9.x %}
It's invalid to create Secrets on zone, if you do, the control plane deletes them when KDS sync runs.
{% endif_version %}
{% if_version gte:2.10.x %}
If there's a name conflict between a secret on global and on zone, then the secret is not overwritten and a warning will appear in the logs.
{% endif_version %}

## Usage

Here is an example of how you can use a {{site.mesh_product_name}} `Secret` with a `provided` [Mutual TLS](/docs/{{ page.release }}/policies/mutual-tls) backend.

The examples below assumes that the `Secret` object has already been created beforehand.

{% tabs %}
{% tab Universal %}

```yaml
type: Mesh
name: default
mtls:
  backends:
    - name: ca-1
      type: provided
      config:
        cert:
          secret: my-cert # name of the {{site.mesh_product_name}} Secret
        key:
          secret: my-key # name of the {{site.mesh_product_name}} Secret
```

{% endtab %}
{% tab Kubernetes %}

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    backends:
      - name: ca-1
        type: provided
        config:
          cert:
            secret: my-cert # name of the Kubernetes Secret
          key:
            secret: my-key # name of the Kubernetes Secret
```

{% endtab %}
{% endtabs %}
