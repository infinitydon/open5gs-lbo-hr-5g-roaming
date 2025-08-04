## General

```
mkdir -p /var/snap/microk8s/common/

cat <<EOF > /var/snap/microk8s/common/.microk8s.yaml
---

version: 0.1.0
addons:

  - name: dns
  - name: community
  - name: multus
  - name: hostpath-storage
    extraKubeletArgs:
      --cluster-domain: cluster.local
      --cluster-dns: 10.152.183.10
      --cpu-manager-policy: "static"
      --reserved-cpus: "0-2"
EOF
```

```
snap install microk8s --classic --channel=1.32/stable

sudo usermod -a -G microk8s $USER
mkdir -p ~/.kube
chmod 0700 ~/.kube

newgrp microk8s

microk8s config > ~/.kube/config

microk8s enable hostpath-storage

microk8s enable community

microk8s enable multus

curl -LO https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

echo 'source <(kubectl completion bash)' >>~/.bashrc

source ~/.bashrc

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

curl -L https://github.com/derailed/k9s/releases/download/v0.50.9/k9s_Linux_amd64.tar.gz | tar xz
sudo mv k9s /usr/local/bin/
sudo chmod +x /usr/local/bin/k9s
```



## IPX Roaming Cluster

```
wget https://github.com/DPDK/dpdk/raw/main/usertools/dpdk-devbind.py -O /usr/local/bin/dpdk-devbind.py

chmod +x /usr/local/bin/dpdk-devbind.py

sed -i 's/^GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="default_hugepagesz=1GB hugepagesz=1G hugepages=4G"/' /etc/default/grub

update-grub

mkdir /opt/dpdk/

cat <<EOF > /usr/lib/systemd/system/config-sriov.service
[Unit]
Description=SR-IOV configuration
DefaultDependencies=no
After=network-online.target
Before=kubelet.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash /opt/dpdk/config-sriov.sh

[Install]
WantedBy=sysinit.target
EOF

cat <<EOF > /opt/dpdk/config-sriov.sh
  #!/bin/bash
  echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
  /usr/local/bin/dpdk-devbind.py -b vfio-pci 0000:00:14.0
  /usr/local/bin/dpdk-devbind.py -b vfio-pci 0000:00:15.0
  /usr/local/bin/dpdk-devbind.py -b vfio-pci 0000:00:16.0
EOF

chmod +x /opt/dpdk/config-sriov.sh

systemctl enable config-sriov
```



## Visiting/Home Roaming Cluster

```
wget https://github.com/DPDK/dpdk/raw/main/usertools/dpdk-devbind.py -O /usr/local/bin/dpdk-devbind.py

chmod +x /usr/local/bin/dpdk-devbind.py

sed -i 's/^GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="default_hugepagesz=1GB hugepagesz=1G hugepages=4G"/' /etc/default/grub

update-grub

mkdir /opt/dpdk/

cat <<EOF > /usr/lib/systemd/system/config-sriov.service
[Unit]
Description=SR-IOV configuration
DefaultDependencies=no
After=network-online.target
Before=kubelet.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash /opt/dpdk/config-sriov.sh

[Install]
WantedBy=sysinit.target
EOF

cat <<EOF > /opt/dpdk/config-sriov.sh
  #!/bin/bash
  echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
  /usr/local/bin/dpdk-devbind.py -b vfio-pci 0000:00:14.0
  /usr/local/bin/dpdk-devbind.py -b vfio-pci 0000:00:15.0
EOF

chmod +x /opt/dpdk/config-sriov.sh

systemctl enable config-sriov
```



Sample helm commands

```
helm -n roaming upgrade --install ipx-gateway ipx-vpp-vpn-ikve2-chart/ --create-namespace

helm -n roaming upgrade --install hplmn-edge-gateway hUPF-vpp-vpn-ikve2-chart/ --create-namespace

helm -n roaming upgrade --install vplmn-edge-gateway vUPF-vpp-vpn-ikve2-chart/ --create-namespace
```

N.B - Another interface is needed in the home and visiting workerNode with IPs in the internal network of the VPP gateway side and a static route 
of 192.168.4.1 (openbao ingress controller LB IP) pointing to the VPP GW as the nexthop