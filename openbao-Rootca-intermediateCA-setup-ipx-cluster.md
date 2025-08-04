# Complete OpenBao PKI Setup for 5G Roaming with AppRole Authentication

```
export BAO_ADDR="https://openbao.192.168.4.1.nip.io"
export VAULT_SKIP_VERIFY=true
bao login -tls-skip-verify <ROOT_TOKEN>



# Enable PKI engine for Root CA

bao secrets enable -path=pki-root pki
bao secrets tune -max-lease-ttl=87600h pki-root

# Generate 5G IPX Root CA

bao write pki-root/root/generate/internal \
    common_name="5G IPX Root CA" \
    organization="5G IPX Exchange" \
    country="US" \
    locality="IPX City" \
    province="IPX State" \
    ttl=87600h \
    key_bits=4096

# Configure CA URLs

bao write pki-root/config/urls \
    issuing_certificates="https://openbao.192.168.4.1.nip.io/v1/pki-root/ca" \
    crl_distribution_points="https://openbao.192.168.4.1.nip.io/v1/pki-root/crl"

# Enable PKI for HPLMN

bao secrets enable -path=pki-hplmn pki
bao secrets tune -max-lease-ttl=8760h pki-hplmn

# Generate HPLMN intermediate CA CSR

bao write -format=json pki-hplmn/intermediate/generate/internal \
    common_name="HPLMN mnc070.mcc999 Intermediate CA" \
    organization="HPLMN MNC070 MCC999" \
    country="US" \
    key_bits=4096 \
    | jq -r '.data.csr' > hplmn_intermediate.csr

# Sign with root CA

bao write -format=json pki-root/root/sign-intermediate \
    csr=@hplmn_intermediate.csr \
    format=pem_bundle \
    ttl=8760h \
    | jq -r '.data.certificate' > hplmn_intermediate.cert.pem

# Set signed certificate

bao write pki-hplmn/intermediate/set-signed \
    certificate=@hplmn_intermediate.cert.pem

# Configure HPLMN URLs

bao write pki-hplmn/config/urls \
    issuing_certificates="https://openbao.192.168.4.1.nip.io/v1/pki-hplmn/ca" \
    crl_distribution_points="https://openbao.192.168.4.1.nip.io/v1/pki-hplmn/crl"

# Create HPLMN role for 48-day certificates

bao write pki-hplmn/roles/5g-roaming-certs \
    allowed_domains="5gc.mnc070.mcc999.3gppnetwork.org" \
    allow_subdomains=true \
    max_ttl=1152h \
    ttl=1152h \
    key_bits=2048 \
    key_type=rsa \
    allow_any_name=false \
    enforce_hostnames=true \
    allow_localhost=false \
    allow_ip_sans=false \
    server_flag=true \
    client_flag=true

# Cleanup

rm hplmn_intermediate.csr hplmn_intermediate.cert.pem

# Enable PKI for VPLMN

bao secrets enable -path=pki-vplmn pki
bao secrets tune -max-lease-ttl=8760h pki-vplmn

# Generate VPLMN intermediate CA CSR

bao write -format=json pki-vplmn/intermediate/generate/internal \
    common_name="VPLMN mnc01.mcc001 Intermediate CA" \
    organization="VPLMN MNC01 MCC001" \
    country="US" \
    key_bits=4096 \
    | jq -r '.data.csr' > vplmn_intermediate.csr

# Sign with root CA

bao write -format=json pki-root/root/sign-intermediate \
    csr=@vplmn_intermediate.csr \
    format=pem_bundle \
    ttl=8760h \
    | jq -r '.data.certificate' > vplmn_intermediate.cert.pem

# Set signed certificate

bao write pki-vplmn/intermediate/set-signed \
    certificate=@vplmn_intermediate.cert.pem

# Configure VPLMN URLs

bao write pki-vplmn/config/urls \
    issuing_certificates="https://openbao.192.168.4.1.nip.io/v1/pki-vplmn/ca" \
    crl_distribution_points="https://openbao.192.168.4.1.nip.io/v1/pki-vplmn/crl"

# Create VPLMN role for 48-day certificates

bao write pki-vplmn/roles/5g-roaming-certs \
    allowed_domains="5gc.mnc001.mcc001.3gppnetwork.org" \
    allow_subdomains=true \
    max_ttl=1152h \
    ttl=1152h \
    key_bits=2048 \
    key_type=rsa \
    allow_any_name=false \
    enforce_hostnames=true \
    allow_localhost=false \
    allow_ip_sans=false \
    server_flag=true \
    client_flag=true

# Cleanup

rm vplmn_intermediate.csr vplmn_intermediate.cert.pem

# Enable AppRole auth method

bao auth enable -path=5g-partners approle

# HPLMN Policy

cat > hplmn-5g-policy.hcl << EOF
path "pki-hplmn/sign/5g-roaming-certs" {
  capabilities = ["create", "update"]
}

path "pki-hplmn/roles/5g-roaming-certs" {
  capabilities = ["read"]
}

path "pki-hplmn/ca/pem" {
  capabilities = ["read"]
}

path "pki-hplmn/cert/ca" {
  capabilities = ["read"]
}

path "pki-root/ca/pem" {
  capabilities = ["read"]
}
EOF

bao policy write hplmn-5g-policy hplmn-5g-policy.hcl

# VPLMN Policy

cat > vplmn-5g-policy.hcl << EOF
path "pki-vplmn/sign/5g-roaming-certs" {
  capabilities = ["create", "update"]
}

path "pki-vplmn/roles/5g-roaming-certs" {
  capabilities = ["read"]
}

path "pki-vplmn/ca/pem" {
  capabilities = ["read"]
}

path "pki-vplmn/cert/ca" {
  capabilities = ["read"]
}

path "pki-root/ca/pem" {
  capabilities = ["read"]
}
EOF

bao policy write vplmn-5g-policy vplmn-5g-policy.hcl

# Create AppRole for HPLMN

bao write auth/5g-partners/role/hplmn-mnc070 \
    token_policies="hplmn-5g-policy" \
    token_ttl=24h \
    token_max_ttl=720h \
    bind_secret_id=true \
    secret_id_ttl=8760h \
    secret_id_num_uses=0

# Create AppRole for VPLMN

bao write auth/5g-partners/role/vplmn-mnc001 \
    token_policies="vplmn-5g-policy" \
    token_ttl=24h \
    token_max_ttl=720h \
    bind_secret_id=true \
    secret_id_ttl=8760h \
    secret_id_num_uses=0

# Cleanup policy files

rm hplmn-5g-policy.hcl vplmn-5g-policy.hcl

# Get HPLMN credentials

HPLMN_ROLE_ID=$(bao read -field=role_id auth/5g-partners/role/hplmn-mnc070/role-id)
HPLMN_SECRET_ID=$(bao write -field=secret_id -f auth/5g-partners/role/hplmn-mnc070/secret-id)

echo "HPLMN MNC070 Credentials:"
echo "Role ID: $HPLMN_ROLE_ID"
echo "Secret ID: $HPLMN_SECRET_ID"
echo "Base64 Role ID: $(echo -n $HPLMN_ROLE_ID | base64)"
echo "Base64 Secret ID: $(echo -n $HPLMN_SECRET_ID | base64)"
echo ""

# Get VPLMN credentials

VPLMN_ROLE_ID=$(bao read -field=role_id auth/5g-partners/role/vplmn-mnc001/role-id)
VPLMN_SECRET_ID=$(bao write -field=secret_id -f auth/5g-partners/role/vplmn-mnc001/secret-id)

echo "VPLMN MNC001 Credentials:"
echo "Role ID: $VPLMN_ROLE_ID"
echo "Secret ID: $VPLMN_SECRET_ID"
echo "Base64 Role ID: $(echo -n $VPLMN_ROLE_ID | base64)"
echo "Base64 Secret ID: $(echo -n $VPLMN_SECRET_ID | base64)"

# HPLMN Partner Kubernetes Configuration:

# hplmn-openbao-secret.yaml

kubectl create ns telco

kubectl apply -f - <<EOF
---

apiVersion: v1
kind: Secret
metadata:
  name: hplmn-openbao-approle
  namespace: telco
type: Opaque
data:
  role_id: MzcwNzA0ODctYTBjNC0xOTY2LTRmMTMtNjc0ZGVmMTQwNTMx

  secret_id: YTliZGQyYTEtMjljMC1kMWI5LTIxODUtMTM1MjAxYmFmNjg4
---

# hplmn-issuer.yaml

apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: hplmn-5g-issuer
  namespace: telco
spec:
  vault:
    server: https://openbao.192.168.4.1.nip.io
    caBundle:  LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURXVENDQWtHZ0F3SUJBZ0lVR2xpc2xjMzNPTnoyR0YwUVVhUXlnS05BeFk0d0RRWUpLb1pJaHZjTkFRRUwKQlFBd0pURWpNQ0VHQTFVRUF3d2FiM0JsYm1KaGJ5NHhPVEl1TVRZNExqUXVNUzV1YVhBdWFXOHdIaGNOTWpVdwpOek14TURBd09EVTVXaGNOTWpZd056TXhNREF3T0RVNVdqQWxNU013SVFZRFZRUUREQnB2Y0dWdVltRnZMakU1Ck1pNHhOamd1TkM0eExtNXBjQzVwYnpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUIKQUp0Qkg0YWVYcTl2UkpHQ2tkWUkxN2FMYlliUDc1V0w1emtvdGUwOHFWczFlV3NDOE9KYnAzNkZaQTV6RWdZRQorWldwRTBGcEZRdy93TEg3SUtEZFF6TWJtNzhnTHUzU2x4V09YWS9hSHQyRnF2L1Z3aThPRHl1Tlh0UVJqckp1CkZkU0ovQVN1WkEyMU5Bb1ZiclpnVU56NkVCMytoT0pMTGhwQUxHS3RkK2k1aXdsWWFVMVJiQjNvaXJzeFhYZk8Kc1JuVk04R0NZUVEvUTlnVlZrWDJ5OEFSUWovRUI2TDRvRVQ3cCtvZWNRcWFZOWRseTN0L1JvdlVuc1FXWDMvegp6alRyNFBKYkVNVWwvSmhWT3M0R0ZQOWNOSGMrWTdCVzBrWGxNa1piNS9CdWptM1BadDBJTk9lWWhXcjEyMkxyCit1VmpMUy83UGxacDNrOWlmb0dCUGFrQ0F3RUFBYU9CZ0RCK01CMEdBMVVkRGdRV0JCU1drc0FwOTVkUUFzT1IKc0tMZkRoQzIzdXcxQ1RBZkJnTlZIU01FR0RBV2dCU1drc0FwOTVkUUFzT1JzS0xmRGhDMjN1dzFDVEFQQmdOVgpIUk1CQWY4RUJUQURBUUgvTUNzR0ExVWRFUVFrTUNLQ0dtOXdaVzVpWVc4dU1Ua3lMakUyT0M0MExqRXVibWx3CkxtbHZod1RBcUFRQk1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQkRKbmlHdEJKa0ZHdERKMlJlNDllZkJxMHcKeTV1Sis5UlVMWXdSdTVCQ2ZsUnB4ZzNWNjl4UHhFaUFUNjNWL2FSeDVZd3BhR1pMcEJkUG9NdnloTlh1bGJvSgowbmxNbFl1eENaaHJIWlNnamJlcnFua01LNzU3ZTlsRUUyS21hZHlGYjNrMEtLZ1FBQTRIMnExZFFxSWZvSStDCmFoY1RJTUphdUJUbmJNRXRCVmVOVGxtd0c5RlV0VkpTYkx2MDd5aGxVSEFWUmdEaXFpOUFjRHU1azFNZkcvOTgKempxTVVETEV0S3BGdEJ3WEpxUzBUekFYZUF5SU9VODBmeWRmZ3JRVjFoZ25xUEdDVjJNY2ZHcXV2VE9RMEZJUwpzeUpSbGZnVWNVbXhzNnpTYnRhS1YzMTdTYVBKUXB5SnpRTXdsZWQrZGRyUXl0c0o5SFkrTFYwdUx1a2sKLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    path: pki-hplmn/sign/5g-roaming-certs
    auth:
      appRole:
        path: 5g-partners
        roleId: 37070487-a0c4-1966-4f13-674def140531
        secretRef:
          name: hplmn-openbao-approle
          key: secret_id
EOF

# VPLMN Partner Kubernetes Configuration:

# vplmn-openbao-secret.yaml

kubectl create ns telco

kubectl apply -f - <<EOF
---

apiVersion: v1
kind: Secret
metadata:
  name: vplmn-openbao-approle
  namespace: telco
type: Opaque
data:
  role_id: NmQ3ZTg3MjMtNGQ5Yi0xMjNlLTc2NjQtOTA3NDk2MjczZWQx

  secret_id: MzcyNmRjNzUtYTBiNS1lMzMyLTU0NjEtOWRkMjJhZjg4NDEz
---

# vplmn-issuer.yaml

apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: vplmn-5g-issuer
  namespace: telco
spec:
  vault:
    server: https://openbao.192.168.4.1.nip.io
    caBundle:  LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURXVENDQWtHZ0F3SUJBZ0lVR2xpc2xjMzNPTnoyR0YwUVVhUXlnS05BeFk0d0RRWUpLb1pJaHZjTkFRRUwKQlFBd0pURWpNQ0VHQTFVRUF3d2FiM0JsYm1KaGJ5NHhPVEl1TVRZNExqUXVNUzV1YVhBdWFXOHdIaGNOTWpVdwpOek14TURBd09EVTVXaGNOTWpZd056TXhNREF3T0RVNVdqQWxNU013SVFZRFZRUUREQnB2Y0dWdVltRnZMakU1Ck1pNHhOamd1TkM0eExtNXBjQzVwYnpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUIKQUp0Qkg0YWVYcTl2UkpHQ2tkWUkxN2FMYlliUDc1V0w1emtvdGUwOHFWczFlV3NDOE9KYnAzNkZaQTV6RWdZRQorWldwRTBGcEZRdy93TEg3SUtEZFF6TWJtNzhnTHUzU2x4V09YWS9hSHQyRnF2L1Z3aThPRHl1Tlh0UVJqckp1CkZkU0ovQVN1WkEyMU5Bb1ZiclpnVU56NkVCMytoT0pMTGhwQUxHS3RkK2k1aXdsWWFVMVJiQjNvaXJzeFhYZk8Kc1JuVk04R0NZUVEvUTlnVlZrWDJ5OEFSUWovRUI2TDRvRVQ3cCtvZWNRcWFZOWRseTN0L1JvdlVuc1FXWDMvegp6alRyNFBKYkVNVWwvSmhWT3M0R0ZQOWNOSGMrWTdCVzBrWGxNa1piNS9CdWptM1BadDBJTk9lWWhXcjEyMkxyCit1VmpMUy83UGxacDNrOWlmb0dCUGFrQ0F3RUFBYU9CZ0RCK01CMEdBMVVkRGdRV0JCU1drc0FwOTVkUUFzT1IKc0tMZkRoQzIzdXcxQ1RBZkJnTlZIU01FR0RBV2dCU1drc0FwOTVkUUFzT1JzS0xmRGhDMjN1dzFDVEFQQmdOVgpIUk1CQWY4RUJUQURBUUgvTUNzR0ExVWRFUVFrTUNLQ0dtOXdaVzVpWVc4dU1Ua3lMakUyT0M0MExqRXVibWx3CkxtbHZod1RBcUFRQk1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQkRKbmlHdEJKa0ZHdERKMlJlNDllZkJxMHcKeTV1Sis5UlVMWXdSdTVCQ2ZsUnB4ZzNWNjl4UHhFaUFUNjNWL2FSeDVZd3BhR1pMcEJkUG9NdnloTlh1bGJvSgowbmxNbFl1eENaaHJIWlNnamJlcnFua01LNzU3ZTlsRUUyS21hZHlGYjNrMEtLZ1FBQTRIMnExZFFxSWZvSStDCmFoY1RJTUphdUJUbmJNRXRCVmVOVGxtd0c5RlV0VkpTYkx2MDd5aGxVSEFWUmdEaXFpOUFjRHU1azFNZkcvOTgKempxTVVETEV0S3BGdEJ3WEpxUzBUekFYZUF5SU9VODBmeWRmZ3JRVjFoZ25xUEdDVjJNY2ZHcXV2VE9RMEZJUwpzeUpSbGZnVWNVbXhzNnpTYnRhS1YzMTdTYVBKUXB5SnpRTXdsZWQrZGRyUXl0c0o5SFkrTFYwdUx1a2sKLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    path: pki-vplmn/sign/5g-roaming-certs
    auth:
      appRole:
        path: 5g-partners
        roleId: 6d7e8723-4d9b-123e-7664-907496273ed1
        secretRef:
          name: vplmn-openbao-approle
          key: secret_id
EOF

# Test Commands:

# Test HPLMN certificate issuance

bao write pki-hplmn/sign/5g-roaming-certs \
    common_name="nrf.5gc.mnc070.mcc999.3gppnetwork.org" \
    alt_names="amf.5gc.mnc070.mcc999.3gppnetwork.org,smf.5gc.mnc070.mcc999.3gppnetwork.org" \
    ttl=1152h

# Test VPLMN certificate issuance

bao write pki-vplmn/sign/5g-roaming-certs \
    common_name="nrf.5gc.mnc001.mcc001.3gppnetwork.org" \
    alt_names="amf.5gc.mnc001.mcc001.3gppnetwork.org,smf.5gc.mnc001.mcc001.3gppnetwork.org" \
    ttl=1152h
```

# Verification commands
```
bao secrets list
bao read pki-root/cert/ca
bao read pki-hplmn/cert/ca
bao read pki-vplmn/cert/ca
bao list auth/5g-partners/role
```