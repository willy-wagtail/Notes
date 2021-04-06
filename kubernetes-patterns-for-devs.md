# Multi containers for pods

why pods
- abstraction above containers
- kube needs additional information that containers dont have such as restart policy
- simplifies using diff underlying container runbtimes
- colocate tightly coupled containers without packaging them as single image

sidecar pattern
- use a helper container to assist a primary container
- commonly used for logging, file syncing, watchers
- benefits include leaner main conatiner, failure isolation, independent update cycles

e.g. primary container is web server, sidecar container is a content puller. Content puller syncs with CMS. Web server just serves content. The content is synced using a volume.

ambassador pattern
- uses proxy container for communicating to and from primary container - because containers in pod share network space, so can communicate each other via local host
- common used for communicating with databases - app always connects to localhost with the ambassador proxying to database
- streamlined development experience, potential to reuse ambassador across languages

e.g. primary container is web app, ambassador is database proxy. Web app handles requests, and makes any database requests to the Database proxy over localhost. Database proxy then forwards the requests to the appropriate database - possibly sharding the requests.

adapter pattern
- adapters present a standardised interface across multiple pods
- commonly used for normalising output logs and monitoring data
- adapts thirrdparty software to meet your needs

# Networking

networking basics in kube
- unique ip per pod
- containers in pod share same ip, comm using ports on localhost
- pod scheduled to a node in cluster. 
- Any node can access pod using pod IP address. Cant reply on pod IPs as pods can be killed, replicated, etc.
- services maintains set of pod replicas using labels and keeps list of endpoints as pods are added and removed from set. 
- clients of services only needs to know the service rather than the pods. Clients can discover services using env vars if pod was created after the service. Or using DNS add on in cluster to resolve service name, or namespace qualified service name to the IP of service. 
- IP given to service is called Cluster IP. This is most basic type of service. Can only be reached inside cluster. Kube cluster cluster component on each node will proxy requests to one of services endpoints
- other services allow clients outside cluster to connect to the service.
- Node port service type opens a given port in every node in cluster. Cluster IP is still given to service. Any request made to the node port of any node will be routed to the Cluster IP.
- Load balancer type exposes service externally via cloud providers load balancer. Also creates a cluster IP and a node port for the service. Request to load balancer is sent to node port and routed to cluster IP.
- external name service: enabled by DNS. Name and request for service returns CNAME record with the external DNS name. Can be used for services running outside of kubernetes such as DB as a service offering.

Kubernetes network policy
- rules for controlling network access to pods, its like a firewall
- similar to security groups controlling access to VMs
- scoped to namespaces
- kubernetes network plugin must support network policy
- some plugins that support network policy are calico and romana
  - can check by checking the kube-system namespace ``kubectl get pods -n kube-system`` and look for pods prefixed by calico for example
- isolated v non isolated pods.
- non isolated pod can receive traffic from any source
- once a pod is selected using network policy (using labels), it becomes an isolated pod
- policy applies only on new connections, rather than already established connections

Service accounts