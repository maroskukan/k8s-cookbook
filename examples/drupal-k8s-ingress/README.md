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
