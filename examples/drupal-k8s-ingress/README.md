# Drupal behind ingress-nginx controller

## Deployment

### Ingress Controller

```bash
# Add helm repository for ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Deploy ingress-nginx in default namespace
helm install ingress-nginx ingress-nginx/ingress-nginx

# Retrieve the service external address
INGRESS_EXT_IP=$(kubectl get svc ingress-nginx-controller  -o=jsonpath="{.status.loadBalancer.ingress[].ip}")

# Verify the reachability, it is normal to get a HTTP/404 since no rules have been set
curl -I "http://$INGRESS_EXT_IP/"
HTTP/1.1 404 Not Found
Date: Tue, 09 Aug 2022 10:34:36 GMT
Content-Type: text/html
Content-Length: 146
Connection: keep-alive
```

### Ingress Record

The ingress record defintion is in `drupal-ingress.yaml` file.

```bash
# Create new ingress record
kubectl apply -f drupal-ingress.yaml
ingress.networking.k8s.io/drupal created

# Display ingress record for drupal namespace
kubectl get ingress -n drupal
NAME     CLASS    HOSTS                ADDRESS          PORTS   AGE
drupal   <none>   drupal.buldogchef.com   192.46.238.100   80      2m25s

# Verify the reachability, this assumes that the DNS record for the drupal.example.com
# has been set locally for testing purposes or at your dns provider

curl -I "http://drupal.buldogchef.com/"
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

# If you don't have DNS records set up, you can still test by setting the ost header
# However proper DNS record will be required for certificate generation
curl -I -H 'Host: drupal.buldogchef.com' $INGRESS_EXT_IP
HTTP/1.1 200 OK
Date: Fri, 12 Aug 2022 09:25:20 GMT
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
X-Drupal-Cache: HIT
```

After the service is reachable through ingress, you can disable external reachability through Node by changing service specification type from `NodePort` to `ClusterIP`.

```bash
# Verify the current service port mapping
kubectl get svc drupal -n drupal
NAME     TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
drupal   NodePort   10.128.16.41   <none>        80:30432/TCP   43m

# Apply the new configuration
kubectl apply -f drupal-service.yaml

# Verify the current service port mapping
kubectl get svc drupal -n drupal
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