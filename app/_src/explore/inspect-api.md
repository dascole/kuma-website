---
title: Inspect API
---

Starting with version 1.5.0, {{site.mesh_product_name}} offers the Inspect API to improve the policy debugging experience.
It's made up of several HTTP endpoints and is fully supported by `kumactl`,
but can be used directly, using the [HTTP API](/docs/{{ page.release }}/reference/http-api/#inspect-api).

## Matched policies

Read [how {{site.mesh_product_name}} chooses the right policy to apply](/docs/{{ page.release }}/policies/how-kuma-chooses-the-right-policy-to-apply)
to understand how {{site.mesh_product_name}} matches policies to data plane proxies.
With so many policies, it's hard to understand which policies are selected for a specific data plane proxy.
That's where the Inspect API can help:

```shell
kumactl inspect dataplane backend-1 --mesh=default
```

{% if_version lte:2.5.x %}
```text
DATAPLANE:
  ProxyTemplate
    pt-1
  TrafficTrace
    backends-eu

INBOUND 127.0.0.1:10010:10011(backend):
  TrafficPermission
    allow-all-default

OUTBOUND 127.0.0.1:10006(gateway):
  Timeout
    timeout-all-default
  TrafficRoute
    route-all-default

SERVICE gateway:
  CircuitBreaker
    circuit-breaker-all-default
  HealthCheck
    gateway-to-backend
  Retry
    retry-all-default
```
{% endif_version %}
{% if_version gte:2.6.x %}
```text
DATAPLANE:
  MeshProxyPatch
    pt-1
  MeshTrace
    backends-eu

INBOUND 127.0.0.1:10010:10011(backend):
  MeshTrafficPermission
    allow-all

OUTBOUND 127.0.0.1:10006(gateway):
  MeshTimeout
    timeout-all-default

SERVICE gateway:
  MeshCircuitBreaker
    circuit-breaker-all-default
  MeshHealthCheck
    gateway-to-backend
  MeshRetry
    retry-all-default
```
{% endif_version %}

Each data plane proxy has 4 policy attachment points:

- Inbound – applied to envoy inbound listener
- Outbound – applied to envoy outbound listener
- Service – applied to envoy outbound cluster (upstream cluster)
- Dataplane – non-specific policy attachment, could affect inbound/outbound listeners and clusters

The command in the example above shows what policies were matched for each type of attachment.

## Affected data plane proxies

Sometimes it's useful to see if it's safe to delete or modify some policy. Before making any critical changes,
it is worth checking which data plane proxies will be affected. This can be done using the Inspect API as well:

```shell
kumactl inspect {% if_version lte:2.5.x inline:true %}traffic-permission{% endif_version %}{% if_version gte:2.6.x inline:true %}meshtrafficpermission{% endif_version %} tp1 --mesh=default
```

```text
Affected data plane proxies:

  backend-1:
    inbound 127.0.0.1:10010:10011(backend)
    inbound 127.0.0.1:20010:20011(backend-admin)
    inbound 127.0.0.1:30010:30011(backend-api)

  web-1:
    inbound 127.0.0.1:10020:10021(web)
```

This command works for all types of policies.

## Envoy proxy configuration

{{site.mesh_product_name}} has 3 components that build on top of envoy – kuma-dp, zone-ingress and zone-egress.
To help with debugging these components, the Inspect API gives access to envoy config dumps:

Get config dump for data plane proxy:

```shell
kumactl inspect dataplane backend-1 --type=config-dump
```

Get config dump for zone ingress:

```shell
kumactl inspect zoneingress zi-1 --type=config-dump
```

Get config dump for zone egress:

```shell
kumactl inspect zoneegress ze-1 --type=config-dump
```

{% warning %}
To retrieve a config dump in a Multizone deployment, `kumactl` should be pointed to a zone CP
Global CPs don't have access to envoy config dumps.
This is [a limitation that will be resolved in an upcoming release](https://github.com/kumahq/kuma/issues/3789).
{% endwarning %}
