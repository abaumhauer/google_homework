# POC for hybrid WordPress deployment in GCP

## The usage patterns:
* 90% of customers live in San Francisco and Tokyo
* Current average hourly traffic is 1k active users
* Estimate adding 10k active users a month
* Social marketing on the 1st & 15th of the month bring in 1,000,000 users to the website over about 24 hours
* Wordpress web content is built once and then updated no more than twice a year
* There are no customer logins for Wordpress, it is just a static informational website
* Database queries are rare and DB usage is very low
* Cost is the least important factor
* Uptime and low complexity of infrastructure management is most important factor
* Cloud native & best practices GCP infrastructure is preferred

## Deployment Overview:
Acme Inc has a datacenter at it's facilities in Palo Alto, CA with available hardware for deploying the WordPress POC. The following equipment is available for use in the deployment:
* (8) Dell FX2 server blades with 36 Cores, 256GB RAM, 1.2TB SSD
* (8) VMWare VSphere ESXi licenses
* (7) Arista 7060SX2 25GB/100GB switches (three spine, and four leaf switches)
* (1) 10GB Fiber Direct connect to Google Colocation facility in San Jose Equinix (SV1)
* (1) 10GB Connection with MegaPort to QTS in Santa Clara

### Compute (control plane)
The physical datacenter will run ESXi hypervisors hosting Google Anthos GKE On-Prem which ties Google's control plane seamlessly between the on-prem data center, and GKE in the cloud. Kubernetes (k8s) will be installed on the VMs hosted on ESXi (KVM is not yet supported). 

### Storage (Ceph)
After standing up a local on-prem k8s cluster, Rook.io will provide storage for the cluster. Rook.io allows for simplified management of Ceph. The raw SSDs will provide Ceph Object Storage Devices (OSD) that is used for the persistent database backing store. Rook.io is installed using the Helm operator chart.

### Networking
The on-site k8s cluster will have three networks: 
* Internal management network (172.16.0.0/22) as VLAN 1600
* Internal VM network (172.16.4.0/22) as VLAN 1640
* External VM network (172.16.8.0/22) as VLAN 1680
* The Google Direct Connect will attach to one of the leaf switches (Leaf-A) and will be configured as a trunking port supporting 802.1Q and allowing only VLAN 10 and 20, which are the VLANs selected for the Google VPCs in San Jose (sjc-zone-1-6) and Tokyo (nrt-zone1-452).
* Google supplies APAIP (169.254.X.X/31) peering addresses for BGP. VLAN 10 and 20 have SVI interfaces with the supplied IPs and EBGP peer with Google Cloud Router to provide route exchange between the on-premise networks and the VPCs at Google. Route maps are used to filter allowable imported/exported prefixes as a security precaution (best practices). MEDs are set to prefer the Google DC over the MegaPort connection, but both links are active and available for traffic redundancy.
* The MegaPort connection will be configured similarly on Leaf-C. Additionally MegaPort API will be used to create a virtual cross-connect to Google for our two VPCs. BGP will be established with the Google Cloud Router(s).
* Three Layer 3 SVI interfaces will have default gateways for the networks mentioned above with IP addresses of 172.16.X.1/32
* A LoadBalancer (MetalLB) will be hosted on VMs and use the external VM network to provide static IPs for incoming requests from Google to be load-balanced to the nginx server containers supporting WordPress.
* Google Cloud DNS hosts the domain (zones) for DNS lookup.

### Software
* Helm chart will deploy MariaDB using Galara for multi-master database replication. Three instances of the database will provide redundancy, and protect against split-brain if a network switch/VM fails.
* Ceph will provide persistent storage to MariaDB database on each pod. Ceph will provide redundancy for the local storage.
* Helm chart will install WordPress/nginx instances. Configuation of site is accomplished by setting up values during the helm install.
* Static content is deployed to Google Cloud Storage buckets.
* Google Content Distribution Network is configured to provide local caching of static content in sjc and nrt regions and handle auto-scaling of load. CDN also provides DDoS protection so that GDC/MegaPort links are not saturated.
* Load Balancer in GKE will direct dynamic and un-cached requests to the WordPress pods at the Palo Alto datacenter via the best path connection (either Google DC or MegaPort).
* TLS cert management is through LetsEncrypt
* GKE is used to host IAP connector.

### Administration
* Admin domain is created to access management network on-prem. (admin.acme.com).
* Load Balancer has Google Identity Aware Proxy enabled. IAP connector provides the front end in the project. Rules are used to forward requests from authenticated and authorized admins to the WP admin page and resources. SSH access to on-prem management services are also enable with IAP TCP proxy. Authorized admins are configured with IAM user/group and roles that will allow their laptop devices access to protected resources. Authorized devices will be enrolled and posture data collected with the browser extension.
* WordPress will be hardened with least access required.
* Stackdriver will provide consolidation of all logs -- on-prem and in GKE.

### Security considerations
* Network prefix advertisements are limited using route maps to the necessary hosts (external load balancer) and admin network.
* ACLs are used to limit exposed ports so that inadvertent services are not exposed.
* Zero trust networking principles are used. Don't assume that the on-prem network is more secure than the open network. Corporate network is not exposed from the on-prem GKE cluster. Management of resources is done via IAP.
* MariaDB database is hardened. All intra-node DB traffic is TLS.
* nginx servers are hardened.
* Containers are built from Dockerfiles/buildah scripts and artifacts are placed in Google Container Registry
* Containers running are periodically scanned for vulnerabilities, and rebuilt on a 90 day basis and re-deployed as helm updates.
* DDoS is accomplised via Google Cloud CDN.
* Calico can provide distributed firewalling in k8s clusters.
* Log Warehouse for storing and analyzing Stackdriver logs.

