# Using ExternalDNS for creating per-pod DNS record

This tutorial describes how to setup ExternalDNS for creating per-pod DNS records.

## Usecases

Sometimes a pod needs to be able to hand out a DNS records pointing to its own IP address, e.g. when a client needs to establish a direct TLS connection to a pod after an initial, load-balanced HTTP request (there needs to be a matching DNS record for the x509 certificate). One example is a WebRTC application where the client communicates directly with a pod via DTLS (UDP).

While this could be also achieved using `hostPort`s on a `StatefulSet`, this way allows to create per-pod DNS records for pods created by `Deployment`s, providing a less restrictive option that fits better for stateless applications.

# Usage

## Deployment

First, lets create a simple nginx deployment called my-nginx.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        port:
        - containerPort: 80
          name: http
```

Pod names will look like `my-nginx-2035384211-7ci7o`.

## Headless Service

Now we need to define a headless service (`ClusterIP: None`) with the correct annotations to enable the creation of per-pod DNS records.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: example.org
    external-dns.alpha.kubernetes.io/publish-pod-names: 'true'
spec:
  clusterIP: None
  selector:
    app: my-nginx
  ports:
  - port: 80
    name: http
```

This will create one DNS record per pod. They will look like: `my-nginx-2035384211-7ci7o.example.com` and point to the corresponding pod IP. This also works for pods with `hostNetwork: true` (records will point to the corresponding host IPs).
