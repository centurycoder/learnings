# Envoy interview question bank: cloud traffic engineer

Focus on **how traffic flows, how policy changes that flow, and how to diagnose failures**—not just configuration syntax.

These **80 questions** cover Envoy’s core concepts and production concerns. They intentionally omit answers so you can use them as a research checklist.

**Study order**
- **Sections 1–6:** understand the data path.
- **Sections 7–12:** understand production behavior and operations.
- **Section 13:** prove your understanding through hands-on scenarios.

For each question, aim to explain: **the mechanism → relevant configuration → failure modes → observable evidence**.

---

## 1. Architecture and mental model

1. What problems does Envoy solve? Where would you deploy it as an edge proxy, internal load balancer, ingress/egress gateway, or sidecar?
2. What is the difference between a **data plane** and a **control plane**? Which responsibilities belong to Envoy?
3. What do **downstream**, **upstream**, **listener**, **filter chain**, **route**, **cluster**, and **endpoint** mean?
4. Trace an HTTPS request through Envoy—from accepting the downstream connection to returning the upstream response.
5. How does **L4 TCP proxying** differ from **L7 HTTP proxying** in routing, load balancing, visibility, and connection handling?
6. How do Envoy’s main thread and worker threads interact? Why can a small number of busy HTTP/2 connections create uneven worker utilization?

## 2. Listeners, filter chains, and request processing

7. What does a listener configure, and how does a **transport socket** differ from a network filter?
8. What is the difference between **listener filters**, **network filters**, and **HTTP filters**? When does each execute?
9. What does the **TLS inspector** discover? Why can Envoy use it without terminating TLS?
10. How does Envoy select a filter chain using properties such as destination address, SNI, and ALPN? What happens if none matches?
11. How does HTTP filter ordering affect authentication, authorization, header modification, rate limiting, and routing? How does response processing order differ?
12. What do the **HTTP Connection Manager** and **router filter** each do? How can a filter stop processing, buffer data, or generate a local response?

## 3. HTTP routing and traffic management

13. How do virtual hosts, domain matching, and route matching interact? How does route ordering affect the selected route?
14. How would you route requests using exact paths, prefixes, regular expressions, headers, query parameters, or HTTP methods?
15. What is the difference between a **redirect**, a **path/host rewrite**, and a **direct response**?
16. How would you implement a 95%/5% canary rollout with weighted clusters? Why might a short measurement not show exactly that distribution?
17. How does **request mirroring** differ from traffic splitting? What risks arise when mirroring requests with side effects?
18. How do global, virtual-host, route-level, and per-filter configurations interact? Where must you check precedence rather than assume it?

## 4. Protocols, connections, and flow control

19. How do HTTP/1.1, HTTP/2, and HTTP/3 differ in multiplexing and head-of-line blocking? What does this mean for a proxy?
20. Can downstream and upstream connections use different HTTP versions? How does Envoy select or negotiate the upstream protocol?
21. How does upstream connection pooling work? What determines whether a request reuses a connection, opens one, or waits?
22. How do HTTP/2 concurrent-stream limits affect connection counts, pending requests, throughput, and circuit breakers?
23. How does Envoy apply backpressure when a client or upstream reads slowly? How do buffers and flow-control windows affect memory?
24. What must you configure differently for gRPC streaming, WebSockets, and server-sent events compared with short HTTP requests?

## 5. Clusters and service discovery

25. What belongs in a cluster configuration, and what belongs in an endpoint assignment?
26. How do `STATIC`, `STRICT_DNS`, `LOGICAL_DNS`, `EDS`, and `ORIGINAL_DST` discovery differ? When would you choose each?
27. What happens when DNS answers change or DNS resolution fails? How can discovery state and existing connections differ?
28. What information can EDS provide beyond an endpoint’s IP address and port? How does that information influence traffic?
29. What happens when endpoints are added, removed, or marked unhealthy? How are new requests and existing connections affected?
30. How do active health checks differ from passive observations of real traffic? Can a successful health check coexist with failing user requests?

## 6. Load balancing and failover

31. When would you choose round robin, least request, random, ring hash, or Maglev? What workload assumptions influence the choice?
32. At what point does Envoy choose an endpoint for HTTP requests versus raw TCP connections? Why does this matter for long-lived connections?
33. How does consistent hashing work? What happens to affinity when endpoints change, become unhealthy, or receive different weights?
34. How do **priority levels**, **locality weights**, and **zone-aware routing** differ?
35. What is **panic mode**? Under what conditions might Envoy send traffic to endpoints considered unhealthy, and why?
36. How would you design availability-zone failover without overwhelming the surviving zone? How can slow start help newly added endpoints?

---

## 7. Timeouts, retries, and failure isolation

37. Distinguish connection timeout, request-receive timeout, route timeout, stream idle timeout, and maximum stream duration. When does each timer start?
38. How do overall route timeout, per-try timeout, retry backoff, and gRPC deadlines interact?
39. Which failures are appropriate to retry? Why can retrying a request duplicate a business operation even when the client received an error?
40. How do retries at the client, gateway, sidecar, and application layers multiply load? How would you bound the amplification?
41. What resources do Envoy’s circuit breakers limit? How do connection, pending-request, active-request, and retry limits differ?
42. How does **outlier detection** eject and restore endpoints? How does it differ from active health checking and circuit breaking?

## 8. Rate limiting and overload protection

43. What is the difference between **local** and **global** rate limiting? How does adding Envoy replicas affect each?
44. How do token-bucket parameters control sustained traffic and bursts? How would you choose them for a particular service?
45. How do rate-limit descriptors represent identities such as tenant, API key, route, or client address? Which inputs can you trust?
46. What happens when an external rate-limit service fails or becomes slow? What are the consequences of fail-open versus fail-closed?
47. How does the **overload manager** differ from upstream circuit breaking? Which resources and actions would you configure?
48. How would you protect Envoy from slow clients, excessive connections, large headers or bodies, and too many concurrent streams?

## 9. TLS, identity, and trust boundaries

49. What is the difference between TLS termination, TLS passthrough, and terminating downstream TLS while originating upstream TLS?
50. What do **SNI** and **ALPN** each communicate? How do they influence filter-chain selection and protocol negotiation?
51. What must Envoy validate when connecting to an upstream over TLS? Why is encryption alone insufficient?
52. How does mutual TLS authenticate peers? How do certificate identity, trusted CAs, and authorization relate?
53. How do JWT authentication, RBAC, and external authorization differ? What happens when the authorization service is unavailable?
54. How should Envoy handle `X-Forwarded-For`, forwarded scheme headers, PROXY protocol, and client-controlled `x-envoy-*` headers across trusted and untrusted hops?

## 10. Dynamic configuration and xDS

55. What remains in bootstrap configuration when most configuration is dynamic? How does Envoy discover and authenticate its control plane?
56. What do **LDS, RDS, CDS, EDS, and SDS** deliver? Draw the dependency graph for one HTTPS route.
57. What is the difference between **State-of-the-World** and **Delta xDS**? How do subscriptions, updates, and resource removals differ?
58. What do xDS **versions**, **nonces**, **ACKs**, and **NACKs** mean? Does an ACK prove a listener is active and serving traffic?
59. What does **ADS** provide? Does sharing one stream make a multi-resource configuration update atomic?
60. What happens during control-plane disconnection, reconnection, or an invalid update? How would you publish dependent resources without transient traffic failures?

## 11. Observability and incident diagnosis

61. Which metrics would you monitor for request rate, errors, latency, connections, pending work, retries, endpoint health, and Envoy saturation?
62. How would you distinguish an upstream-generated error from an Envoy-generated local response? Why is an HTTP status code alone insufficient?
63. What do access-log response flags such as `NR`, `UH`, `UF`, `UO`, and `UT` mean? Which evidence would you investigate next for each?
64. Which structured access-log fields help identify the selected route, cluster, endpoint, retry attempts, failure reason, and request timing?
65. How would you use `/config_dump`, `/clusters`, `/stats`, and `/server_info` to compare intended configuration with actual state? Why must the admin interface be protected?
66. How do request IDs and distributed tracing help across multiple proxies? What can sampling, missing propagation, and high-cardinality metrics hide?

## 12. Deployment, performance, and cloud integration

67. What is the difference between dynamic configuration updates, listener draining, hot restart, and replacing an Envoy process or container?
68. How would you shut down Envoy gracefully in Kubernetes? How must endpoint removal, readiness, draining, and termination grace periods cooperate?
69. Why might scaling out an Envoy fleet fail to redistribute existing HTTP/2 or TCP traffic? What options address connection stickiness?
70. How would you investigate high CPU or memory use? How can TLS, buffering, filters, connection churn, logging, and worker imbalance contribute?
71. How do Envoy configuration, Kubernetes Services/EndpointSlices, Gateway API resources, and a service-mesh control plane relate? Which component translates desired policy into xDS?
72. How would you roll out an Envoy binary or configuration change safely—covering validation, compatibility, canaries, observability, and rollback?

---

## 13. Hands-on interview scenarios

For these, **build the setup or inject the failure**, then answer using configuration, logs, metrics, and observed traffic.

73. **Minimal proxy:** Can you run Envoy in front of two HTTP backends, add host/path routing and structured access logs, and prove which backend handled each request?

74. **Canary rollout:** Can you shift traffic from 100% old to 90%/10% and then roll back? Which measurements would justify progressing or stopping?

75. **Failure diagnosis:** Your proxy starts returning 503s. How would you distinguish no healthy endpoints, connection refusal, TLS failure, and circuit-breaker overflow?

76. **Timeouts and retries:** One backend becomes slow. Can you demonstrate the effects of route timeout, per-try timeout, and retries—and measure additional upstream attempts?

77. **Streaming failure:** Unary gRPC calls succeed, but streaming calls disconnect at a repeatable interval. How would you identify the responsible timeout or intermediary?

78. **Uneven utilization:** After scaling from two Envoy replicas to six, the original two remain overloaded. How would you prove or disprove long-lived connection stickiness?

79. **Dynamic updates:** Can you update endpoints and routes without restarting Envoy, submit an invalid update, and observe both the rejection and continued serving behavior?

80. **TLS rotation:** Can you rotate certificates using SDS while traffic continues? How would you distinguish what existing connections use from what newly established connections use?

---

## Official reading map

- **Architecture and request flow:** [Life of a Request](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)
- **Filters, routing, discovery, security, and operations:** [Architecture overview](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/arch_overview)
- **Load balancing and failover:** [Load balancing](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancing)
- **Timeout semantics:** [How do I configure timeouts?](https://www.envoyproxy.io/docs/envoy/latest/faq/configuration/timeouts.html)
- **Control-plane behavior:** [xDS overview](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/dynamic_configuration) and [xDS protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)
- **Proxy protection:** [Overload manager](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/overload_manager)

Use documentation matching your installed Envoy version; the `latest` documentation currently describes a development build.

**Readiness criterion:** You can explain the request path without notes, predict behavior under failure, identify the controlling configuration, and demonstrate the result. Definitions alone are not enough.
