kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        log . {
          class error
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }

    5gc.mnc070.mcc999.3gppnetwork.org:53 {
        hosts {
            192.168.2.50 amf.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.51 bsf.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.52 ausf.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.53 udm.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.54 udr.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.55 nssf.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.56 pcf.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.57 smf.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.58 scp.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.59 sepp-sbi.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.66 sepp.5gc.mnc070.mcc999.3gppnetwork.org
            192.168.2.60 nrf.5gc.mnc070.mcc999.3gppnetwork.org
            fallthrough
        }
    }

    5gc.mnc001.mcc001.3gppnetwork.org:53 {
        hosts {
            192.168.1.50 amf.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.51 bsf.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.52 ausf.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.53 udm.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.54 udr.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.55 nssf.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.56 pcf.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.57 smf.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.58 scp.5gc.mnc001.mcc001.3gppnetwork.org            
            192.168.1.59 sepp-sbi.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.66 sepp.5gc.mnc001.mcc001.3gppnetwork.org
            192.168.1.60 nrf.5gc.mnc001.mcc001.3gppnetwork.org
            fallthrough
        }
    }
EOF

kubectl -n kube-system rollout restart deployment coredns

kubectl run netshoot --rm -it --image=nicolaka/netshoot --restart=Never -- bash
