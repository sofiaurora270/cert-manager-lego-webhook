# cert-manager-lego-webhook

[All in one](https://go-acme.github.io/lego/dns/#dns-providers) cert-manager dns solver webhook, based on lego.

This is a fork of https://yxwuxuanl.github.io/cert-manager-lego-webhook/ which didn't support ARM64, so I forked the repo, and made some changes and built newer binaries for it. Enjoy.

## Install

```sh
helm repo add cert-manager-lego-webhook https://sofiaurora270.github.io/cert-manager-lego-webhook/

helm upgrade --install cert-manager-lego-webhook cert-manager-lego-webhook/cert-manager-lego-webhook \
    --namespace cert-manager \
    --set=groupName=acme.lego.example.com \
    --set=certManager.namespace=cert-manager \
    --set=certManager.serviceAccount.name=cert-manager
```

## Usage

Apply each file with `kubectl apply -f <filename>`

```yaml
# step 1: create secret for dns provider
kind: Secret
apiVersion: v1
metadata:
  name: lego-alidns-secret
  namespace: cert-manager
stringData:
  # The key will be passed to Lego DNS Provider as an credentials
  # for example, https://go-acme.github.io/lego/dns/alidns/#credentials
  ALICLOUD_ACCESS_KEY: ''
  ALICLOUD_SECRET_KEY: ''
```

```yaml
# step 2a: create ClusterIssuer
kind: ClusterIssuer
apiVersion: cert-manager.io/v1
metadata:
  name: letsencrypt-staging
  namespace: cert-manager
spec:
  acme:
    privateKeySecretRef:
      name: letsencrypt-staging # <- this is the one you need to ref when creating certificates
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: '' # <- your email
    solvers:
      - dns01:
          webhook:
            groupName: acme.lego.example.com # <- Use the same groupname as when installing
            solverName: lego-solver # <- solver name, do not change
            config:
              # Available provider refer to https://go-acme.github.io/lego/dns/#dns-providers
              provider: alidns
              envFrom: # <- use secret
                secret:
                  name: lego-alidns-secret
                  namespace: 'cert-manager' # <- if not set, use cert-manager namespace
```

```yaml
# step 2b: create ClusterIssuer
kind: ClusterIssuer
apiVersion: cert-manager.io/v1
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    privateKeySecretRef:
      name: letsencrypt-prod # <- this is the one you need to ref when creating certificates
    server: https://acme-v02.api.letsencrypt.org/directory
    email: '' # <- your email
    solvers:
      - dns01:
          webhook:
            groupName: acme.lego.example.com # <- Use the same groupname as when installing
            solverName: lego-solver # <- solver name, do not change
            config:
              # Available provider refer to https://go-acme.github.io/lego/dns/#dns-providers
              provider: alidns
              envFrom: # <- use secret
                secret:
                  name: lego-alidns-secret
                  namespace: 'cert-manager' # <- if not set, use cert-manager namespace
```

```yaml
# step 3a: create Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: lego.example.com
  namespace: cert-manager
spec:
  issuerRef:
    name: lego-alidns # <- This is the cluserissuername you need to reference
    kind: ClusterIssuer
  secretName: lego.example.com-tls
  commonName: lego.example.com
  dnsNames:
    - lego.example.com
    - '*.lego.example.com'

```
```yaml
# step 3b: but better yet, just tell your ingress to use the proper clusterIssuer we created just now and let it handle everything.
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  labels:
    app: hello-world
  name: 
  namespace: <namespace> # if non-default namespace
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod # <- This is the magic!
spec:
  rules:
  - host: example.com # your domain for kube to know
    http:
      paths:
      - backend:
          service:
            name: <your-service>
            port:
              number: 80 # use appropriate port
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - example.com # your domain for cert manager to handle.
    secretName: letsencrypt-prod # This is the other magic!
```
