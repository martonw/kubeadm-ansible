## Create POD ip pool without NAT

To use calicoctl, first port-forward the calico etcd:
```
kubectl port-forward svc/calico-etcd -n kube-system 6666:6666
```
then use calicoctl as follows:
```
ETCD_ENDPOINTS=http://localhost:6666 calicoctl version
```

Configure a new ip pool:
```YAML
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: test-pool
spec:
  cidr: 192.168.39.0/24
  ipipMode: Always
  natOutgoing: false

```
and then deploy it:
```
ETCD_ENDPOINTS=http://192.168.36.136:6666 ./calicoctl create -f ippool.v2
```


To create a namespace, annotated with the ippool:
```YAML
apiVersion: v1
kind: Namespace
metadata:
  name: test1
  annotations:
    "cni.projectcalico.org/ipv4pools": "[\"test-pool\"]"

spec:
  finalizers:
  - kubernetes
```

then:
```
kubectl create -f test1-ns.yaml
```

To run a temporary alpine with interactive shell (in test1 namespace):
```
kubectl run -it --rm debug --image=alpine --restart=Never -n test1 -- sh
```

When useing the default namespace, and thus, the default ippool, the `ping 192.168.35.1` command shall succeed:
22:29:51.846367 IP 192.168.35.11 > MWCS: ICMP echo request, id 0, seq 0, length 64
22:29:51.846505 IP MWCS > 192.168.35.11: ICMP echo reply, id 0, seq 0, length 64

while, using the test1 namespace, the same command shall output:
22:15:04.574499 IP 192.168.39.192 > MWCS: ICMP echo request, id 2048, seq 0, length 64
22:15:05.580001 IP 192.168.39.192 > MWCS: ICMP echo request, id 2048, seq 1, length 64

The above is an output of the `sudo tcpdump -i vboxnet1 icmp` command.

So, there is no return path for the IP packets - however, the traffic is sourced
from the POD IP address directly. The backward IP route is missing, since there is
no BGP at this point that would announce the POD IP to networks outside the cluster.


## Experimental

BGP default config:
```YAML
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  asNumber: 63400
```

then provision with `ETCD_ENDPOINTS=http://192.168.36.136:6666 ./calicoctl create -f bgp_peer`.

BGP peer config:
```YAML
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-35-1
spec:
  peerIP: 192.168.35.1
  asNumber: 64567
```

provisioning is similar as above. To check the results, use the command:
```
ETCD_ENDPOINTS=http://192.168.36.136:6666 ./calicoctl get bgppeer
```


# Bird configuration
This configuration shall be deployed in the physical host (192.168.35.1),
representing external parties from the cluster's point of view.

to install bird:
```
apt install bird
```

configuration file:
```
# Configure logging
log syslog { debug, trace, info, remote, warning, error, auth, fatal, bug };
log stderr all;

# Override router ID
router id 192.168.35.1;


filter import_kernel {
if ( net != 0.0.0.0/0 ) then {
   accept;
   }
reject;
}

# Turn on global debugging of all protocols
debug protocols all;

# This pseudo-protocol watches all interface up/down events.
protocol device {
  scan time 2;    # Scan interfaces every 2 seconds
}

protocol kernel {
  export all;
}

protocol bgp n1 {
  description "192.168.35.10";
  local as 64567;
  neighbor 192.168.35.10 as 63400;
  next hop self;
#  rr client;
  graceful restart;
  import all;
  export all;
}

protocol bgp n2 {
  description "192.168.35.11";
  local as 64567;
  neighbor 192.168.35.11 as 63400;
  next hop self;
#  rr client;
  graceful restart;
  import all;
  export all;
}


protocol bgp n3 {
  description "192.168.35.12";
  local as 64567;
  neighbor 192.168.35.12 as 63400;
  next hop self;
#  rr client;
  graceful restart;
  import all;
  export all;
}

```

warning: the above configuration file will write a log of trace level logs into
the syslog.

Restart bird service after config update.
