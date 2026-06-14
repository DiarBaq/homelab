# homelab
This is just a way for me to document and keep track of my K3S cluster.

It outlines the architecture, configuration, and deployment of a bare-metal Kubernetes (K3s) cluster. Designed for high-performance, latency-sensitive applications, the project demonstrates advanced network routing, resource isolation, and high availability without relying on cloud-provider infrastructure.

Hardware Architecture
The cluster operates on a fleet of Lenovo ThinkCentre nodes, providing an efficient balance of compute capability and power consumption:

Control Plane (m710q): Manages cluster orchestration, ingress routing, and dedicated network tunnel agents.

Worker Node 1 (m720q): Handles application compute and dynamic workloads.

Worker Node 2 (m910q): Provides additional compute capacity and redundancy.

Network Architecture & Routing Strategy
A primary focus of this project is optimising network throughput and ensuring the stability of long-running, persistent UDP and TCP connections.

1. CNI Optimisation (Flannel host-gw)
By default, K3s utilises Flannel with a VXLAN backend. While useful for cloud environments, VXLAN introduces unnecessary packet encapsulation overhead for bare-metal nodes operating on the same physical switch.

Implementation: The cluster configuration was migrated to the host-gw backend.

Impact: Pod traffic now routes natively over local physical network interfaces, bypassing encapsulation and significantly reducing network latency for UDP traffic.

2. Decoupled Ingress and Tunneling
To allow application pods to migrate freely between worker nodes without dropping external client connections, the network layer is fully decoupled from the application layer.

Implementation: External network tunnels run as independent deployments, strictly pinned to the Control Plane using nodeSelectors.

Impact: Compute-heavy application spikes on worker nodes cannot starve the networking agents of CPU cycles. The ingress agents maintain a stable connection state, and Kubernetes handles the internal routing as application pods shift across the cluster.

3. Eliminating Asymmetric Routing
Tunneling external UDP traffic into a Kubernetes cluster often triggers asymmetric routing. The kube-proxy NAT alters incoming packets, but return packets often bypass the service IP, causing external edge servers to drop the connection for security reasons.

Implementation: The cluster implements Kubernetes Headless Services (clusterIP: None) for latency-sensitive deployments.

Impact: External tunnel agents resolve and communicate directly with the underlying Pod IP. This establishes a clean, bidirectional, NAT-free path that maintains persistent UDP streams.

Development Roadmap
[x] Phase 1: Infrastructure Foundation

Provision base operating systems.

Bootstrap the K3s cluster and join worker nodes.

Establish persistent storage volume claims.

[x] Phase 2: Network Optimisation

Identify packet encapsulation latency.

Migrate the Flannel CNI from VXLAN to host-gw.

Configure internal agents to prefer IPv4 to resolve K8s DNS routing delays.

[x] Phase 3: High Availability Routing

Decouple tunnel and ingress agents from application pods.

Pin networking deployments to the control plane.

Implement Headless Services to resolve NAT-induced packet drops.

[ ] Phase 4: Observability & Automation (Upcoming)

Implement metrics scraping via Prometheus and Grafana for traffic monitoring.

Automate deployment updates using a GitOps workflow (ArgoCD/Flux).

Deployment Configuration Standards
Resource Isolation
To ensure critical network agents are never throttled by application workloads, requests and limits are strictly enforced in the deployment manifests to guarantee CPU time:
