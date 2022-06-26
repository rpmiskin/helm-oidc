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

### Creating certificates

For local development I have set up a local DNS server so that `*.localhost`
will point to my development environment both from the host and also from
within the k8s environment. Instructions on how to do this can be found
in Github [here](https://github.com/rpmiskin/localhost-dns).

I have also configured a tool called `mkcert` to easily create certificates
which my local environment trusts. The basic flow for enable `mkcert`,
creating creating new certificates and adding them as k8s secrets
is as follows:

```bash
mkcert -install
mkcert test.localhost
kubectl create secret tls \
        test.localhost \
        --key test.localhost-key.pem \
        --cert test.localhost.pem
```

(The secret name doesn't have to match the hostname of the cert, but
it seems logical to me!)

### Setting up the ingress

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

## Authentication at the ingress

The [nginx-ingress-controller](https://kubernetes.github.io/ingress-nginx/) does not
directly support OAuth authentiction, but it does support
[external providers](https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/)
with the documentation covering the use of
[oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/).

### Deploy OAuth2 Proxy via Helm

For this will add OAuth2 Proxy to our main chart. This means we we have a new
instance of the proxy within our helm release. The good folk at Bitnami have packaged
a [helm chart](https://github.com/bitnami/charts/tree/master/bitnami/oauth2-proxy/)
and we will use that as our starting point.

First add the bitnami helm repo

```
helm repo add bitnami https://charts.bitnami.com/bitnami`
```

The we can add it as a depdendency to our main-chart by adding the following to the
`Chart.yaml`.

```yaml
dependencies:
  - name: "oauth2-proxy"
    repository: "@bitnami"
    version: "2.1.9"
```

Now if we upgrade our running chart we will see some `oauth2` related pods running.

```
% helm upgrade --install tst ./main-chart
% kubectl get pod
NAME                                READY   STATUS    RESTARTS        AGE
tst-main-chart-68d656c79f-4sd9n     1/1     Running   0               175m
tst-oauth2-proxy-868bc96498-578c6   1/1     Running   2 (5m37s ago)   6m7s
tst-redis-master-0                  1/1     Running   0               6m7s
tst-sub1-5c48868895-rqfz9           1/1     Running   0               175m
tst-sub2-6968cc6d47-zqlc5           1/1     Running   0               175m
tst-sub3-64685d587f-rv8gw           1/1     Running   0               175m
```

In the above output you'll see that oauth2 restarted twice, that appears to have been
while waiting for `redis` to be available.

oauth2 can now be configured by use of the `main-chart/values.yaml` file
For example the following will add an annotation to the oauth2-proxy

```yaml
oauth2-proxy:
  podAnnotations:
    foo: "bar"
```

Redeploying and describing the pod will show our new annotation has been applied.

```
Name:         tst-oauth2-proxy-797775f8c9-swqjr
Namespace:    default
Priority:     0
Node:         docker-desktop/192.168.65.4
Start Time:   Sat, 25 Jun 2022 19:26:45 +0100
Labels:       app.kubernetes.io/component=oauth2-proxy
              app.kubernetes.io/instance=tst
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=oauth2-proxy
              helm.sh/chart=oauth2-proxy-2.1.9
              pod-template-hash=797775f8c9
Annotations:  checksum/config: 82b27fec360e192c2ff02b3516e43c75328b8dfa3e8b6ccd7ac50aca69019436
              foo: bar
...
```

Full details of the variables available to configure the proxy available in the
[Bitnami
documentation pages](https://github.com/bitnami/charts/tree/master/bitnami/oauth2-proxy/).

### Configuring oauth-proxy and nginx for authentication

(?) How to set the `--provider` option with the Helm chart? - Looks like `extraArgs`

Some of the configuration that is needed by oauth2-proxy is sensitive, these can be provided
in the `values.yaml` individually and they are used to populate a secret, alternatively
it is possibly do create the secret manually and provide the necessary details.

The secret needs to looks like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: oauth-proxy-secret
type: Opaque
stringData:
  client-id: "aaaa"
  client-secret: "bbbb"
  cookie-secret: "cccc"
```

To create a cookie-secret value you can run the following:

```python
python3 -c 'import os,secrets,base64; print(base64.b64encode(os.urandom(16)).decode("ascii"))'
```

`client-id` and `client-secret` need to come from your OAuth provider - I followed the
instructions [here](https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/)
to use GitHub as a provider.

If you save the secret yaml to a file you can apply it like this:

```
kubectl apply -f oauth-proxy-secret.yaml
```

And check that it was correctly created using:

```
% kubectl describe secret oauth-proxy-secret

Name:         oauth-proxy-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
client-id:      20 bytes
client-secret:  40 bytes
cookie-secret:  24 bytes
```

The `oauth2-proxy` section of the `main-chart/values.yaml` should now be updated to look
like this:

```yaml
oauth2-proxy:
  configuration:
    existingSecret: "oauth-proxy-secret"
  extraArgs:
    - "--provider=github"
```

First we enable an ingress to allow access to the newly deployed proxy. This can be enabled
via the values file. Below is my example using, not that the same secret is used for
the certificates as before.

```yaml
oauth2-proxy:
  configuration:
    existingSecret: "oauth-proxy-secret"
  extraArgs:
    - "--provider=github"
  ingress:
    path: "/oauth"
    pathType: "Prefix"
    enabled: true
    ingressClassName: "nginx"
    hostname: "test.localhost"
    extraTls:
      - secretName: "test.localhost"
        hosts:
          - "test.localhost"
```

Secondly we add some annotations to the original ingress so it will use the new proxy, specifically
the `auth-url` and `auth-signin` annotations.

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    # This annotation, and the path regexes, strip off the path
    # prefix before sending to the downstream service
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    # The auth-url and auth-signin annotations ensure that connections via this
    # ingress are forwarded to oauth2-proxy
    nginx.ingress.kubernetes.io/auth-url: "https://$host/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://$host/oauth2/start?rd=$escaped_request_uri"
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

If this update is applied, attempts to access https://test.localhost/main will now require a login
via GitHub.

# Future Work

1. Enable OIDC at the ingress
2. Enable OIDC at the pods via a sidecar
3. Enable OIDC authentication of service accounts
4. Multiple ingress controllers.
