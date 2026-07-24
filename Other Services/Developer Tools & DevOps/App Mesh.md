# AWS App Mesh

AWS App Mesh is a managed service mesh control plane that configures Envoy sidecar proxies deployed alongside services on ECS, EKS, Fargate, and EC2. Every request in and out of a service passes through its proxy, which means routing, retries, timeouts, circuit breaking, telemetry, and transport encryption are configured centrally and enforced outside the application, with no code changes and no per-language library to maintain. For security work the value is transport encryption and identity at layer seven: mutual TLS between services gives each workload a verifiable certificate identity independent of its IP address, which is the foundation of any zero-trust story in an environment where pods and tasks move constantly and security groups cannot meaningfully distinguish one service from another on the same subnet. The important caveat is that App Mesh's authorization model is thinner than a full service mesh like Istio: it constrains which backends a service can reach and can validate peer certificate subject names, but it has no rich per-request policy engine. The most consequential fact is that AWS has announced end of support for App Mesh, so it now appears in existing estates and exam questions rather than new designs. The thing to hold onto is that App Mesh moves encryption and identity into the proxy layer, giving you mTLS without touching application code, and gives you far less authorization expressiveness than the mesh label implies.

## How it works

- **Mesh.** The logical boundary containing all the resources. Services inside a mesh can be discovered and routed to; services outside are external destinations.

- **Virtual node.** Represents an actual service deployment, defining its listeners, its service discovery method through Cloud Map or DNS, its health checks, its TLS configuration, and its declared backends. The backend list is the egress control: Envoy is only configured with clusters for declared backends, so a service cannot route to a mesh destination it did not declare.

- **Virtual service, virtual router, and route.** A virtual service is the name callers address. It resolves either directly to a virtual node or to a virtual router, which holds routes matching on path prefix, headers, method, gRPC service and method, or metadata, and distributing weighted traffic across virtual nodes. Weighted routing is what implements canary and blue/green shifting.

- **Virtual gateway.** An Envoy deployment at the mesh edge accepting traffic from outside and routing it in through gateway routes, so external ingress terminates TLS and enters the mesh under mesh policy rather than bypassing it.

- **Transport encryption.** One-way TLS terminates at the server's Envoy, with the certificate sourced from ACM, from AWS Private CA, or from a file on the local filesystem, and with the client optionally validating the server. Mutual TLS adds a client certificate so both ends authenticate, with client certificates provided through the file-based or SDS mechanism rather than ACM.

- **Certificate validation and SAN matching.** A client can be configured to require that the server's certificate subject alternative name matches an expected value, and a server can require the same of clients. That SAN matching is the closest App Mesh comes to service-to-service authorization: a service accepts connections only from peers presenting a certificate with an approved name.

- **Envoy Secret Discovery Service.** Certificates can be delivered dynamically to Envoy through a local SDS endpoint, typically served by SPIRE, which gives short-lived automatically rotated workload identities rather than long-lived files. This is the pattern for mTLS at scale.

- **Resilience controls.** Retries with configurable conditions and counts, per-route and per-request timeouts, connection pool limits, and outlier detection for circuit breaking. These matter for security because they bound the impact of a failing or hostile dependency and prevent a retry storm from becoming self-inflicted denial of service.

- **Observability.** Envoy emits access logs to stdout for collection by Fluent Bit or CloudWatch Logs, metrics to CloudWatch or Prometheus, and traces to X-Ray, Jaeger, or Datadog through the ADOT collector. Access logs are the per-request audit trail, and they exist only because the proxy is in the path.

- **IAM surface.** `appmesh:*` actions govern mesh resource configuration, and `appmesh:StreamAggregatedResources` is what permits an Envoy proxy to connect to the control plane and receive configuration for a specific virtual node. Scoping that action per virtual node ARN is what prevents an arbitrary workload from claiming another service's mesh identity and configuration.

- **Cross-account meshes.** A mesh can be shared with other accounts through AWS Resource Access Manager, letting services in different accounts participate in one mesh, which is how a shared platform team runs the mesh while workload accounts own their virtual nodes.

## App Mesh versus adjacent connectivity and policy layers

| Option | Where enforcement happens | Workload identity | Encryption in transit | Authorization expressiveness | Operational overhead | Current status |
|---|---|---|---|---|---|---|
| AWS App Mesh | Envoy sidecar per workload | Certificate SAN, via ACM PCA or SPIRE | mTLS, configurable | Backend declaration plus SAN matching | Sidecar per task or pod | End of support announced |
| Amazon VPC Lattice | Managed service network, no sidecar | IAM principal of the caller | TLS to the service network | IAM auth policies on services and networks | None, no proxy to run | Current AWS direction |
| Istio on EKS | Envoy sidecar or ambient | SPIFFE identity | mTLS, automatic | Rich AuthorizationPolicy, per method and claim | Substantial, you run the control plane | Actively developed |
| Security groups | ENI, layer three and four | None, IP based | None | Port and source security group only | Low | Current |
| ALB or NLB with target groups | Load balancer | None inherently | TLS termination | Listener rules, not identity | Low | Current |
| API Gateway | Managed gateway | IAM, Cognito, Lambda authorizer | TLS | Rich, at the API boundary | Low | Current |

## What gets tested

- **Encryption between services without changing application code** is the App Mesh answer, since TLS terminates in the sidecar rather than in the application.

- **mTLS gives identity independent of IP address**, which is what security groups cannot do when many services share a subnet or when pod IPs change constantly.

- **`appmesh:StreamAggregatedResources` scoped per virtual node** is the control preventing a workload from assuming another service's mesh configuration and identity. A broad grant means any task in the account can present itself as any virtual node.

- **Backend declaration is the egress control.** A virtual node can only route to virtual services it declares as backends, which is the mechanism behind "service A cannot call service C."

- **SAN validation is how a server restricts callers.** If a question asks how to ensure only an approved service can connect, the answer is requiring a client certificate and validating its subject alternative name, not a security group.

- **SPIRE with SDS is the answer for automatically rotated short-lived workload certificates.** ACM Private CA covers server certificates; client certificates for mTLS come through the file-based or SDS path.

- **Virtual gateways** are the answer for bringing external ingress into the mesh under mesh policy, rather than letting a load balancer route directly to a pod and bypass the proxy.

- **Envoy access logs are the per-request audit trail**, and they are the only source of service-to-service request records, since neither CloudTrail nor VPC Flow Logs sees application-layer requests.

- **App Mesh has reached end of support.** For new architecture, the AWS-native answer is VPC Lattice for IAM-authorized service-to-service connectivity without sidecars, or a self-managed Istio deployment when full mesh policy expressiveness is required.

- **VPC Lattice versus App Mesh.** Lattice authorizes with IAM auth policies and requires no proxy to operate, so any question emphasizing IAM-based service authorization or eliminating sidecar overhead points at Lattice.

## Limitations

- End of support has been announced, so it is not a service to build new architecture on, and existing deployments need a migration plan to VPC Lattice, Istio, or an alternative.

- The authorization model is thin. There is no per-request policy engine, no evaluation of JWT claims, no method-level or header-level allow and deny rules. Backend declaration and SAN matching are the whole story, which is considerably less than the zero-trust framing suggests.

- A sidecar per task or pod costs CPU, memory, and startup latency on every workload, and at scale that overhead is a real capacity and cost consideration.

- Envoy is additional software in the request path with its own CVEs, its own configuration failure modes, and its own version lifecycle. A proxy misconfiguration takes services down in a way that is harder to diagnose than an application error.

- The mesh only sees traffic that goes through the proxy. Anything reaching a workload by direct IP, bypassing service discovery, is outside the mesh entirely, so mesh policy is not a substitute for security groups and network policy underneath it.

- Client certificates for mTLS cannot come from ACM. They require file-based provisioning or SDS, which means running SPIRE or an equivalent, which is a substantial additional system to operate and secure.

- Configuration is declarative and eventually consistent across proxies, so a policy change does not take effect atomically across the fleet, and a partial rollout can produce inconsistent enforcement.

- Access logs are emitted by Envoy to stdout and require a log pipeline you build. There is no managed audit trail, and losing that pipeline loses the only record of service-to-service traffic.

- Cross-account and cross-cluster meshes add service discovery, certificate trust, and network reachability requirements that must all be satisfied independently, and a failure in any of them presents as a routing problem rather than a security one.