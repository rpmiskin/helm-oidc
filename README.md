# Using Helm to Deploy OIDC Auth

Investigations into using k8s OIDC for service account
auth. Using Helm to deploy for even greater learning.

## Creating the charts

For initial investigation I created a main chart with a few subcharts.

```
helm create main-chart
cd main-chart/charts
helm create sub1
helm create sub2
helm create sub3
```

These can be installed into a release called `tst` as follows:

```
helm upgrade --install tst ./main-chart
```

## Setting up an Ingress Controller

To be able to access these externally we need an ingress, and ingresses
use a controller. For this I used the [nginx-ingress-controller](https://kubernetes.github.io/ingress-nginx/)
mainly because it is more or less the default one. (NB: The is also a version
from available directly from nginx, this has different configuration and
is not directly compatible).

I installed the ingress controller using Helm as detailed in the quick
start guide [here](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)

```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

This creates a single ingress controller for use across the cluster.

## Creating certificates

For local development I have set up a local DNS server so that `*.localhost`
will point to my development environment both from the host and also from
within the k8s environment. Instructions on how to do this can be found
in Github [here](https://github.com/rpmiskin/localhost-dns).

I have also configured a tool called `mkcert` to easily create certificates
which my local environment trusts. The basic flow for enable `mkcert`,
creating creating new certificates and adding them as k8s secrets
is as follows:

```
mkcert -install
mkcert test.localhost
kubectl create secret tls \
        test.localhost
        --key test.localhost-key.pem \
        --cert test.localhost.pem
```

(The secret name doesn't have to match the hostname of the cert, but
it seems logical to me!)

## Setting the ingress

Each of the default charts has an ingress resource template, but by default
they are disabled. To try and keep all ingress configuration together I
decided to leave the subchart ingresses disables and configure it all
in the main chart.

To do this I updated `main-chart/values.yaml` to do the following:

1. Forward to the `nginx` instances deployed by the `main-chart` and `sub1` charts.
   Note the use of `{{ tpl Values.main-chart.fullname .}}` which means the mapping will
   work whatever Release name I choose.
2. Use a `nginx.ingress.kubernetes.io/rewrite-target` annotation to strip off part of
   the path when forwarding. This allows me to access paths hosted `/` on the deployed
   nginx as `https://test.localhost/main`.
3. Enable tls using the secret that was previously deployed.

The updated `yaml` is below:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    # This annotation, and the path regexes, strip off the path
    # prefix before sending to the downstream service
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    name: rewrite
  hosts:
    - host: test.localhost
      paths:
        - path: /main(/|$)(.*)
          pathType: ImplementationSpecific
          backend:
            service:
              name: "{{ tpl Values.main-chart.fullname . }}"
              port:
                number: 80
        - path: /sub1(/|$)(.*)
          pathType: ImplementationSpecific
          backend:
            service:
              name: "{{ tpl Values.sub1.fullname . }}"
              port:
                number: 80
  tls:
    - secretName: test.localhost
      hosts:
        - test.localhost
```

# Future Work

1. Enable OIDC at the ingress
2. Enable OIDC at the pods via a sidecar
3. Enable OIDC authentication of service accounts
4. Multiple ingress controllers.
