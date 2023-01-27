---
title: Mesh HTTP Route (beta)
---

{% warning %}
This policy uses new policy matching algorithm and is in beta state,
it should not be mixed with [TrafficRoute](../traffic-route).
{% endwarning %}

The `MeshHTTPRoute` policy allows altering and redirecting HTTP requests
depending on where the request coming from and where it's going to.

## TargetRef support matrix

| TargetRef type    | top level | to  | from |
| ----------------- | --------- | --- | ---- |
| Mesh              | ✅        | ❌  | ❌   |
| MeshSubset        | ✅        | ❌  | ❌   |
| MeshService       | ✅        | ✅  | ❌   |
| MeshServiceSubset | ✅        | ❌  | ❌   |
| MeshGatewayRoute  | ❌        | ❌  | ❌   |

If you don't understand this table you should read [matching docs](/docs/{{ page.version }}/policies/targetref).

## Configuration

Unlike others outbound policies `MeshHTTPRoute` doesn't contain `default` directly in the `to` array. 
The `default` section is nested inside `rules`, so the policy structure looks like this:

```yaml
spec:
  targetRef: # top-level targetRef selects a group of proxies to configure
    kind: Mesh|MeshSubset|MeshService|MeshServiceSubset 
  to:
    - targetRef: # targetRef selects a destination (outbound listener)
        kind: MeshService
        name: backend
      rules:
        - matches: [...] # various ways to match an HTTP request (path, method, query)
          default: # configuration applied for the matched HTTP request
            filters: [...]
            backendRefs: [...]
```

### Matches

- **`path`** - (optional) - HTTP path to match the request on
  - **`type`** - one of `Exact`, `Prefix`, `RegularExpression`
  - **`value`** - actual value that's going to be matched depending on the `type`
- **`method`** - (optional) - HTTP2 method, available values are 
  `CONNECT`, `DELETE`, `GET`, `HEAD`, `OPTIONS`, `PATCH`, `POST`, `PUT`, `TRACE`
- **`queryParams`** - (optional) - list of HTTP URL query parameters. Multiple matches are ANDed together 
  such that all listed matches must succeed
  - **`type`** - one of `Exact` or `RegularExpression`
  - **`name`** - name of the query parameter
  - **`value`** - actual value that's going to be matched depending on the `type`

### Default conf

- **`filters`** - (optional) - a list of modifications applied to the matched request
  - **`type`** - available values are `RequestHeaderModifier`, `ResponseHeaderModifier`,
    `RequestRedirect`, `URLRewrite`.
  - **`requestHeaderModifier`** - [HeaderModifier](#headermodifier), must be set if the `type` is `RequestHeaderModifier`.
  - **`responseHeaderModifier`** - [HeaderModifier](#headermodifier), must be set if the `type` is `ResponseHeaderModifier`. 
  - **`requestRedirect`** - must be set if the `type` is `RequestRedirect`
    - **`scheme`** - one of `http` or `http2`
    - **`hostname`** - is the fully qualified domain name of a network host. This
      matches the RFC 1123 definition of a hostname with 1 notable exception that
      numeric IP addresses are not allowed.
    - **`port`** - is the port to be used in the value of the `Location` header in 
      the response. When empty, port (if specified) of the request is used.
    - **`statusCode`** - is the HTTP status code to be used in response. Available values are
      `301`, `302`, `303`, `307`, `308`.
  - **`urlRewrite`** - must be set if the `type` is `URLRewrite`
    - **`hostname`** - (optional) - is the fully qualified domain name of a network host. This
      matches the RFC 1123 definition of a hostname with 1 notable exception that
      numeric IP addresses are not allowed.
    - **`path`** - (optional)
      - **`type`** - one of `ReplaceFullPath`, `ReplacePrefixMatch`
      - **`replaceFullPath`** - must be set if the `type` is `ReplaceFullPath`
      - **`replacePrefixMatch`** - must be set if the `type` is `ReplacePrefixMatch`
- **`backendRefs`** - (optional) - list of destination for request to be redirected to
  - **`kind`** - one of `MeshService`, `MeshServiceSubset`
  - **`name`** - service name
  - **`tags`** - service tags, must be specified if the `kind` is `MeshServiceSubset`
  - **`weight`** - when a request matches the route, the choice of an upstream cluster 
    is determined by its weight. Total weight is a sum of all weights in `backendRefs` list.

### HeaderModifier

- **`set`** - (optional) - list of headers to set. Overrides value if the header exists.
  - **`name`** - header's name
  - **`value`** - header's value
- **`add`** - (optional) - list of headers to add. Appends value if the header exists.
  - **`name`** - header's name
  - **`value`** - header's value
- **`remove`** - (optional) - list of headers' names to remove

## Examples

### Traffic split

We can use `MeshHTTPRoute` to split an HTTP traffic between services with different tags 
implementing A/B testing or canary deployments.

Here is an example of a `MeshHTTPRoute` that splits the traffic from 
`frontend_kuma-demo_svc_8080` to `backend_kuma-demo_svc_3001` between versions, 
but only on endpoints starting with `/api`. All other endpoints will go to version: `1.0`.

{% tabs split useUrlFragment=false %}
{% tab split Kubernetes %}

```yaml
apiVersion: kuma.io/v1alpha1
kind: MeshHTTPRoute
metadata:
  name: http-route-1
  namespace: {{site.mesh_namespace}}
  labels:
    kuma.io/mesh: default
spec:
  targetRef:
    kind: MeshService
    name: frontend_kuma-demo_svc_8080
  to:
    - targetRef:
        kind: MeshService
        name: backend_kuma-demo_svc_3001
      rules:
        - matches:
            - path:
                type: Prefix
                value: /api
          default:
            backendRefs:
              - kind: MeshServiceSubset
                name: backend_kuma-demo_svc_3001
                tags:
                  version: "1.0"
                weight: 90
              - kind: MeshServiceSubset
                name: backend_kuma-demo_svc_3001
                tags:
                  version: "2.0"
                weight: 10
```
We will apply the configuration with `kubectl apply -f [..]`.
{% endtab %}

{% tab split Universal %}

```yaml
type: MeshHTTPRoute
name: http-route-1
mesh: default
spec:
  targetRef:
    kind: MeshService
    name: frontend_kuma-demo_svc_8080
  to:
    - targetRef:
        kind: MeshService
        name: backend_kuma-demo_svc_3001
      rules:
        - matches:
            - path:
                type: Prefix
                value: /api
          default:
            backendRefs:
              - kind: MeshServiceSubset
                name: backend_kuma-demo_svc_3001
                tags:
                  version: "1.0"
                weight: 90
              - kind: MeshServiceSubset
                name: backend_kuma-demo_svc_3001
                tags:
                  version: "2.0"
                weight: 10
```
We will apply the configuration with `kumactl apply -f [..]` or via the [HTTP API](/docs/{{ page.version }}/reference/http-api).
{% endtab %}
{% endtabs %}

### Traffic modifications

We can use `MeshHTTPRoute` to modify outgoing requests, by setting new path 
or changing request and response headers.

Here is an example of a `MeshHTTPRoute` that adds `x-custom-header` with value `xyz` 
when `frontend_kuma-demo_svc_8080` tries to consume `backend_kuma-demo_svc_3001`.

{% tabs modifications useUrlFragment=false %}
{% tab modifications Kubernetes %}

```yaml
apiVersion: kuma.io/v1alpha1
kind: MeshHTTPRoute
metadata:
  name: http-route-1
  namespace: {{site.mesh_namespace}}
  labels:
    kuma.io/mesh: default
spec:
  targetRef:
    kind: MeshService
    name: frontend_kuma-demo_svc_8080
  to:
    - targetRef:
        kind: MeshService
        name: backend_kuma-demo_svc_3001
      rules:
        - matches:
            - path:
                type: Exact
                value: /
          default:
            filters:
              - type: RequestHeaderModifier
                requestHeaderModifier:
                  set:
                    - name: x-custom-header
                      value: xyz
```
We will apply the configuration with `kubectl apply -f [..]`.
{% endtab %}

{% tab modifications Universal %}

```yaml
type: MeshHTTPRoute
name: http-route-1
mesh: default
spec:
  targetRef:
    kind: MeshService
    name: frontend_kuma-demo_svc_8080
  to:
    - targetRef:
        kind: MeshService
        name: backend_kuma-demo_svc_3001
      rules:
        - matches:
            - path:
                type: Exact
                value: /
          default:
            filters:
              - type: RequestHeaderModifier
                requestHeaderModifier:
                  set:
                    - name: x-custom-header
                      value: xyz
```
We will apply the configuration with `kumactl apply -f [..]` or via the [HTTP API](/docs/{{ page.version }}/reference/http-api).
{% endtab %}
{% endtabs %}

## Merging

When several `MeshHTTPRoute` policies target the same data plane proxy they're merged.
Similar to the new policies the merging order is determined by
[the top level targetRef](/docs/{{ page.version }}/policies/targetref#merging-configuration).
The difference is in `spec.to[].rules`.
{{site.mesh_product_name}} treats `rules` as a key-value map
where `matches` is a key and `default` is a value. For example MeshHTTPRoute policies:

```yaml
# MeshHTTPRoute-1
rules:
  - matches: # key-1
      - path:
          type: Exact
          name: /orders
        method: GET
    default: CONF_1 # value
  - matches: # key-2
      - path:
          type: Exact
          name: /payments
        method: POST
    default: CONF_2 # value
---
# MeshHTTPRoute-2
rules:
  - matches: # key-3
      - path:
          type: Exact
          name: /orders
        method: GET
    default: CONF_3 # value
  - matches: # key-4
      - path:
          type: Exact
          name: /payments
        method: POST
    default: CONF_4 # value
```

merged in the following list of rules:

```yaml
rules:
  - matches: 
      - path:
          type: Exact
          name: /orders
        method: GET
    default: merge(CONF_1, CONF_3) # because 'key-1' == 'key-3'
  - matches:
      - path:
          type: Exact
          name: /payments
        method: POST
    default: merge(CONF_2, CONF_4) # because 'key-2' == 'key-4'
```