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

If this update is applied, attempts to access https://test.localhost/main will
now require a login via GitHub.

### And now the problems start...

Unfortunately logging in via GitHub caused me a 500 error. Looking in the logs showed the
following:

```
[2022/06/26 07:19:24] [oauthproxy.go:775] Error creating session during OAuth2 callback: unexpected status "404": {"message":"Not Found","documentation_url":"https://docs.github.com/rest/reference/users#list-email-addresses-for-the-authenticated-user"}
```

Ffrom bit of poking around I _think_ this is down to an error in the GitHub provider
with oauth2-proxy, but I am not 100% sure.

I tried setting up with the default Google provider, but that cannot authorise domains
that do not have valid TLDs (e.g. .com), so it cannot work with my local test domain.

# Custom local OAuth provider

As an alternative to using a public OAuth provider it should be possible to use the
same sort of configuration with a local provider.

To do this we'll create a top level helm chart so we can install/uninstall the provider
independently of our app, and once again make use of the Bitnami charts, rather than
rolling our own. At least initally we don't want any additional resources, so we'll create a chart, remove the default templates and update the dependencies to include keycloak.

```
helm create local-oauth
rm local-oauth/templates/*.yaml
```

Then added the following to `Chart.yaml`:

```yaml
dependencies:
  - name: "keycloak"
    repository: "@bitnami"
    version: "9.3.2"
```

The chart can then be installed by doing the following:

```
helm dep update ./local-oauth
helm upgrade --install oauth ./local-oauth --namespace oauth --create-namespace
```

You can see what has been deployed as follows:

```
% kubectl get all --namespace oauth
NAME                     READY   STATUS    RESTARTS   AGE
pod/oauth-keycloak-0     1/1     Running   0          2m8s
pod/oauth-postgresql-0   1/1     Running   0          2m8s

NAME                              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/oauth-keycloak            LoadBalancer   10.110.195.85   <pending>     80:30490/TCP,443:32708/TCP   2m8s
service/oauth-keycloak-headless   ClusterIP      None            <none>        80/TCP                       2m8s
service/oauth-postgresql          ClusterIP      10.111.26.89    <none>        5432/TCP                     2m8s
service/oauth-postgresql-hl       ClusterIP      None            <none>        5432/TCP                     2m8s

NAME                                READY   AGE
statefulset.apps/oauth-keycloak     1/1     2m8s
statefulset.apps/oauth-postgresql   1/1     2m8s
```

(Note that I have chosen to create a new namespace for the oauth provider, this is
purely to keep it logically separate to the rest of the system.)

As you can see there are pods and services, but as yet no ingress. This can be configured via
the values file. Additionally I'm going to use a different host for this deployment to try and
make things a little clearer, and this will need some new certificates and add them as a secret

```
mkcert oauth.localhost
kubectl create secret tls oauth.localhost \
     --namespace oauth \
     --key oauth.localhost-key.pem \
     --cert oauth.localhost.pem
```

Then we can update `local-oauth/values.yaml` to enable the ingress with TLS.

```yaml
keycloak:
  ingress:
    enabled: true
    path: "/"
    pathType: "Prefix"
    ingressClassName: "nginx"
    hostname: "oauth.localhost"
    extraTls:
      - secretName: "oauth.localhost"
        hosts:
          - "oauth.localhost"
```

And now navigating to https://oauth.localhost will take us the Keycloak welcome page. By default
the admin password is randomly generated, to get the details you can do the following:

```
kubectl get secret --namespace oauth oauth-keycloak -o yaml
```

and then base64 decode the value of `admin-password`. This can then be used to log into keycloak
and configure things...

Follow the [instructions from oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#keycloak-oidc-auth-provider)
to configure a new oidc provider, update the oauth-proxy secret with the new values and change the
values file to use the following.

```yaml
extraArgs:
  - "--provider=keycloak-oidc"
  - "--redirect-url=https://test.localhost/oauth2/callback"
  - "--oidc-issuer-url=https://oauth.localhost/realms/master"
```

This can then be applied by running:

```
helm upgrade --install tst ./main-chart
```

Unfortunately all is not well. While oauth2-proxy can reach https://oauth.localhost it does not
trust the CA. Looking in the logs shows:

```
[2022/06/26 09:08:49] [provider.go:55] Performing OIDC Discovery...
[2022/06/26 09:08:49] [main.go:60] ERROR: Failed to initialise OAuth2 Proxy: error intiailising provider: could not create provider data: error building OIDC ProviderVerifier: could not get verifier builder: error while discovery OIDC configuration: failed to discover OIDC configuration: error performing request: Get "https://oauth.localhost/realms/master/.well-known/openid-configuration": x509: certificate signed by unknown authority
```

The cert created via `mkcert` uses it's own CA, and so we need to make our deployed apps trust it.
`oauth2-proxy` includes an option `--provider-ca-file` that should help with this.
You can find the location of the CA cert by running `mkcert -CAROOT`. The following commands will
create a secret with the mkcert root CA PEM file.

```
% MKCERT_ROOT=$(mkcert -CAROOT)
% kubectl create secret generic --from-file $MKCERT_ROOT/rootCA.pem mkcert-ca
```

We then need to mount the secret into the oauth2-proxy pod, and configure oauth2-proxy to use it.

```yaml
extraVolumeMounts:
  - name: mkcert-ca
    mountPath: "/opt/certs/ca"
    readOnly: true
extraVolumes:
  - name: mkcert-ca
    secret:
      secretName: mkcert-ca
      optional: false
```

And add an argument so that the proxy uses the mounted secret:

```yaml
extraArgs:
  - "--provider=keycloak-oidc"
  - "--redirect-url=https://test.localhost/oauth2/callback"
  - "--oidc-issuer-url=https://oauth.localhost/realms/master"
  - "--provider-ca-file=/opt/certs/ca/rootCA.pem"
```

_NOTE_ You will need to configure some users in KeyCloak. I set up a couple of test users with
noddy passwords and email address which I marked as verified. If you do not mark these as verified
then, as configured above, oauth2-proxy will return an unhelpful 500 error and put the following
into the log:

```
[2022/06/26 10:05:42] [oauthproxy.go:768] Error redeeming code during OAuth2 callback: email in id_token (test3@example.com) isn't verified
10.1.0.100:56036 - 4dba7c4ddd604349c6561e602a511ab5 - - [2022/06/26 10:05:42] test.localhost GET - "/oauth2/callback?state=3SP-tysNfZ2zJ7rShXhiqRGd4mdeJGj6JxJN_Wmma3Y%3A%2Fmain&session_state=20cabb0e-3a80-4b19-83eb-31e267a42d97&code=bbe1f4fa-f38a-4c7c-82a1-8fc0ae68a10d.20cabb0e-3a80-4b19-83eb-31e267a42d97.861952cb-aa8a-4276-8ec3-fd49e999cc6b" HTTP/1.1 "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/15.5 Safari/605.1.15" 500 2837 0.014
```

There is an option to allow unverifed email addresses, but in general marking them verified seems
slightly preferable.

# Future Work

1. Enable OIDC at the pods via a sidecar
2. Enable OIDC authentication of service accounts
3. Multiple ingress controllers.

```

```
