## Reference
* [SDN](https://docs.openshift.org/latest/architecture/additional_concepts/sdn.html)
* [Router](https://docs.openshift.org/latest/install_config/router/default_haproxy_router.html)
* [Integrating External Services](https://docs.openshift.org/latest/dev_guide/integrating_external_services.html )  
* [Troubleshooting](https://docs.openshift.org/latest/admin_guide/sdn_troubleshooting.html)
* [Route](https://docs.openshift.org/latest/architecture/core_concepts/routes.html)    
* [HA](https://docs.openshift.org/latest/admin_guide/high_availability.html#admin-guide-high-availability)
* [External Access ](https://docs.openshift.org/latest/install_config/routing_from_edge_lb.html#install-config-routing-from-edge-lb)

## OpenShift SDN
### Overview
OpenShift Origin uses a software-defined networking (SDN) approach to provide a unified cluster network that enables communication between pods across the OpenShift Origin cluster. This pod network is established and maintained by the OpenShift Origin SDN, which configures an overlay network using Open vSwitch (OVS)

Two SDN plug-ins:
* ovs-subnet:provides a "flat" pod network where every pod can communicate with every other pod and service.
* ovs-multitenant: provides OpenShift Origin project level isolation for pods and services. Each project receives a unique Virtual Network ID (VNID) that identifies traffic from pods assigned to the project. 


> VNID 0 are more privileged in that they are allowed to communicate with all other pods, and all other pods can communicate with them.
> default project has VNID 0. This facilitates certain services like the load balancer, etc. to communicate with all other pods in the cluster and vice versa.

Check VNIDs by running:

	$ oc get netnamespace

### Configuration
Master:/etc/origin/master/master-config.yaml 

	networkConfig:
	  clusterNetworkCIDR: 10.128.0.0/14 
	  hostSubnetLength: 9 
	  networkPluginName: "redhat/openshift-ovs-subnet" 
	  #networkPluginName: "redhat/openshift-ovs-multitenant" 	
	  serviceNetworkCIDR: 172.30.0.0/16  

Nodes:/etc/origin/node/node-config.yaml 

	networkConfig:
	  mtu: 1450 
	  networkPluginName: "redhat/openshift-ovs-subnet" 

> Migrating Between Openshift SDN Plug-ins
>* Change **networkPluginName** on master and nodes
>* Restart master and nodes
>* Migrating OpenShift Origin SDN plug-in to a third-party plug-in 
>
>       $ oc delete clusternetwork --all
>       $ oc delete hostsubnets --all
>       $ oc delete netnamespaces --all


### Design on Masters
Maintain registry of nodes in **etcd** for allocation and deletion of subnet.
When using the ovs-multitenant plug-in, the OpenShift Origin SDN master also watches for the creation and deletion of projects, and assigns VXLAN VNIDs to them, which will be used later by the nodes to isolate traffic correctly.

### Design on Nodes

* br0, the OVS bridge device that pod containers will be attached to. OpenShift Origin SDN also configures a set of non-subnet-specific flow rules on this bridge. The ovs-multitenant plug-in does this immediately.
* lbr0, a Linux bridge device, which is configured as the Docker service’s bridge and given the cluster subnet gateway address (eg, 10.128.x.1/23).
* tun0, an OVS internal port (port 2 on br0). This also gets assigned the cluster subnet gateway address, and is used for external network access. OpenShift Origin SDN configures netfilter and routing rules to enable access from the cluster subnet to the external network via NAT.
* vlinuxbr and vovsbr, two Linux peer virtual Ethernet interfaces. vlinuxbr is added to lbr0 and vovsbr is added to br0 (port 9 with the ovs-subnet plug-in and port 3 with the ovs-multitenant plug-in) to provide connectivity for containers created directly with the Docker service outside of OpenShift Origin.
* vxlan0, the OVS VXLAN device (port 1 on br0), which provides access to containers on remote nodes.
* vethX (in the main netns): A Linux virtual ethernet peer of eth0 in the docker netns. It will be attached to the OVS bridge on one of the other ports.
![SDN-Flows](https://raw.githubusercontent.com/wodeamd/MarkdownPhotos/master/SDN-Flows-Inside-Node.png)

There are four different places the SDN connects (inside a node). They are labeled in red on the diagram above.

* Pod: Traffic is going from one pod to another on the same machine (1 to a different 1)

* Remote Node (or Pod): Traffic is going from a local pod to a remote node or pod in the same cluster (1 to 2)

* External Machine: Traffic is going from a local pod outside the cluster (1 to 3)

* Local Docker: Traffic is going from a local pod to a local container that is not managed by Kubernetes (1 to 4)

Of course the opposite traffic flows are also possible.

### Packet Flow
Suppose you have two containers, A and B, where the peer virtual Ethernet device for container A’s eth0 is named vethA and the peer for container B’s eth0 is named vethB.
Container A to container B on same host:

	eth0 (in A's netns) -> vethA -> br0 -> vethB -> eth0 (in B's netns)

Container A to container B on different hosts:

	eth0 (in A's netns) -> vethA -> br0 -> vxlan0 -> network [1] -> vxlan0 -> br0 -> vethB -> eth0 (in B's netns)

Container A to external host

	eth0 (in A's netns) -> vethA -> br0 -> tun0 -> (NAT) -> eth0 (physical device) -> Internet

## HAProxy Router
The OpenShift Origin router is the ingress point for all external traffic destined for services in your OpenShift Origin installation

The following diagram illustrates how data flows from the master through the plug-in and finally into an HAProxy configuration:
![HAProxy Data Flow](https://raw.githubusercontent.com/wodeamd/MarkdownPhotos/master/HAProxy%20Router%20Data%20Flow.png)

All pieces involved in external access:
![Router Pods](https://raw.githubusercontent.com/wodeamd/MarkdownPhotos/master/Router-Pod.png)


### Basic setup

	$ oadm policy add-scc-to-user hostnetwork system:serviceaccount:default:router
	$ oadm policy add-cluster-role-to-user cluster-reader system:serviceaccount:default:router
	$ oadm router router --replicas=1 --selector='region=infra' --service-account=router

### Multiple Routers
Multiple routers can be grouped to distribute routing load in the cluster and separate tenants to different routers or shards. 

Each router or shard in the group handles routes based on the selectors in the router.

An administrator can create shards over the whole cluster using ROUTE_LABELS. 
A user can create shards over a namespace (project) by using NAMESPACE_LABELS.


	$ oadm router router-east --selector='region=east' --replicas=0             
	$ oc env   dc/router-east ROUTE_LABELS="geo=east,dept in (dev, ops)" NAMESPACE_LABELS="router=r1"  
	# $ oc env   dc/router-west ROUTE_LABELS="geo=west,dept in (test)" NAMESPACE_LABELS="router=r1"  
	$ oc scale dc/router-east --replicas=1  

	$ oc label namespace default "router=r1"  // All pods in namespace will be selected by router-east
	
	
## DNS

### Drawback of Service IP
* Dependency on order of starting services
* Dependant has to restart to pick up new ip when service it depend on restart

### Build in Origin DNS
OpenShift Origin supports split DNS by running SkyDNS on the master that answers DNS queries for services. The master listens to port 53 by default.

When the node starts, the following message indicates the Kubelet is correctly resolved to the master:

    0308 19:51:03.118430    4484 node.go:197] Started Kubelet for node
    openshiftdev.local, server at 0.0.0.0:10250
    I0308 19:51:03.118459    4484 node.go:199]   Kubelet is setting 10.0.2.15 as a
    DNS nameserver for domain "local"

* Container’s nameserver has the master name added to the front.
* Default search domain for the container will be .<pod_namespace>.cluster.local. 
* Container direct any nameserver queries to the master before any other nameservers on the node

|Object Type|Example|
|---|:---|
|Default|<pod_namespace>.cluster.local|
|Services|<service>.<pod_namespace>.svc.cluster.local|
|Endpoints|<name>.<namespace>.endpoints.cluster.local|	

## Routing from Edge Load Balancers

An edge load balancer can be used to accept traffic from outside networks and proxy the traffic to pods inside the OpenShift Origin cluster. In cases where the load balancer is not part of the cluster network, routing becomes a hurdle as the internal cluster network is not accessible to the edge load balancer.

### Solution:
* Run an OpenShift Origin node instance on the load balancer itself that uses OpenShift Origin SDN as the networking plug-in.
* Run the load balancer as a pod with the host port exposed

## Integrating External Services

Many OpenShift Origin applications use external resources, such as external databases, or an external SaaS endpoint. These external resources can be modeled as native OpenShift Origin services, so that applications can work with them as they would any other internal service.

### Integrating with an external MySQL database
1. Create an OpenShift Origin service to represent your external database.

	Leave the Selector field unset. This represents the external service, making the EndpointsController ignore the service and allows you to specify endpoints manually:
	

    kind: "Service"
    apiVersion: "v1"
    metadata:
    name: "external-mysql-service"
    spec:
	  ports:
	    -
		  name: "mysql"
		  protocol: "TCP"
		  port: 3306
		  targetPort: 3306
		  nodePort: 0
    selector: {}

2. Create the required endpoints for the service. This gives the service proxy and router the location to send traffic directed to the service:	


	  
	  
	kind: "Endpoints"
    apiVersion: "v1"
    metadata:
	  name: "external-mysql-service" 
    subsets: 
  	  -
	    addresses:
		  -
		    ip: "10.0.0.0" 
	    ports:
		  -
		    port: 3306 
		    name: "mysql"