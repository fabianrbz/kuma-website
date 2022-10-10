---
---
# Protocol support in Kuma

At its core, `Kuma` distinguishes between the following major categories of traffic: `http`, `grpc`, `kafka` and opaque `tcp` traffic.

For `http`, `grpc` and `kafka` traffic Kuma provides deep insights down to application-level transactions, in the latter `tcp` case the observability is limited to connection-level statistics.

So, as a user of Kuma, you're _highly encouraged_ to give it a hint whether your service supports `http` , `grpc`, `kafka` or not.

By doing this,

* you will get richer metrics with [`Traffic Metrics`](../traffic-metrics) policy
* you will get richer logs with [`Traffic Log`](../traffic-log) policy
* you will be able to use [`Traffic Trace`](../traffic-trace) policy

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
On `Kubernetes`, to give Kuma a hint that your service supports `HTTP` protocol, you need to add a `<port>.service.kuma.io/protocol` annotation to the `k8s` `Service` object.

E.g.,

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: kuma-example
  annotations:
    8080.service.kuma.io/protocol: http # let Kuma know that your service supports HTTP protocol
spec:
  selector:
    app: web
  ports:
  - port: 8080
```

:::
::: tab "Universal"
On `Universal`, to give Kuma a hint that your service supports the `http` protocol, you need to add a `kuma.io/protocol` tag to the `inbound` interface of your `Dataplane`.

E.g.,

```yaml
type: Dataplane
mesh: default
name: web
networking:
  address: 192.168.0.1 
  inbound:
  - port: 80
    servicePort: 8080
    tags:
      kuma.io/service: web
      kuma.io/protocol: http # let Kuma know that your service supports HTTP protocol
```
:::
::::

## HTTP/2 support

Kuma by default upgrades connection between Dataplanes to HTTP/2. If you want to enable HTTP/2 on connections between a dataplane and an application, use `kuma.io/protocol: http2` tag.


## TLS support

Whenever a service already initiates a TLS request to another service - and [mutual TLS](../mutual-tls) is enabled - Kuma can enforce both TLS connections end-to-end as long as the service that is generating the TLS traffic is explicitly tagged with `tcp` [protocol](../protocol-support-in-kuma) (ie: `kuma.io/protocol: tcp`).

:::tip
Effectively `kuma-dp` will send the raw original TLS request as-is to the final destination, while in the meanwhile it will be enforcing its own TLS connection (if [mutual TLS](../mutual-tls) is enabled). Hence, the traffic must be marked as being `tcp`, so `kuma-dp` won't try to parse it.
:::

Note that in this case no advanced HTTP or GRPC statistics or logging are available. As a best practice - since Kuma will already secure the traffic across services via the [mutual TLS](../mutual-tls) policy - we suggest disabling TLS in the original services in order to get L7 metrics and capabilities.

## Websocket support

Kuma out of the box support's `Websocket` protocol. The service exposing `Websocket` should be annotated with `kuma.io/protocol: tcp` annotation

As `Websockets` use pure `TCP` connections under the hood, your service have to be recognised by `Kuma` as the `TCP` one. It's also the default behavior for Kuma to assume the service's `inbound` interfaces are the TCP ones, so you don't have to do anything, but if you want to be explicit, you can annotate your services exposing `Websocket` endpoints with `kuma.io/protocol: tcp` annotation. I.e.:

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
```yaml
apiVersion: v1
kind: Service
metadata:
  name: websocket-server
  namespace: kuma-example
  annotations:
    8080.service.kuma.io/protocol: tcp
spec:
  selector:
    app: websocket-server
  ports:
  - port: 8080
```

:::
::: tab "Universal"
```yaml
type: Dataplane
mesh: default
name: websocket-server
networking:
  address: 192.168.0.1 
  inbound:
  - port: 80
    servicePort: 8080
    tags:
      kuma.io/service: websocket-server
      kuma.io/protocol: tcp
```
:::
::::