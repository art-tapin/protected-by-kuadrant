# Protecting a simple service with Kuadrant

Big kudos to [Gui](https://github.com/Kuadrant/kuadrant-operator/commits?author=guicassolato) for his original guide on [Authenticated Rate Limiting for Application Developers](https://github.com/Kuadrant/kuadrant-operator/blob/7e8b91103d954bd91d293a62203063a89931dad4/doc/user-guides/authenticated-rl-for-app-developers.md). Most of the material was taken from it.


In this guide, we will rate limit a sample REST API called **Httpbin**. In reality, this API is just a Request&Response service that contains some useful endopints for debugging purpouses. The one that we are focuesd on right now is a GET request for getting cookie data.

We will define 2 users of the API, which can send requests to the API at different rates, based on their user IDs. The authentication method used is **API key**.

| User ID | Rate limit                             |
|---------|----------------------------------------|
| alice   | 5rp10s ("5 requests every 10 seconds") |
| bob     | 2rp10s ("2 requests every 10 seconds") |

<br/>

## Run the steps ① → ④

### ① Setup

This step uses tooling from the Kuadrant Operator component to create a containerized Kubernetes server locally using [Kind](https://kind.sigs.k8s.io),
where it installs Istio, Kubernetes Gateway API and Kuadrant itself.

> **Note:** In production environment, these steps are usually performed by a cluster operator with administrator privileges over the Kubernetes cluster.

Clone the project:

```sh
git clone https://github.com/Kuadrant/kuadrant-operator && cd kuadrant-operator
```

Setup the environment:

```sh
make local-setup
```

Request an instance of Kuadrant:

```sh
kubectl -n kuadrant-system apply -f - <<EOF
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
spec: {}
EOF
```

### ② Deploy the Httpbin API
Navigate to `protected-by-kuadrant` root folder and create the deployment:

```sh
kubectl apply -f example/httpbin/httpbin.yaml
```

Create a HTTPRoute to route traffic to the service via Istio Ingress Gateway:

```sh
kubectl apply -f example/httpbin/route-httpbin.yaml
```

Verify the route works:

```sh
curl -H 'Host: api.httpbin.com' http://localhost:9080/ -i
# HTTP/1.1 200 OK
```

> **Note**: If the command above fails to hit the Httpbin API on your environment, try forwarding requests to the service:
>
> ```sh
> kubectl port-forward -n istio-system service/istio-ingressgateway 9080:80 2>&1 >/dev/null &
> ```

### ③ Enforce authentication on requests to the Httpbin API

Create a Kuadrant `AuthPolicy` to configure the authentication. 

This AuthPolicy safeguards the paths that match "/cookies*." It employs an "api-key-users" identity scheme, which utilizes API key credentials found in the authorization header. This policy ensures that only authenticated users with valid API keys can access the protected "/cookies*" endpoints:
```sh
kubectl apply -f example/httpbin/authpolicy-api-based.yaml
```

Verify the authentication works by sending a request to the Httpbin API without API key:

```sh
curl -H 'Host: api.httpbin.com' http://localhost:9080/cookies -i
# HTTP/1.1 401 Unauthorized
# www-authenticate: APIKEY realm="api-key-users"
# x-ext-auth-reason: "credential not found"
```

Create API keys for users `alice` and `bob` to authenticate:

> **Note:** Kuadrant stores API keys as Kubernetes Secret resources. User metadata can be stored in the annotations of the resource.

```sh
kubectl apply -f example/httpbin/api-keys-bob-alice.yaml
```

### ④ Enforce authenticated rate limiting on requests to the Httpbin API

Create a Kuadrant `RateLimitPolicy` to configure rate limiting:

```sh
kubectl apply -f example/httpbin/v2-rate-limit-policy.yaml
```

> **Note:** It may take a couple of minutes for the RateLimitPolicy to be applied depending on your cluster.

<br/>

Verify the rate limiting works by sending requests as Alice and Bob.

Up to 5 successful (`200 OK`) requests every 10 seconds allowed for Alice, then `429 Too Many Requests`:

```sh
while :; do curl --write-out '%{http_code}' --silent --output /dev/null -H 'Authorization: APIKEY IAMBOB' -H 'Host: api.httpbin.com' http://localhost:9080/cookies | egrep --color "\b(429)\b|$"; sleep 1; done
```

Up to 2 successful (`200 OK`) requests every 10 seconds allowed for Bob, then `429 Too Many Requests`:

```sh
while :; do curl --write-out '%{http_code}' --silent --output /dev/null -H 'Authorization: APIKEY IAMALICE' -H 'Host: api.httpbin.com' http://localhost:9080/cookies | egrep --color "\b(429)\b|$"; sleep 1; done
```

## Cleanup

```sh
make local-cleanup
```
