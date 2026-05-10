# KUBERNETES

## Install via Operator (recommended)

```bash
# Install the Keycloak Operator
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/24.0.0/kubernetes/keycloaks.k8s.keycloak.org-v1.yml
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/24.0.0/kubernetes/keycloakrealmimports.k8s.keycloak.org-v1.yml
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/24.0.0/kubernetes/kubernetes.yml
```

Or via Helm (community chart):
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install keycloak bitnami/keycloak \
  --namespace keycloak \
  --create-namespace \
  -f values.yaml
```

---

## Keycloak CR

```yaml
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: keycloak
  namespace: keycloak
spec:
  instances: 2                       # HA — 2+ replicas

  db:
    vendor: postgres
    host: postgres.keycloak.svc
    port: 5432
    database: keycloak
    usernameSecret:
      name: keycloak-db-secret
      key: username
    passwordSecret:
      name: keycloak-db-secret
      key: password

  http:
    tlsSecret: keycloak-tls          # TLS secret name (cert + key)
    httpEnabled: false

  hostname:
    hostname: auth.example.com
    strict: true
    strictBackchannel: false

  proxy:
    headers: xforwarded             # if behind ingress/LB

  additionalOptions:
    - name: cache
      value: ispn
    - name: cache-stack
      value: kubernetes

  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "2Gi"
```

---

## DB secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-db-secret
  namespace: keycloak
type: Opaque
stringData:
  username: keycloak
  password: supersecret
```

## TLS secret

```bash
kubectl create secret tls keycloak-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n keycloak
```

---

## KeycloakRealmImport CR (config as code)

Define realm configuration declaratively — Keycloak Operator applies it automatically.

```yaml
apiVersion: k8s.keycloak.org/v2alpha1
kind: KeycloakRealmImport
metadata:
  name: myrealm-import
  namespace: keycloak
spec:
  keycloakCRName: keycloak             # references the Keycloak CR
  realm:
    realm: myrealm
    enabled: true
    displayName: My Application
    sslRequired: external
    registrationAllowed: false
    loginWithEmailAllowed: true
    duplicateEmailsAllowed: false
    bruteForceProtected: true
    failureFactor: 5
    waitIncrementSeconds: 30
    maxFailureWaitSeconds: 900

    clients:
      - clientId: my-frontend
        name: My Frontend App
        publicClient: true
        redirectUris:
          - https://app.example.com/*
        webOrigins:
          - https://app.example.com
        standardFlowEnabled: true
        directAccessGrantsEnabled: false

      - clientId: my-api
        name: My API Service
        publicClient: false
        serviceAccountsEnabled: true
        standardFlowEnabled: false

    roles:
      realm:
        - name: app-admin
          description: Application administrator
        - name: app-viewer
          description: Read-only access

    groups:
      - name: ops-team
        realmRoles:
          - app-admin

    users:
      - username: admin-user
        email: admin@example.com
        firstName: Admin
        lastName: User
        enabled: true
        credentials:
          - type: password
            value: changeme
            temporary: true
        realmRoles:
          - app-admin
```

---

## Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak
  namespace: keycloak
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffer-size: "128k"
    nginx.ingress.kubernetes.io/proxy-buffers: "4 256k"
    nginx.ingress.kubernetes.io/proxy-busy-buffers-size: "256k"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - auth.example.com
      secretName: keycloak-tls
  rules:
    - host: auth.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: keycloak-service
                port:
                  number: 8443
```

---

## Useful kubectl commands

```bash
# Check Keycloak pods
kubectl get pods -n keycloak

# Keycloak operator logs
kubectl logs -n keycloak deployment/keycloak-operator -f

# Keycloak pod logs
kubectl logs -n keycloak statefulset/keycloak -f

# Check realm import status
kubectl get keycloakrealmimport -n keycloak
kubectl describe keycloakrealmimport myrealm-import -n keycloak

# Get Keycloak CR status
kubectl describe keycloak keycloak -n keycloak

# Open admin console via port-forward
kubectl port-forward -n keycloak svc/keycloak-service 8080:80
# then: http://localhost:8080/admin
```
