# Installing Ambassador Pro
---

Ambassador Pro is a commercial version of Ambassador that includes integrated Single Sign-On, powerful rate limiting, custom filters, and more. Ambassador Pro also uses a certified version of Ambassador OSS that undergoes additional testing and validation. In this tutorial, we'll walk through the process of installing Ambassador Pro in Kubernetes and show the JWT filter in action.

## 1. Clone the Ambassador Pro configuration repository
Ambassador Pro consists of a series of modules that communicate with Ambassador. The core Pro module is typically deployed as a sidecar to Ambassador. This means it is an additional process that runs on the same pod as Ambassador. Ambassador communicates with the Pro sidecar locally. Pro thus scales in parallel with Ambassador. Ambassador Pro also relies on a Redis instance for its rate limit service and several Custom Resource Definitions (CRDs) for configuration.

For this installation, we'll start with a standard set of Ambassador Pro configuration files.

```
git clone https://github.com/datawire/pro-ref-arch
```

## 2. License Key

In the `ambassador/ambassador-pro.yaml` file, update the `AMBASSADOR_LICENSE_KEY` environment variable field with the license key that is supplied as part of your trial email.

**Note:** Ambassador Pro will not start without a valid license key.

## 3. Deploy Ambassador Pro

Once you have fully configured Ambassador Pro, deploy your updated configuration. Note that the default configuration will also redeploy your current Ambassador configuration, so verify that you have the correct Ambassador version before deploying Pro.

If you're on GKE, first, create the following `ClusterRoleBinding`:

```
kubectl create clusterrolebinding my-cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud info --format="value(config.account)")
```

Then, deploy Ambassador Pro and related dependencies:

```
kubectl apply -f ambassador/
```

Verify that Ambassador Pro is running:

```
kubectl get pods | grep ambassador
ambassador-79494c799f-vj2dv            2/2       Running            0         1h
ambassador-pro-redis-dff565f78-88bl2   1/1       Running            0         1h
```

**Note:** If you are not deploying in a cloud environment that supports the `LoadBalancer` type, you will need to change the `ambassador/ambassador-service.yaml` to a different service type (e.g., `NodePort`).

By default, Ambassador Pro uses ports 8081 and 8082 for rate-limiting
and filtering, respectively.  If for whatever reason those assignments
are problematic (perhaps you [set
`service_port`](/reference/running/#running-as-non-root) to one of
those), you can set adjust these by setting environment variables:

  - `GRPC_PORT`: Which port to serve the RateLimitService on; `8081`
    by default.
  - `APRO_AUTH_PORT`: Which port to serve the filtering AuthService
    on; `8082` by default.

If you have deployed Ambassador with
[`AMBASSADOR_NAMESPACE`, `AMBASSADOR_SINGLE_NAMESPACE`](/reference/running/#namespaces), or
[`AMBASSADOR_ID`](/reference/running/#multiple-ambassadors-in-one-cluster)
set, you will also need to set them in the Pro container.

## 4. Configure JWT authentication

Now that you have Ambassador Pro running, we'll show a few features of Ambassador Pro. We'll start by configuring Ambassador Pro's JWT authentication filter.

```
kubectl apply -f jwt/
```

This will configure the following `FilterPolicy`:

```
---
apiVersion: getambassador.io/v1beta2
kind: FilterPolicy
metadata:
  name: httpbin-filterpolicy
  namespace: default
spec:
  # everything defaults to private; you can create rules to make stuff
  # public, and you can create rules to require additional scopes
  # which will be automatically checked
  rules:
  - host: "*"
    path: /jwt-httpbin/*
    filters:
    - name: jwt-filter
  - host: "*"
    path: /httpbin/*
    filters: null
```

Get the External IP address of your Ambassador service:

```
AMBASSADOR_IP=$(kubectl get svc ambassador -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

We'll now test Ambassador Pro with the `httpbin` service. First, curl to the `httpbin` URL This URL is public, so it returns successfully without an authentication token.

```
$ curl -k https://$AMBASSADOR_IP/httpbin/ip # No authentication token
{
  "origin": "108.20.119.124, 35.194.4.146, 108.20.119.124"
}
```

Send a request to the `jwt-httpbin` URL, which is protected by the JWT filter. This URL is not public, so it returns a 401.

```
$ curl -i -k https://$AMBASSADOR_IP/jwt-httpbin/ip # No authentication token
HTTP/1.1 401 Unauthorized
content-length: 58
content-type: text/plain
date: Mon, 04 Mar 2019 21:18:17 GMT
server: envoy
```

Finally, send a request with a valid JWT to the `jwt-httpbin` URL, which will return successfully.

```
$ curl -k --header "Authorization: BwidHlwIjoiSldUIn0.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ." https://$AMBASSADOR_IP/jwt-httpbin/ip
{
  "origin": "108.20.119.124, 35.194.4.146, 108.20.119.124"
}
```

## 5. Configure additional Ambassador Pro services

Ambassador Pro has many more features such as rate limiting, OAuth integration, and more.

### Enabling Rate limiting

For more information on configuring rate limiting, consult the [Advanced Rate Limiting tutorial ](/user-guide/advanced-rate-limiting) for information on configuring rate limits.

### Enabling Single Sign-On

 For more information on configuring the OAuth filter, see the [Single Sign-On with OAuth and OIDC](/user-guide/oauth-oidc-auth) documentation.

### Enabling Service Preview

Service Preview requires a command-line client, `apictl`. For instructions on configuring Service Preview, see the [Service Preview tutorial](/docs/dev-guide/service-preview).

### Enabling Consul Connect integration

Ambassador Pro's Consul Connect integration is deployed as a separate Kubernetes service. For instructions on deploying Consul Connect, see the [Consul Connect integration guide](/user-guide/consul-connect-ambassador).
