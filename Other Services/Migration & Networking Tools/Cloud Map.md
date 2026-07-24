# AWS Cloud Map

AWS Cloud Map is a service discovery registry: it maps friendly names to the actual backing resources behind them, whether those are ECS tasks, EC2 IP addresses, Lambda ARNs, or arbitrary endpoints, and lets consumers resolve them at runtime through DNS or the `DiscoverInstances` API. As services scale, restart, and move, they register and deregister, so callers address a stable name instead of an IP that changes constantly. Its security relevance is quieter than an encryption or policy service, but real. Cloud Map is the mechanism that lets you stop hardcoding IP addresses and static hostnames, which removes stale records pointing at reassigned addresses and removes internal topology from application configuration. It provides namespace-level segmentation so environments and trust zones have separate discovery domains, and every registration and discovery call is an IAM-authorized, CloudTrail-logged action, which means service registration is a permissioned operation rather than an open free-for-all. The important caveat is that Cloud Map answers where a service is, not whether the caller should reach it or whether the responder is authentic. The thing to hold onto is that Cloud Map is name resolution with IAM on the registration and discovery calls, so it reduces stale-record and topology-exposure risk but is not itself an authentication or authorization layer between services.

## How it works

- **Namespaces.** The top-level grouping and the segmentation boundary. A namespace can be API-only, discoverable through `DiscoverInstances`, or DNS-backed through a Route 53 private or public hosted zone. Private DNS namespaces are scoped to a VPC, which is what keeps internal names resolvable only inside the network they belong to.

- **Services.** A named, discoverable component within a namespace, such as `auth-api` inside `internal.example`. A service defines its DNS record types and TTLs, and its optional health check configuration.

- **Service instances.** The registered backing resources. Each instance carries attributes: for DNS namespaces an IP and port, for API discovery any custom attributes such as a version tag, region, or ARN. Consumers filter on these attributes at discovery time, which is how weighted or versioned routing is expressed.

- **Discovery methods.** DNS resolution through the Route 53 hosted zone for clients that just want a hostname, or the `DiscoverInstances` API for clients that want to filter on attributes and health status without DNS caching. The API path returns richer data and avoids TTL staleness.

- **Health checks.** Route 53 health checks for public and IP-based instances, or custom health status set through `UpdateInstanceCustomHealthStatus` for resources Route 53 cannot probe. Unhealthy instances are withheld from discovery results, which is what prevents traffic routing to a dead task.

- **ECS integration.** ECS service discovery registers and deregisters tasks in Cloud Map automatically as they start and stop, keeping the registry current without application code. ECS Service Connect builds on the same registry for service-to-service communication.

- **App Mesh integration.** App Mesh virtual nodes use Cloud Map as a service discovery source, so the mesh learns endpoint membership from the registry and layers its own routing, retries, and mTLS on top. Cloud Map supplies the where; App Mesh supplies the how and the encryption.

- **IAM surface.** `servicediscovery:RegisterInstance` and `DeregisterInstance` govern who can add or remove backing resources, and `servicediscovery:DiscoverInstances` governs who can resolve. Scoping registration is what stops an unauthorized workload from registering itself as a legitimate service, and scoping discovery limits which principals can enumerate the registry.

- **Encryption and network.** Discovery API traffic is TLS. DNS resolution for a private namespace stays within the VPC. There is no data at rest to speak of beyond the registry metadata, which AWS manages.

- **Logging.** CloudTrail records namespace and service creation, and instance registration and deregistration, which is the audit trail for changes to the registry. High-volume `DiscoverInstances` calls are data plane operations and are not individually logged by default, so who resolved what is not routinely captured.

## Cloud Map versus adjacent discovery and routing mechanisms

| Mechanism | What it resolves | Authorization on the operation | Health awareness | Attribute-based filtering | Provides authentication between services |
|---|---|---|---|---|---|
| AWS Cloud Map | Names to instances, IP, ARN, or custom | IAM on register and discover | Yes, Route 53 or custom status | Yes, via `DiscoverInstances` | No |
| Route 53 private hosted zone | Names to DNS records | IAM on record changes | Health checks on records | No | No |
| ECS Service Connect | Service names within a cluster | IAM plus ECS configuration | Yes | Limited | No, but adds a proxy |
| App Mesh | Virtual services to virtual nodes | IAM on mesh config | Via the discovery source | Via routes | Yes, mTLS in the sidecar |
| VPC Lattice | Services in a service network | IAM auth policies | Target health | Listener rules | Yes, IAM-based |
| Kubernetes DNS and CoreDNS | Service names to pods | Kubernetes RBAC | Readiness probes | Selectors | Only with a mesh or network policy |

## What gets tested

- **Cloud Map resolves location, it does not authenticate or authorize service calls.** If a requirement is that only an approved service may connect, the answer is App Mesh mTLS, VPC Lattice IAM auth, or security groups, not Cloud Map. Cloud Map is the discovery layer beneath those.

- **Private DNS namespaces scope resolution to a VPC**, which is the answer for keeping internal service names unresolvable outside the network, and for environment segmentation with separate namespaces per environment.

- **`DiscoverInstances` with attribute filtering** is the answer for versioned or weighted routing and for health-aware discovery without DNS TTL staleness, contrasted with plain DNS resolution which caches and cannot filter on custom attributes.

- **Restricting `servicediscovery:RegisterInstance`** is the control preventing a rogue or compromised workload from registering itself as a legitimate service and attracting traffic, which is the closest Cloud Map comes to a spoofing defense.

- **Custom health status** is the answer for resources Route 53 cannot health-check directly, such as a Fargate task behind no load balancer, keeping unhealthy endpoints out of discovery results.

- **ECS service discovery** is the answer for keeping the registry current automatically as tasks cycle, rather than registering and deregistering in application code.

- **App Mesh consumes Cloud Map for discovery**, so a question about how a mesh learns endpoint membership points at Cloud Map as the source, with the mesh adding routing and encryption on top.

- **Registration and deregistration are in CloudTrail**, which is the audit answer for changes to the registry. Discovery calls at volume are not individually logged, so resolution activity attribution is limited.

- **VPC Lattice versus Cloud Map plus App Mesh.** For new architecture wanting IAM-authorized service-to-service connectivity without running a mesh or a separate registry, Lattice combines discovery, routing, and authorization, which is the current AWS direction.

## Limitations

- It answers where, not whether. Cloud Map provides no authentication of the resolving client and no verification that the registered instance is authentic, so it is a discovery convenience, not a trust boundary.

- Registration integrity depends entirely on IAM scoping. If registration permissions are broad, a workload can register itself under any service name in the namespace and draw traffic, and Cloud Map has no built-in defense beyond the IAM policy.

- DNS-based discovery inherits DNS caching and TTL behavior, so a deregistered instance can still receive traffic from clients holding a cached record until the TTL expires. The `DiscoverInstances` API avoids this but requires application changes.

- Discovery calls are not routinely audited. There is no default record of which principal resolved which service, so enumeration of the registry is not visible the way registration changes are.

- Health check coverage is uneven. Route 53 can probe public and IP-reachable targets, but many workloads require custom health status set by external logic, which is only as reliable as the code setting it.

- Namespaces provide name-level segmentation but not network enforcement. Two services in different namespaces are separated by name only, and without security groups or a mesh underneath, nothing stops a workload that learns an address from connecting across the boundary.

- At scale, discovery call volume drives cost and can become significant for busy ECS or App Mesh deployments, and the per-call charge is easy to overlook in capacity planning.

- It is infrastructure glue with no standalone security value. Its benefits are realized only in combination with the network, mesh, or IAM controls that actually enforce who may talk to whom.