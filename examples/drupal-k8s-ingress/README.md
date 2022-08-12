# Drupal behind ingress-nginx controller

## Deployment

### Ingress Controller

In order to deploy ingress-nginx as an ingress coontroller for Kubernetes follow these steps:

Add helm repository for ingress-nginx:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

Then, deploy ingress-nginx in default namespace:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx
```

Save service external address in environment variable:

```bash
INGRESS_EXT_IP=$(kubectl get svc ingress-nginx-controller  -o=jsonpath="{.status.loadBalancer.ingress[].ip}")
```

Finally, create a HTTP request, and display only document info with `--head` argument:

```bash
curl --head "http://$INGRESS_EXT_IP/"
```

If everything worked well, you should see the following response.

```bash
HTTP/1.1 404 Not Found
Date: Tue, 09 Aug 2022 10:34:36 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive
```

The `HTTP/404 Not Found` is expected since we did not created any ingress records yet. We will do that in next section.

### Ingress Record

Ingress records are needed to properly route traffic that is reaching the ingress controller external address.

Start by examining the record defintion is in `drupal-ingress.yaml` file.

Next, create new ingress record:

```bash
kubectl apply -f drupal-ingress.yaml
```

Next, display ingress record for drupal namespace:

```bash
kubectl get ingress -n drupal
```

If the record was created successfully, you should see similar output as below:

```bash
NAME     CLASS    HOSTS                ADDRESS          PORTS   AGE
drupal   <none>   drupal.buldogchef.com   192.46.238.100   80      2m25s
```

Next, verify the application reachability. This assumes that the DNS record for the `drupal.buldogchef.com` has been set locally for testing purposes or at your DNS hosting provider.

```bash
curl --head "http://drupal.buldogchef.com/"
```

For quick test, without any DNS change, a custom header can be defined in the request.

```bash
curl --head --header 'Host: drupal.buldogchef.com' $INGRESS_EXT_IP
```

In either case, the response should contain a `HTTP/1.1 200 OK` response, indicating that the website is reachable thorough the ingress.

```bash
HTTP/1.1 200 OK
Date: Tue, 09 Aug 2022 10:55:15 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/8.0.22
Cache-Control: must-revalidate, no-cache, private
X-Drupal-Dynamic-Cache: MISS
X-UA-Compatible: IE=edge
Content-language: en
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Permissions-Policy: interest-cohort=()
Expires: Sun, 19 Nov 1978 05:00:00 GMT
X-Generator: Drupal 9 (https://www.drupal.org)
X-Drupal-Cache: MISS
```

After the service is reachable through ingress, you can disable external reachability through Node by changing service specification type from `NodePort` to `ClusterIP`.

Verify the current service port mapping

```bash
kubectl get svc drupal -n drupal
```

You should see that there is an existing port mapping between Node's higher level port `30432` and service port `80`

```bash
NAME     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
drupal   NodePort   10.128.16.41   <none>        80:30432/TCP   43m
```

Next, apply the new service definition from file:

```bash
kubectl apply -f drupal-service.yaml
```

Finally, verify the new service port mapping:

```bash
kubectl get svc drupal -n drupal
```

This time there is no external port mapping for the service:

```bash
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
drupal   ClusterIP   10.128.16.41   <none>        80/TCP    44m
```

### Cert Manager

```bash
# Create namespace
kubectl create namespace cert-manager

# Add helm repository
helm repo add jetstack https://charts.jetstack.io

# Update local cache
helm repo update

# Install custom resource definitions
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml

# Install cert-manager
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.9.1

# Verify pods in cert-manager namespace
kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-85f68f95f-bvqd2               1/1     Running   0          38s
cert-manager-cainjector-7d9668978d-ld8k9   1/1     Running   0          38s
cert-manager-webhook-66c8f6c75-xb9l4       1/1     Running   0          38s
```

#### ClusterIssuer

In order to begin issuing certificates, you will need to set up a ClusterIssuer or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

```bash
# Deploy cluster issuer
kubectl apply -f cluster-issuer.yaml
clusterissuer.cert-manager.io/letsencrypt-prod created

# Verify cluster issuer
kubectl get clusterissuer
NAME               READY   AGE
letsencrypt-prod   True    32s
```

#### Update Ingress Record

In order to use request and use a certificate, you need to add new annotation `cert-manager.io/cluster-issuer` in `metadata` key and `tls` in `spec` key for ingress record.

```bash
# Apply new definition
kubectl apply -f drupal-ingress-tls.yaml

# Monitor the cert manager logs
kubectl logs -f -cert-manager -l app=cert-manager

# Verify the TLS by hitting the http endpoint
# You will see redirection to https
curl -I -L http://drupal.buldogchef.com
HTTP/1.1 308 Permanent Redirect
Date: Fri, 12 Aug 2022 10:27:32 GMT
Content-Type: text/html
Content-Length: 164
Connection: keep-alive
Location: https://drupal.buldogchef.com

HTTP/2 200
date: Fri, 12 Aug 2022 10:27:33 GMT
content-type: text/html; charset=UTF-8
x-powered-by: PHP/8.0.22
cache-control: must-revalidate, no-cache, private
x-drupal-dynamic-cache: MISS
x-ua-compatible: IE=edge
content-language: en
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
permissions-policy: interest-cohort=()
expires: Sun, 19 Nov 1978 05:00:00 GMT
x-generator: Drupal 9 (https://www.drupal.org)
x-drupal-cache: HIT
strict-transport-security: max-age=15724800; includeSubDomains
```