Openshift Core Services
=================================

Revision de todos los servicios core de openshift networking, dns, pki, etc.

# Tabla de Contenido

## Dia X
1. [Openshift Core Services](#0)
    1. [Openshift DNS Operator](#1)
    2. [Openshift Network Operator](#2)

## 1.1. Openshift DNS Operator<a name="1"></a>

El operador de DNS despliega y adminsitra CoreDNS para proveer el servicio de resolucion de nombres de pods y servicios en Openshift.

### DNS Operator

DNS Operator implementa la extension de la API con el recurso `dns`. El Operador DNS despliega CoreDNS usando DaemonSets, crea el servicio de DaemonSets y configura kubelet para que los pods puedan utilizar CoreDNS para resolver nombres.

El Operador DNS es desplegado cuando es realizada la instalacion.

Vemos el estado del deploy del operador.
```
oc get -n openshift-dns-operator deployment/dns-operator
```

Vemos el estado del operador
```
oc get clusteroperator/dns
```

### Default DNS

Todo openshift posee la instancia `default` configurada por el dns.operator.

Detalle de la configuracion.
```
oc describe dns.operator/default
...
Cluster Domain:  cluster.local  >> Dominio del cluster
Cluster IP:      172.30.0.10    >> Ip de servicio de cluster
```

Para ver el CIDR de los servicios
```
oc get networks.config/cluster -o jsonpath='{$.status.serviceNetwork}'
```

### Using DNS Forwarding

Se puede usar DNS Forwarind para sobreescribir el archivo de configuracion etc/resolv.conf.

Modificamos el operador de dns default
```
oc edit dns.operator/default
```

Puede agregarse una configuracion como la que sigue
```
apiVersion: operator.openshift.io/v1
kind: DNS
metadata:
  name: default
spec:
  servers:
  - name: foo-server 
    zones: 
      - foo.com
    forwardPlugin:
      upstreams: 
        - 1.1.1.1
        - 2.2.2.2:5353
  - name: bar-server
    zones:
      - bar.com
      - example.com
    forwardPlugin:
      upstreams:
        - 3.3.3.3
        - 4.4.4.4:5454
```

Editar el configmaps
```
oc get configmap/dns-default -n openshift-dns -o yaml
```

El contenido puede ser el siguiente
```
apiVersion: v1
data:
  Corefile: |
    foo.com:5353 {
        forward . 1.1.1.1 2.2.2.2:5353
    }
    bar.com:5353 example.com:5353 {
        forward . 3.3.3.3 4.4.4.4:5454 
    }
    .:5353 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            policy sequential
        }
        cache 30
        reload
    }
kind: ConfigMap
metadata:
  labels:
    dns.operator.openshift.io/owning-dns: default
  name: dns-default
  namespace: openshift-dns
```

### Ver estado de operador
```
oc describe clusteroperators/dns
```

### Ver logs de operador
```
oc logs --namespace=openshift-dns-operator deployment/dns-operator
```

### Troubleshooting Openshift 4.x DNS
https://access.redhat.com/solutions/3804501

#### Pasos para el diagnostico

1. Check de estado de operador
```
oc get clusteroperator dns
```

2. Check pods openshift-dns-operator
```
oc get all  -n openshift-dns-operator
```

3. Check logs del pod operador
```
 oc logs pod/$(oc get pods -o=jsonpath="{.items[0].metadata.name}" -n openshift-dns-operator) -n openshift-dns-operator --all-containers=true
```
4. Check de los componentes en el despliegue del operador de dns
```
oc get all  -n openshift-dns
```

5. Check pod resolver
```
export PODS=$(oc get pods -o=jsonpath="{.items[*].metadata.name}" -n openshift-apiserver)
for i in $(echo $PODS) ; do oc exec $i -n openshift-apiserver -- cat /etc/resolv.conf ; done
```

6. Check de CoreDNS container logs
```
mkdir /tmp/coredns && cd /tmp/coredns
export PODS=$(oc get pods -o=jsonpath="{.items[*].metadata.name}" -n openshift-dns)
for i in $(echo $PODS) ; do echo "-- $i --" ; oc logs $i -c dns -n openshift-dns > $i.log ; done
```
7. Testing resolve kuberntes service hostname
```
oc get pods -n openshift-dns -o custom-columns="Pod Name:.metadata.name,Pod IP:.status.podIP,Node IP:.status.hostIP,Status:.status.phase"
```
```
for dnspod in `oc get pods -n openshift-dns -o name --no-headers`; do echo "Testing $dnspod"; for dnsip in `oc get pods -n openshift-dns -o go-template='{{ range .items }} {{index .status.podIP }} {{end}}'`; do echo -e "\tMaking query to $dnsip"; oc exec -n openshift-dns $dnspod -- dig @$dnsip kubernetes.default.svc.cluster.local -p 5353 +short 2>/dev/null | sed 's/^/\t/'; done; done
```

8. Test resolving external queries (google.com)
```
for dnspod in `oc get pods -n openshift-dns -o name --no-headers`; do echo "Testing $dnspod"; for dnsip in `oc get pods -n openshift-dns -o go-template='{{ range .items }} {{index .status.podIP }} {{end}}'`; do echo -e "\t Making query to $dnsip"; oc exec -n openshift-dns $dnspod -- dig @$dnsip redhat.com -p 5353 +short 2>/dev/null; done; done
```

9. Chequemos el tiempo de busqueda vs el tiempo de solicitud total
```
$ echo $pod
pod-example-5f78c768b-cg88c
$ oc exec $pod  -- bash -c 'while true; do echo -n "$(date)  "; curl -s -o /dev/null -w "%{time_namelookup} %{time_total} %{http_code}\n" https://www.redhat.com -k; sleep 10; done'
$ oc exec $pod -- bash -c 'while true; do echo -n "$(date)  "; curl -s -o /dev/null -w "%{time_namelookup} %{time_total} %{http_code}\n" -4 https://www.redhat.com -k; sleep 10; done'
$ oc exec $pod -- bash -c 'while true; do echo -n "$(date)  "; curl -s -o /dev/null -w "%{time_namelookup} %{time_total} %{http_code}\n" -6 https://www.redhat.com -k; sleep 10; done'
```


## 1.2. Openshift Network Operator<a name="1"></a>

Ver deployment del operador
```
oc get -n openshift-network-operator deployment/network-operator
```

Ver el estado del operador
```
oc get clusteroperator/network
```

Ver la configuracion del operador
```
oc describe network.config/cluster\
```

## Links de referencia
- Corefile for adding additional nameserver to CoreDNS configuration file in OCP 4
    https://access.redhat.com/solutions/4765861

- How to collect TCP dump from OCP 4 CoreOS nodes and pods running on it ?
    https://access.redhat.com/solutions/4537671

- Actualizar nameserver en OCP4
    https://access.redhat.com/solutions/4518671

- Troubleshooting Openshift 4.x DNS
    https://access.redhat.com/solutions/3804501