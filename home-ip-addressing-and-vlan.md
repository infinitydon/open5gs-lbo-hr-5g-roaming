## Home VM

**Kernel Multus Interface: ens21**

**VLANs:**
    100 - N3/N9
    200 - N2
    300 - SBI
    400 - N4
    500 - SEPP-Control
    600 - SEPP-Forward
    700 - K8S-POD-NW-TO-IPX-LB

**N3/N9 - 192.168.2.0/28**
    GW: 192.168.2.1
    UPF: 192.168.2.2
    gNB: 192.168.2.3

**N2 - 192.168.2.16/28**
    GW: 192.168.2.17
    AMF: 192.168.2.18
    gNB: 192.168.2.19

**N4 - 192.168.2.32/28**
    GW: 192.168.2.33
    SMF: 192.168.2.34
    UPF: 192.168.2.35

**SBI - 192.168.2.48/28**
    GW: 192.168.2.49
    AMF: 192.168.2.50
    BSF: 192.168.2.51
    AUSF: 192.168.2.52
    UDM: 192.168.2.53
    UDR: 192.168.2.54
    NSSF: 192.168.2.55
    PCF: 192.168.2.56
    SMF: 192.168.2.57
    SCP: 192.168.2.58
    SEPP: 192.168.2.59
    NRF: 192.168.2.60
    MONGODB: 192.168.2.61

**SEPP-Control - 192.168.2.64/28**
    GW: 192.168.2.65
    SEPP: 192.168.2.66

**SEPP-Forward - 192.168.2.80/28**
    GW: 192.168.2.81
    SEPP: 192.168.2.82

**K8S-POD-NW-TO-IPX-LB - 192.168.2.96/28**
    GW: 192.168.2.97
    InternalPodNetwork: 192.168.2.98

## Netplan

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
        macaddress: bc:24:11:9c:16:a8
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
      addresses: [192.168.2.98/28]
      routes:
        - to: 192.168.4.0/24
          via: 192.168.2.97
      optional: true
```      