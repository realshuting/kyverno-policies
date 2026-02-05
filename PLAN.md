Create Kyverno policies for the following

1. Runtime Class Assignment
Example
spec:
  runtimeClassName: kata-qemu-snp


2. InitData Injection via Annotation
Ref: https://confidentialcontainers.org/docs/features/initdata/

Annotation Format:
metadata:
  annotations:
    io.katacontainers.config.runtime.cc_init_data: "<base64-gzip-encoded-json>"

The plaintext initdata can be provided as a configMap which makes it easier to modify as needed.

Example initdata toml configmap

apiVersion: v1
kind: ConfigMap
metadata:
  name: initdata-config
data: |
version = "0.1.0"
algorithm = "sha384"

[data]
"policy.rego" = '''
package agent_policy
# Your OPA Rego policy here
'''

"aa.toml" = '''
[token_configs.kbs]
url = "https://your-kbs:8080"
'''

"cdh.toml" = '''
[kbc]
name = "cc_kbc"
url = "https://your-kbs:8080"
'''


3. Sidecar/InitContainers

Init Container: Adds attestation verification container

Secure Sidecar: Deploys mTLS-protected proxy with auto-generated certificates

Automatically inject a secure mTLS sidecar that provides:
- Attestation status API endpoint (HTTPS with mTLS)
- TEE environment information
- Optional reverse proxy for application traffic
- Certificates retrieved from KBS during attestation

Based on: https://github.com/confidential-devhub/cococtl/blob/main/sidecar/README.md

ConfigMap-Based Configuration:

# ConfigMap
name: coco-sidecar-config
data:
  sidecar_image: "ghcr.io/.../secure-sidecar:v0.1.0"
  tls_cert_uri: "kbs:///default/sidecar-tls/app/cert.pem"
  https_port: "8443"
  forward_port: "8080"


4. Sealed secrets injection
Ref: https://confidentialcontainers.org/docs/features/sealed-secrets/

ConfigMap-Based Configuration:
# ConfigMap
name: coco-secret
data:
  secret_resource: "kbs:///myns/secret"
  target_path: "/secret/user"


5. Validate structure of initdata

Ref: https://confidentialcontainers.org/docs/features/initdata/

Validate Initdata

 - version = "0.1.0" present
 - algorithm is sha256/sha384/sha512
 - [data] section exists
 - "policy.rego" field present
 - "aa.toml" field present with KBS config
 - "cdh.toml" field present with KBC config

## Kyverno references

https://github.com/kyverno/policies
https://kyverno.io/

