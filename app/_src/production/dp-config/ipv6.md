---
title: IPv6 support
content_type: how-to
---

All {{site.mesh_product_name}} entities do support running in mixed IPv4 and IPv6 environments as well as pure IPv6 setup. This includes
global and zone control planes, the data plane proxy, the accompanying iptables scripts and the CNI.

For the most part any IPv6 setup will work out of the box, but there are some specifics that need to be taken into account:

{% if_version gte:2.7.x %}
 * when data plane proxies are run in an IPv6-only environment (i.e. no IPv4 address), the DNS should be set to generate relevant
   IPv6 addresses using `KUMA_DNS_SERVER_CIDR`. Please make sure there is no overlap with a pre-existing network in your environment.

## Disabling IPv6

In some cases you might not want to use IPv6 at all.

{% tabs %}
{% tab Kubernetes %}
To turn it off for all workloads set either:
- config option `{{site.set_flag_values_prefix}}runtime.kubernetes.injector.sidecarContainer.ipFamilyMode=ipv4`
- the environment variable `KUMA_RUNTIME_KUBERNETES_INJECTOR_SIDECAR_CONTAINER_IP_FAMILY_MODE=ipv4`

To turn it off for a specific Pod, add the annotation `kuma.io/transparent-proxying-ip-family-mode: ipv4`.
{% endtab %}
{% tab Universal %}
In your Dataplane resource, set `networking.transparentProxying.ipFamilyMode=IPv4`.
{% endtab %}
{% endtabs %}
{% endif_version %}
{% if_version lte:2.6.x %}
 * when the data plane proxies are run in IPv6 only environment (i.e. no IPv4 address), the DNS shall be set to generate relevant
   IPv6 addresses using `KUMA_DNS_SERVER_CIDR`. Please make sure there is no overlap with a pre-existing network in your environment.

 * On Universal, when the Dataplane resource is generated, specify `networking.transparentProxying.redirectPortInboundV6`.
   The default value for that port is `15010`.

```yaml
type: Dataplane
mesh: default
name: {{ name }}
networking:
  address: {{ address }}
  inbound:
  - port: {{ port }}
    tags:
      kuma.io/service: demo-client
  transparentProxying:
    redirectPortInbound: 15006
    redirectPortInboundV6: 15010
    redirectPortOutbound: 15001
```

## Disabling IPv6

In some cases you might not want to use IPv6 at all.

{% tabs %}
{% tab Kubernetes %}
To turn it off for all workloads set either:
- `{{site.set_flag_values_prefix}}runtime.kubernetes.injector.sidecarContainer.redirectPortInboundV6` to 0
- the environment variable: `KUMA_RUNTIME_KUBERNETES_INJECTOR_SIDECAR_CONTAINER_REDIRECT_PORT_INBOUND_V6=0`

To turn it off for a specific pod add the annotation `kuma.io/transparent-proxying-inbound-v6-port: "0"`.
{% endtab %}
{% tab Universal %}
In your Dataplane resource don't set `networking.transparentProxying.redirectPortInboundV6`.
{% endtab %}
{% endtabs %}
{% endif_version %}
