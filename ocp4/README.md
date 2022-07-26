# OpenShift 4.3 and CIS

Environment parameters

* OCP 4.3
* CIS 2.0
* AS3: 3.20
* BIG-IP 15.1.0.2

## OpenShift 4.3


**Example Topology**

                                            BIGIP NW                                                                 CLUSTER NW
                                Openshift (Self)       Internal Self                                      Openshift Node.        Backend Pod
                                10.131.2.15            10.192.74.189  ------------------------------------10.192.74.177          10.131.0.185
 
```
❯ oc get hostsubnets
NAME                                 HOST                                 HOST IP         SUBNET          EGRESS CIDRS   EGRESS IPS
master0.ocp4-cluster-001.bddemo.io   master0.ocp4-cluster-001.bddemo.io   10.192.74.174   10.129.0.0/23
master1.ocp4-cluster-001.bddemo.io   master1.ocp4-cluster-001.bddemo.io   10.192.74.175   10.130.0.0/23
master2.ocp4-cluster-001.bddemo.io   master2.ocp4-cluster-001.bddemo.io   10.192.74.176   10.128.0.0/23
worker0.ocp4-cluster-001.bddemo.io   worker0.ocp4-cluster-001.bddemo.io   10.192.74.177   10.131.0.0/23
worker1.ocp4-cluster-001.bddemo.io   worker1.ocp4-cluster-001.bddemo.io   10.192.74.178   10.128.2.0/23
```

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIGIP. Follow the link to install AS3
 
* Install AS3 on BIGIP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

## BIG-IP Controller -  Initial Set-Up

### Create a new OpenShift HostSubnet

Create a host subnet for the BIPIP. This will provide the subnet for creating the tunnel self-IP

```
oc create -f f5-bigip-hostsubnet.yaml
```

```
❯ oc get hostsubnets
NAME                                 HOST                                 HOST IP         SUBNET          EGRESS CIDRS   EGRESS IPS
master0.ocp4-cluster-001.bddemo.io   master0.ocp4-cluster-001.bddemo.io   10.192.74.174   10.129.0.0/23
master1.ocp4-cluster-001.bddemo.io   master1.ocp4-cluster-001.bddemo.io   10.192.74.175   10.130.0.0/23
master2.ocp4-cluster-001.bddemo.io   master2.ocp4-cluster-001.bddemo.io   10.192.74.176   10.128.0.0/23
openshift-f5-node                    openshift-f5-node                    10.192.74.189   10.131.2.0/23
worker0.ocp4-cluster-001.bddemo.io   worker0.ocp4-cluster-001.bddemo.io   10.192.74.177   10.131.0.0/23
worker1.ocp4-cluster-001.bddemo.io   worker1.ocp4-cluster-001.bddemo.io   10.192.74.178   10.128.2.0/23
```

### Create a new partition on your BIG-IP system
```
(tmos)# create auth partition openshift
```
This needs to match the partition in the controller configuration


### Create a BIG-IP VXLAN tunnel


#### create net tunnels vxlan vxlan-mp flooding-type multipoint
```
(tmos)# create net tunnels vxlan vxlan-mp flooding-type multipoint
(tmos)# create net tunnels tunnel openshift-vxlan key 0 profile vxlan-mp local-address 10.192.74.189
```
#### Add the BIG-IP device to the OpenShift overlay network
```
(tmos)# create net self openshift-self address 10.131.2.15/14 allow-service all vlan openshift-vxlan
```
Subnet comes from the creating the hostsubnets. Used .83 to be consistent with BigIP internal interface


## Deploying the BIG-IP Controller
### Create CIS Controller, BIG-IP credentials and RBAC Authentication
```
oc create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=bigip123
oc create serviceaccount bigip-ctlr -n kube-system
oc create -f f5-kctlr-openshift-clusterrole.yaml
oc create -f f5-cluster-deployment.yaml
```

```
args: [
        "--bigip-username=$(BIGIP_USERNAME)",
        "--bigip-password=$(BIGIP_PASSWORD)",
        # Replace with the IP address or hostname of your BIG-IP device
        "--bigip-url=192.168.200.83",
        # Replace with the name of the BIG-IP partition you want to manage
        "--bigip-partition=openshift",
        "--pool-member-type=cluster",
        # Replace with the path to the BIG-IP VXLAN connected to the
        # OpenShift HostSubnet
        "--openshift-sdn-name=/Common/openshift-vxlan",
        "--manage-routes=true",
        "--namespace=f5demo",
        "--route-vserver-addr=10.192.75.107",
        "--log-level=DEBUG",
        # Self-signed cert
        "--insecure=true",
        "--agent=as3"
       ]
```
```
oc create secret generic bigip-login --namespace kube-system --from-literal=username=admin --from-literal=password=f5PME123
oc create serviceaccount bigip-ctlr -n kube-system
oc create -f f5-kctlr-openshift-clusterrole.yaml
oc create -f f5-k8s-bigip-ctlr-openshift.yaml
oc adm policy add-cluster-role-to-user cluster-admin -z bigip-ctlr -n kube-system
```
## Delete kubernetes bigip container connecter, authentication and RBAC
```
oc delete serviceaccount bigip-ctlr -n kube-system
oc delete -f f5-kctlr-openshift-clusterrole.yaml
oc delete -f f5-k8s-bigip-ctlr-openshift.yaml
oc delete secret bigip-login -n kube-system
```
## Create container f5-demo-app-route
```
oc create -f f5-demo-app-route-deployment.yaml -n f5demo
oc create -f f5-demo-app-route-service.yaml -n f5demo
oc create -f f5-demo-app-route-basic.yaml -n f5demo
oc create -f f5-demo-app-route-balance.yaml -n f5demo
oc create -f f5-demo-app-route-edge-ssl.yaml -n f5demo
oc create -f f5-demo-app-route-reencrypt-ssl.yaml -n f5demo
oc create -f f5-demo-app-route-passthrough-ssl.yaml -n f5demo
oc create -f f5-demo-app-route-waf.yaml -n f5demo
oc create -f f5-demo-app-route-ab.yaml -n f5demo
```
Please look for example files in my repo

## Delete container f5-demo-app-route
```
oc delete -f f5-demo-app-route-deployment.yaml -n f5demo
oc delete -f f5-demo-app-route-service.yaml -n f5demo
oc delete -f f5-demo-app-route-basic.yaml -n f5demo
oc delete -f f5-demo-app-route-balance.yaml -n f5demo
oc delete -f f5-demo-app-route-edge-ssl.yaml -n f5demo
oc delete -f f5-demo-app-route-reencrypt-ssl.yaml -n f5demo
oc delete -f f5-demo-app-route-passthrough-ssl.yaml -n f5demo
oc delete -f f5-demo-app-route-waf.yaml -n f5demo
oc delete -f f5-demo-app-route-ab.yaml -n f5demo
``` 
## Enable logging for AS3
```
oc get pod -n kube-system
oc log -f f5-server-### -n kube-system | grep -i 'as3'
```

## Deleting a BIG-IP partition
```
# bash
# rm /config/partitions/ocp_AS3/bigip.conf
# rm -r /config/partitions/ocp

tmsh load sys config

# tmsh delete sys folder /ocp_AS3/Shared
# tmsh delete sys folder /ocp_AS3
# tmsh delete sys folder /ocp
```