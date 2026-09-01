# Kubernetes ConfigMaps & Secrets

## Objective

Deploy a simple Nginx Pod and use a Kubernetes **ConfigMap** and **Secret** to provide configuration to the container without hard-coding any values directly into the Pod spec.

## Repository Contents

```
kubernetes-configmaps-secrets/
├── README.md
├── configmap.yaml
├── secret.yaml
└── pod.yaml
```

|File            |Purpose                                                                                       |
|----------------|----------------------------------------------------------------------------------------------|
|`configmap.yaml`|Defines `APP_NAME`, `APP_ENV`, `APP_PORT`                                                     |
|`secret.yaml`   |Defines `DB_USERNAME`, `DB_PASSWORD`                                                          |
|`pod.yaml`      |Nginx (`nginx:alpine`) pod that injects both the ConfigMap and Secret as environment variables|

## Prerequisites


• A running Kubernetes cluster (e.g. Minikube, Kind, or a cloud cluster).
• `kubectl` installed and configured to point at that cluster.

Check both before starting:

```bash
kubectl version --client
kubectl cluster-info
```

## Task 1 — Create the ConfigMap

**`configmap.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: bookstore
  APP_ENV: development
  APP_PORT: "8080"
```

**Apply and verify:**

```bash
kubectl apply -f configmap.yaml
kubectl get configmap app-config
kubectl describe configmap app-config
```

## Task 2 — Create the Secret

**`secret.yaml`:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_USERNAME: bookstore_user
  DB_PASSWORD: BookStore@123
```

**Apply and verify:**

```bash
kubectl apply -f secret.yaml
kubectl get secret app-secret
kubectl describe secret app-secret
```


Note: `describe` only shows key names and byte sizes, not the actual values — that’s expected Secret behavior, not a fault. `stringData` is used here for readability; Kubernetes automatically base64-encodes it into the underlying `data` field on creation.

## Task 3 — Create the Pod

**`pod.yaml`** (base version before injection):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bookstore-pod
spec:
  containers:

◦ name: bookstore.
      image: nginx:alpine
```

**Apply and verify:**

```bash
kubectl apply -f pod.yaml
kubectl get pods
```

## Task 4 — Inject the ConfigMap

Update `pod.yaml` so `APP_NAME`, `APP_ENV`, and `APP_PORT` are pulled from the ConfigMap using `envFrom`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bookstore-pod
spec:
  containers:

◦ name: bookstore.
      image: nginx:alpine
      envFrom:

■ configMapRef:.
            name: app-config
```

Pod specs are immutable, so the Pod must be deleted and recreated:

```bash
kubectl delete pod bookstore-pod
kubectl apply -f pod.yaml
kubectl get pods
kubectl exec -it bookstore-pod -- env | grep APP_
```

## Task 5 — Inject the Secret

Update `pod.yaml` again to also pull `DB_USERNAME` and `DB_PASSWORD` from the Secret:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bookstore-pod
spec:
  containers:

◦ name: bookstore.
      image: nginx:alpine
      envFrom:

■ configMapRef:.
            name: app-config

■ secretRef:.
            name: app-secret
```

```bash
kubectl delete pod bookstore-pod
kubectl apply -f pod.yaml
kubectl get pods
kubectl exec -it bookstore-pod -- env | grep DB_
```

## Task 6 — Final Verification

```bash
kubectl get pods
kubectl exec -it bookstore-pod -- env
```

Expected environment variables in the output:

```
APP_NAME=bookstore
APP_ENV=development
APP_PORT=8080
DB_USERNAME=bookstore_user
DB_PASSWORD=BookStore@123
```

## Key Concepts Demonstrated


• **ConfigMap**: stores non-sensitive configuration data as key-value pairs, decoupled from the Pod spec..
• **Secret**: stores sensitive data (credentials) similarly to a ConfigMap, but base64-encoded at rest and treated with tighter access controls by Kubernetes..
• **`envFrom`**: injects *all* keys from a ConfigMap or Secret as environment variables in one block, rather than mapping each variable individually with `env.valueFrom`..
• **Pod immutability**: most fields in a running Pod’s spec (like `containers`) can’t be patched in place — changes require deleting and recreating the Pod (or managing it via a Deployment, which handles this automatically)..

## Cleanup

```bash
kubectl delete pod bookstore-pod
kubectl delete configmap app-config
kubectl delete secret app-secret
