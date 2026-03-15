# Kyverno Policies for Confidential Containers (CoCo)

This repository contains Kyverno policies for managing Confidential Containers workloads on Kubernetes/OpenShift clusters.

## Prerequisites

- Kubernetes/OpenShift cluster with Kata Containers runtime configured
- Kyverno installed in the cluster

### Install Kyverno

```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.16.2/install.yaml
```

## Policies

### Mutating Policies

| Policy | File | Description |
|--------|------|-------------|
| Add RuntimeClass | `policies/mutate/add-runtimeclass.yaml` | Automatically adds `runtimeClassName: kata-qemu-coco-dev` to pods with label `coco: enabled` |
| Inject InitData | `policies/mutate/inject-initdata.yaml` | Injects CoCo initdata annotation from a ConfigMap |
| Inject Sidecar | `policies/mutate/inject-sidecar.yaml` | Injects secure mTLS sidecar for attestation API |
| Inject Init Container | `policies/mutate/inject-init-container.yaml` | Injects attestation verification init container |
| Inject Sealed Secret | `policies/mutate/inject-sealed-secret.yaml` | Injects sealed secret references (secretKeyRef + volume mount) for KBS |

### Validation Policies

| Policy | File | Description |
|--------|------|-------------|
| Validate InitData ConfigMap | `policies/validate/validate-initdata-configmap.yaml` | Validates structure of initdata ConfigMaps |

## Usage

### 1. Runtime Class Assignment

Add the label `coco: enabled` to your pods to automatically assign the `kata-qemu-coco-dev` runtime class:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-coco-pod
  labels:
    coco: enabled
spec:
  containers:
    - name: app
      image: myapp:latest
```

### 2. InitData Injection

1. Create a ConfigMap with base64-gzip encoded initdata:

```bash
# Encode your initdata TOML
echo '<your-initdata-toml>' | gzip | base64
```

2. Create the ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-initdata
data:
  data: <base64-gzip-encoded-content>
```

3. Reference it in your pod:

```yaml
metadata:
  labels:
    coco: enabled
  annotations:
    coco.io/initdata-configmap: my-initdata
```

### 3. Sidecar Injection

1. Create a sidecar configuration ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-sidecar-config
data:
  sidecar_image: "ghcr.io/confidential-containers/secure-sidecar:latest"
  tls_cert_uri: "kbs:///default/sidecar-tls/app/cert.pem"
  https_port: "8443"
  forward_port: "8080"
```

2. Reference it in your pod:

```yaml
metadata:
  annotations:
    coco.io/sidecar-config: my-sidecar-config
```

### 4. Init Container Injection

1. Create an init container configuration ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-init-config
data:
  init_image: "ghcr.io/confidential-containers/attestation-init:latest"
  kbs_url: "https://kbs.example.com:8080"
  timeout: "60"
```

2. Reference it in your pod:

```yaml
metadata:
  annotations:
    coco.io/init-config: my-init-config
```

### 5. Sealed Secrets Injection

The sealed secret flow:
1. Create a JSON envelope: `{"version":"0.1.0","type":"vault","name":"kbs:///default/mysecret/secret","provider":"kbs","provider_settings":{},"annotations":{}}`
2. Encode as: `sealed.<header>.<base64url(json)>.<signature>`
3. Store as a regular K8s Secret
4. Pod references it via secretKeyRef env vars and volume mounts (injected by this policy)
5. Kata agent unwraps the sealed secret inside TEE via KBS

Steps:

1. Create the sealed secret (K8s Secret with sealed value):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sealed-secret
type: Opaque
stringData:
  secret: "sealed.<header>.<base64url(json)>.<signature>"
```

2. Create a ConfigMap that tells the policy how to inject the secret:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-secret-config
data:
  env_name: PROTECTED_SECRET
  secret_name: sealed-secret
  secret_key: secret
  volume_name: sealed-secret-volume
  mount_path: /sealed/myvalue
```

3. Reference it in your pod:

```yaml
metadata:
  labels:
    coco: enabled
  annotations:
    coco.io/sealed-secret-config: my-secret-config
```

### 6. Validate InitData ConfigMap

ConfigMaps with label `coco.io/type: initdata` are validated to ensure they contain required fields:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-initdata
  labels:
    coco.io/type: initdata
data:
  version: "0.1.0"
  algorithm: "sha384"  # sha256, sha384, or sha512
  policy.rego: |
    package agent_policy
    default allow = true
  aa.toml: |
    [token_configs.kbs]
    url = "https://kbs.example.com:8080"
  cdh.toml: |
    [kbc]
    name = "cc_kbc"
    url = "https://kbs.example.com:8080"
```

## Folder Structure

```
.
├── configmaps/           # Example ConfigMaps for testing
├── policies/
│   ├── mutate/           # Mutating policies
│   └── validate/         # Validation policies
└── tests/                # Test manifests
```

## References

- [Kyverno Documentation](https://kyverno.io/)
- [Kyverno Policies Repository](https://github.com/kyverno/policies)
- [Confidential Containers InitData](https://confidentialcontainers.org/docs/features/initdata/)
- [Confidential Containers Sealed Secrets](https://confidentialcontainers.org/docs/features/sealed-secrets/)
