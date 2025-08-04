## Home VM

Kernel Multus Interface: ens21

**VLANs:**
    100 - N3/N9
    200 - N2
    300 - SBI
    400 - N4
    500 - SEPP-Control
    600 - SEPP-Forward
    700 - K8S-POD-NW-TO-IPX-LB

**N3/N9 - 192.168.1.0/28**
    GW: 192.168.1.1
    UPF: 192.168.1.2
    gNB: 192.168.1.3

**N2 - 192.168.1.16/28**
    GW: 192.168.1.17
    AMF: 192.168.1.18
    gNB: 192.168.1.19

**N4 - 192.168.1.32/28**
    GW: 192.168.1.33
    SMF: 192.168.1.34
    UPF: 192.168.1.35

**SBI - 192.168.1.48/28**
    GW: 192.168.1.49
    AMF: 192.168.1.50
    BSF: 192.168.1.51
    AUSF: 192.168.1.52
    UDM: 192.168.1.53
    UDR: 192.168.1.54
    NSSF: 192.168.1.55
    PCF: 192.168.1.56
    SMF: 192.168.1.57
    SCP: 192.168.1.58
    SEPP: 192.168.1.59
    NRF: 192.168.1.60
    MONGODB: 192.168.1.61

**SEPP-Control - 192.168.1.64/28**
    GW: 192.168.1.65
    SEPP: 192.168.1.66

**SEPP-Forward - 192.168.1.80/28**
    GW: 192.168.1.81
    SEPP: 192.168.1.82

**K8S-POD-NW-TO-IPX-LB - 192.168.1.96/28**
    GW: 192.168.1.97
    InternalPodNetwork: 192.168.1.98

## Netplan WorkerNode

```
# This file is generated from information provided by the datasource. Changes
# to it will not persist across an instance reboot. To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
  version: 2
  ethernets:
    ens21:
      dhcp4: false
    eth0:
      dhcp4: true
      match:
        macaddress: bc:24:11:3d:ae:42
      set-name: eth0

  vlans:
    vlan100:
      id: 100
      link: ens21
      dhcp4: no
      optional: true
    vlan200:
      id: 200
      link: ens21
      dhcp4: no
      optional: true
    vlan300:
      id: 300
      link: ens21
      dhcp4: no
      optional: true
    vlan400:
      id: 400
      link: ens21
      dhcp4: no
      optional: true
    vlan500:
      id: 500
      link: ens21
      dhcp4: no
      optional: true
    vlan600:
      id: 600
      link: ens21
      dhcp4: no
      optional: true
    vlan700:
      id: 700
      link: ens21
      dhcp4: no
      addresses: [192.168.1.98/28]
      routes:
        - to: 192.168.4.0/24
          via: 192.168.1.97
      optional: true
```

## Netplan PackerRusher VM

```
# This file is generated from information provided by the datasource. Changes    
# to it will not persist across an instance reboot. To disable cloud-init's    
# network configuration capabilities, write a file  
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:  
# network: {config: disabled}   
# This file is generated from information provided by the datasource.  Changes   
# to it will not persist across an instance reboot.  To disable cloud-init's   
# network configuration capabilities, write a file   
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:  
# network: {config: disabled}
network:
  version: 2
  ethernets:
      eth0:
          dhcp4: true
          match:
              macaddress: bc:24:11:be:ad:db
          set-name: eth0
      eth1:
          match:
              macaddress: bc:24:11:d5:95:c3
          set-name: eth1
      eth2:
          match:
              macaddress: bc:24:11:1a:1d:95
          set-name: eth2
  vlans:
   vlan100:
    id: 100
    link: eth2
    dhcp4: no
    dhcp4: no
    addresses: [192.168.1.3/28]
    optional: true
   vlan200:
    id: 200
    link: eth1
    dhcp4: no
    dhcp4: no
    addresses: [192.168.1.19/28]
    optional: true
```  