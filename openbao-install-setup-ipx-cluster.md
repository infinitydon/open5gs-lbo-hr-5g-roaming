## MetaLLB Install

helm repo add metallb https://metallb.github.io/metallb
helm -n kube-system install metallb metallb/metallb

## Setup routes towards the VPP Gateway since metallb does not receive advertized routes
sudo ip r add 192.168.1.0/24 via 192.168.3.1
sudo ip r add 192.168.2.0/24 via 192.168.3.1

### BGP Peer to the VPP
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: ipx-roaming-gateway
  namespace: kube-system
spec:
  myASN: 64513
  peerASN: 64513
  peerAddress: 192.168.3.1
  peerPort: 179
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: roaming-lb-pool
  namespace: kube-system
spec:
  addresses:
  - 192.168.4.1/32
  - 192.168.4.2/32
  - 192.168.4.3/32
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: roaming
  namespace: kube-system
spec:
  ipAddressPools:
  - roaming-lb-pool
EOF

## Install Nginx ingress controller
# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller with LoadBalancer (default service type)
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

## Install openbao

kubectl create ns openbao

# Get LoadBalancer IP first

LB_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Generate self-signed certificate including both LB_IP and 192.168.4.1
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -out openbao-tls.crt -keyout openbao-tls.key \
  -subj "/CN=openbao.${LB_IP}.nip.io" \
  -addext "subjectAltName=DNS:openbao.${LB_IP}.nip.io,IP:192.168.4.1"

# Create TLS secret
kubectl create secret tls openbao-tls \
  --cert=openbao-tls.crt \
  --key=openbao-tls.key \
  --namespace openbao

# Install with TLS
helm upgrade --install openbao openbao/openbao \
  --set='server.service.type=ClusterIP' \
  --set='ui.enabled=true' \
  --set='server.dataStorage.enabled=true' \
  --set='server.dataStorage.size=10Gi' \
  --set='server.ingress.enabled=true' \
  --set='server.ingress.ingressClassName=nginx' \
  --set="server.ingress.hosts[0].host=openbao.${LB_IP}.nip.io" \
  --set='server.ingress.tls[0].secretName=openbao-tls' \
  --set="server.ingress.tls[0].hosts[0]=openbao.${LB_IP}.nip.io" \
  --namespace openbao \
  --create-namespace

kubectl exec -n openbao openbao-0 -- bao operator init
Unseal Key 1: uzUNYQwnFV1PWpA+GwLlq8HFYnmxsfUxjhZJAeRUNQkK
Unseal Key 2: zWg2wihnGiftWVmoulxRln5Of+Pz843fgI40vdzfEVwY
Unseal Key 3: e03gcuQeNjSX8cdWIFkWNpZAXSVtcaxdln9sUNlzEhR+
Unseal Key 4: oNDo/MGtD2Jkhk8GtpEPFPUV4RQ+JeSZ7poWpP6DQhPh
Unseal Key 5: 4BGHPEK4XU9zYTkwVij/h0xgKSPIIlb+geIN87qUJQ0t

Initial Root Token: s.CrabD7y3pFTRUXwa6g2yD2B2  

# Check seal status
kubectl exec -n openbao openbao-0 -- bao status

# Unseal with first key
kubectl exec -n openbao openbao-0 -- bao operator unseal <UNSEAL_KEY_1>

# Unseal with second key
kubectl exec -n openbao openbao-0 -- bao operator unseal <UNSEAL_KEY_2>

# Unseal with third key (reaches threshold)
kubectl exec -n openbao openbao-0 -- bao operator unseal <UNSEAL_KEY_3>

curl -s -k https://openbao.192.168.4.1.nip.io/v1/sys/health | jq
{
  "initialized": true,
  "sealed": false,
  "standby": false,
  "performance_standby": false,
  "replication_performance_mode": "disabled",
  "replication_dr_mode": "disabled",
  "server_time_utc": 1753897776,
  "version": "2.3.1",
  "cluster_name": "vault-cluster-1ac2bd52",
  "cluster_id": "b34bcdb3-7877-6e87-1d93-ef93f085e363"
}


hplmn domain: *.5gc.mnc070.mcc999.3gppnetwork.org

vplmn domain: *.5gc.mnc01.mcc001.3gppnetwork.org

## CLI install

wget https://github.com/openbao/openbao/releases/download/v2.3.1/bao_2.3.1_Linux_x86_64.tar.gz

tar -xzf bao_2.3.1_Linux_x86_64.tar.gz

sudo mv bao /usr/local/bin/