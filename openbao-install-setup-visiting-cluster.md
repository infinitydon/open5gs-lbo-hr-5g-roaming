# OpenBao Helm Installation and Configuration for cert-manager

# 1. Add OpenBao Helm repository
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update

# 2. Install OpenBao with Helm using --set flags (in-cluster only)
helm upgrade --install openbao openbao/openbao \
  --set='server.service.type=ClusterIP' \
  --set='ui.enabled=true' \
  --set='server.dataStorage.enabled=true' \
  --set='server.dataStorage.size=10Gi' \
  --namespace openbao \
  --create-namespace

# 3. Wait for OpenBao pod to be ready
kubectl wait --for=condition=Ready pod/openbao-0 -n openbao --timeout=300s

# 4. Initialize OpenBao (run inside the openbao pod)
kubectl exec -it openbao-0 -n openbao -- bao operator init -key-shares=1 -key-threshold=1 -format=json > openbao-keys.json

# Extract keys from the output (no jq needed)
UNSEAL_KEY=$(cat openbao-keys.json | jq -r '.unseal_keys_b64[0]')
ROOT_TOKEN=$(cat openbao-keys.json | jq -r '.root_token')

# 5. Unseal OpenBao (run inside the openbao pod)
kubectl exec -it openbao-0 -n openbao -- bao operator unseal $UNSEAL_KEY

# 6. Configure PKI (all commands run inside the openbao pod)
# Set environment variables and run all PKI configuration commands inside the pod

kubectl exec -it openbao-0 -n openbao -- sh -c "
export BAO_ADDR='http://127.0.0.1:8200'
export BAO_TOKEN='$ROOT_TOKEN'

# Enable PKI secrets engine
bao secrets enable pki

# Configure PKI engine with longer TTL
bao secrets tune -max-lease-ttl=87600h pki

# Generate root CA certificate
bao write pki/root/generate/internal \
    common_name='5GC Root CA' \
    ttl=87600h \
    key_bits=4096

# Configure CA and CRL URLs
bao write pki/config/urls \
    issuing_certificates='http://openbao.openbao.svc.cluster.local:8200/v1/pki/ca' \
    crl_distribution_points='http://openbao.openbao.svc.cluster.local:8200/v1/pki/crl'

# Create role for 5GC domain certificates
bao write pki/roles/5gc-domain \
    allowed_domains='5gc.mnc001.mcc001.3gppnetwork.org' \
    allow_subdomains=true \
    max_ttl='720h' \
    key_bits=2048 \
    key_type=rsa

# Enable intermediate PKI for better security
bao secrets enable -path=pki_int pki
bao secrets tune -max-lease-ttl=43800h pki_int

# Generate intermediate CSR (save to file first, then use it)
bao write -format=json pki_int/intermediate/generate/internal \
    common_name='5GC Intermediate CA' > /tmp/intermediate_output.json

# Extract CSR from JSON output using basic text processing
grep '\"csr\":' /tmp/intermediate_output.json | cut -d'\"' -f4 | sed 's/\\\\n/\\n/g' > /tmp/pki_intermediate.csr

# Sign intermediate certificate with root CA
bao write -format=json pki/root/sign-intermediate \
    csr=@/tmp/pki_intermediate.csr \
    format=pem_bundle ttl='43800h' > /tmp/cert_output.json

# Extract certificate from JSON output
grep '\"certificate\":' /tmp/cert_output.json | cut -d'\"' -f4 | sed 's/\\\\n/\\n/g' > /tmp/intermediate.cert.pem

# Set intermediate certificate
bao write pki_int/intermediate/set-signed certificate=@/tmp/intermediate.cert.pem

# Create role for intermediate CA
bao write pki_int/roles/5gc-domain \
    allowed_domains='5gc.mnc001.mcc001.3gppnetwork.org' \
    allow_subdomains=true \
    max_ttl='720h' \
    key_bits=2048 \
    key_type=rsa

# Enable Kubernetes auth method
bao auth enable kubernetes

# Configure Kubernetes auth (using service account token)
bao write auth/kubernetes/config \
    token_reviewer_jwt=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) \
    kubernetes_host='https://kubernetes.default.svc.cluster.local:443' \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create policy for cert-manager
bao policy write cert-manager - <<EOF
path \"pki_int/sign/5gc-domain\" {
  capabilities = [\"create\", \"update\"]
}
path \"pki_int/issue/5gc-domain\" {
  capabilities = [\"create\"]
}
EOF

# Create role for cert-manager service account
bao write auth/kubernetes/role/cert-manager \
    bound_service_account_names=cert-manager \
    bound_service_account_namespaces=cert-manager \
    policies=cert-manager \
    ttl=24h

echo 'PKI configuration completed successfully'
"

# 7. Create cert-manager ClusterIssuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: openbao-issuer
spec:
  vault:
    server: http://openbao.openbao.svc.cluster.local:8200
    path: pki_int/sign/5gc-domain
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: cert-manager
        serviceAccountRef:
          name: cert-manager
EOF

# 8. Example Certificate resource for 5GC domain
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: 5gc-domain-cert
  namespace: default
spec:
  privateKey:
   rotationPolicy: Always
  secretName: 5gc-domain-tls
  issuerRef:
    name: openbao-issuer
    kind: ClusterIssuer
  commonName: api.5gc.mnc001.mcc001.3gppnetwork.org
  dnsNames:
  - api.5gc.mnc001.mcc001.3gppnetwork.org
  duration: 720h # 30 days
  renewBefore: 240h # 10 days
EOF

# 9. Verify certificate issuance
echo "Checking certificate status..."
kubectl get certificate -n default
kubectl describe certificate 5gc-domain-cert -n default
kubectl get secret 5gc-domain-tls -n default

# 10. Verify OpenBao status
echo "OpenBao status:"
kubectl exec -it openbao-0 -n openbao -- bao status