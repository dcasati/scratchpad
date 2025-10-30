---
title: Systematic approach to troubleshooting network problems in AKS clusters
description: Learn about systematic approaches and methodologies to troubleshoot network problems in Azure Kubernetes Service (AKS) clusters. For detailed troubleshooting guides, see the AKS connectivity troubleshooting documentation.
author: dcasati
ms.author: dicasati
ms.date: 10/27/2025
ms.topic: conceptual
ms.subservice: architecture-guide
ms.custom:
  - e2e-aks
  - arb-containers
---

# Systematic approach to troubleshooting network problems in AKS clusters

Network problems in Azure Kubernetes Service (AKS) clusters can manifest in various ways, from API server connectivity issues to pod-to-pod communication failures. This article provides a systematic approach to diagnosing and resolving network problems, focusing on methodology and decision-making rather than prescriptive step-by-step instructions.

## General troubleshooting methodology

When approaching network problems in AKS, adopt a systematic methodology:

1. **Define the problem scope** - Identify what's failing, where it's failing, and under what conditions
2. **Establish a baseline** - Understand what normal behavior looks like
3. **Isolate the issue** - Narrow down the problem to specific components or layers
4. **Form hypotheses** - Develop theories about root causes based on symptoms
5. **Test systematically** - Validate or invalidate hypotheses through targeted testing
6. **Document findings** - Keep track of what you've tested and learned

Always consult the [AKS troubleshooting guide](/azure/aks/troubleshooting) as a starting point, as many common issues are documented there.

### Applying the USE method to network troubleshooting

The [USE Method](http://www.brendangregg.com/usemethod.html), developed by Brendan Gregg, provides a structured approach to performance analysis by examining resources for:

- **Utilization** - How busy a resource is (percentage of time the resource is active)
- **Saturation** - The degree of extra work a resource cannot service (often measured as queue length)
- **Errors** - Count of error events

> **For every resource, check utilization, saturation, and errors.**

In the context of AKS network troubleshooting, apply the USE method to systematically analyze network-related resources:

#### Network resource analysis

| Resource | Type | Metrics to check |
|----------|------|------------------|
| Network interfaces | Utilization | Network bytes in/out, bandwidth usage, throughput percentage |
| Network interfaces | Saturation | Network queue depth, dropped packets, retransmissions |
| Network interfaces | Errors | Network errors, failed connections, CRC errors |
| DNS | Utilization | DNS query rate, CoreDNS CPU/memory usage |
| DNS | Saturation | DNS query queue depth, response time latency |
| DNS | Errors | DNS failures, timeouts, SERVFAIL responses |
| Load balancers | Utilization | Connection count, throughput |
| Load balancers | Saturation | Connection queue depth, backend pool capacity |
| Load balancers | Errors | Failed health probes, connection failures |
| Network policies | Utilization | Policy evaluation rate |
| Network policies | Saturation | iptables rule count, rule processing time |
| Network policies | Errors | Dropped packets, policy violations |

The USE method helps identify whether network performance problems stem from resource constraints, capacity limits, or configuration errors.

For more information, see the [USE Method: Linux Performance Checklist](https://www.brendangregg.com/USEmethod/use-linux.html).

## Understanding AKS network layers

Network connectivity in AKS involves multiple layers, each of which can be a potential point of failure:

- **Control plane layer** - API server accessibility and authentication
- **Data plane layer** - Node-to-node communication and health
- **Pod network layer** - CNI plugin, IP address management, and pod-to-pod connectivity
- **Service layer** - kube-proxy, load balancing, and service discovery
- **DNS layer** - CoreDNS functionality and name resolution
- **Egress layer** - Outbound connectivity, firewall rules, and routing
- **Platform layer** - Azure Virtual Network, NSGs, route tables, and Azure networking services

Understanding these layers helps you isolate problems more effectively.

## Approach to API server connectivity issues

When clients cannot reach the API server, consider these investigation paths:

### Network access control

The API server may have network restrictions that prevent access:

- **Authorized IP ranges** - Verify whether the cluster has API server authorized IP ranges enabled and if the client's IP is included
- **Private cluster configuration** - Determine if the cluster is private and whether access is being attempted from an appropriate network location
- **Network policies** - Check if network policies are blocking access to the control plane

### Connectivity path validation

Trace the network path from client to API server:

- **DNS resolution** - Ensure the API server FQDN resolves correctly
- **Network routing** - Verify routing tables and confirm traffic can reach the API server endpoint
- **Firewall rules** - Check that firewalls (Azure NSGs, Azure Firewall, or third-party solutions) allow the necessary traffic
- **TLS/certificate issues** - Validate that certificates are valid and trusted

### Authentication and authorization

Even if network connectivity exists, access may be denied:

- **Credential validity** - Ensure kubeconfig credentials are current and valid
- **RBAC configuration** - Verify that the user or service principal has appropriate permissions
- **Token expiration** - Check if authentication tokens have expired

### Rate limiting and throttling

The API server implements rate limiting to protect against overload:

- **429 Too Many Requests errors** - Indicate that the client has exceeded the allowed request rate to the API server
- **Request patterns** - Review application behavior to identify excessive API calls or polling loops
- **Client configuration** - Check if multiple clients or controllers are making redundant requests
- **Informer caching** - Ensure Kubernetes clients are using informers with proper caching rather than repeated LIST operations

High request rates can stem from misconfigured applications, excessive reconciliation loops in controllers, or too many clients polling the API server. For more information, see [Troubleshoot 429 Too Many Requests errors](/troubleshoot/azure/azure-kubernetes/create-upgrade-delete/429-too-many-requests-errors).

## Approach to pod IP allocation failures

When pods fail to obtain IP addresses, investigate these areas:

### IP address exhaustion

The most common cause is running out of available IP addresses:

- **Subnet sizing** - Evaluate whether the subnet has sufficient IP addresses for the cluster's scale
- **IP allocation efficiency** - Understand how your CNI plugin allocates IPs (per-pod vs. per-node)
- **IP address collision** - Investigate whether the AKS cluster is learning new network addresses from other sources (for example, from on-premises) and whether this is causing collisions in routing. One example would be when a cluster connects back to an on-premises datacenter and is learning new network address spaces through Border Gateway Protocol (BGP)

### CNI plugin issues

The Container Network Interface plugin is responsible for IP allocation:

- **Plugin health** - Verify the CNI plugin is functioning correctly
- **IPAM store consistency** - Check if the IPAM (IP Address Management) store reflects reality
- **Plugin configuration** - Ensure CNI plugin settings align with network architecture

## Approach to service accessibility issues

When services are not accessible from pods, systematically evaluate:

### Service discovery layer

Ensure the service is properly registered and discoverable:

- **Endpoint creation** - Verify that endpoints are automatically created for the service
- **Label selectors** - Confirm that service label selectors correctly match target pods
- **Pod readiness** - Check that backend pods are in a ready state

### Network connectivity layer

Test connectivity at different levels:

- **Pod IP reachability** - Verify direct pod IP connectivity bypassing the service
- **Container port configuration** - Ensure containers are listening on expected ports
- **Service port mapping** - Confirm service port configuration matches container ports

### Service proxy layer

The kube-proxy component handles service load balancing:

- **kube-proxy health** - Verify kube-proxy pods are running and healthy
- **iptables/IPVS rules** - Examine whether proper forwarding rules are configured
- **Service mode** - Understand which proxy mode is in use (iptables, IPVS, etc.)

### Network policies and security

Access may be blocked by security controls:

- **Network policy impact** - Determine if Kubernetes Network Policies are restricting traffic
- **CNI plugin policies** - Check if the CNI plugin (e.g., Calico, Cilium) has additional policies
- **Platform-level controls** - Consider Azure NSGs or other platform security controls

## Approach to node-to-API server connectivity

When nodes or pods cannot reach the API server, investigate from multiple angles:

### Internal service functionality

The kubernetes service provides API server access within the cluster:

- **Service existence** - Confirm the kubernetes-internal service exists and has correct endpoints
- **DNS resolution** - Verify the service DNS name resolves correctly
- **Endpoint health** - Ensure the endpoint points to a reachable API server

### Network path validation

Trace the path from node/pod to API server:

- **Routing configuration** - Check that routes exist to reach the API server
- **Platform networking** - Verify Azure Virtual Network configuration is correct
- **Firewall traversal** - Ensure outbound rules permit API server communication

### Configuration-specific issues

Different cluster configurations present different challenges:

- **Private clusters** - Validate that private DNS zones are configured correctly and that DNS forwarders work
- **Custom DNS** - If using custom DNS servers, ensure they can resolve cluster-internal names
- **Outbound restrictions** - Verify that egress restrictions don't block required FQDNs
- **User-Defined Routes (UDR) with egress lockdown** - When using UDR in outbound type with restricted egress (such as through Azure Firewall or a network virtual appliance), ensure all required FQDNs and network rules are properly configured. Missing rules can prevent nodes from reaching the API server, container registries, or other essential services. For detailed requirements, see [Control egress traffic for cluster nodes in AKS](/azure/aks/outbound-rules-control-egress)

### Authorization layer

Network connectivity is necessary but not sufficient:

- **RBAC configuration** - Ensure pods have appropriate ServiceAccount permissions
- **Workload Identity** - Check if the Workload Identity has the correct set of RBAC permissions and that the Managed Identity tied to it has the appropriate role assignments
- **Role bindings** - Verify that necessary RoleBinding and ClusterRoleBinding objects exist
- **Admission controllers** - Consider whether admission webhooks might be affecting access

## Diagnostic tools and techniques

Choose appropriate tools based on the specific problem you're investigating. The following categories represent common diagnostic approaches for network troubleshooting in AKS:

### Connectivity testing

- **Test pods** - Deploy ephemeral test pods with networking tools
- **Direct IP testing** - Use curl, telnet, or netcat to test specific endpoints
- **DNS queries** - Use dig or nslookup to validate name resolution

### Traffic analysis

- **Packet captures** - Capture traffic at various points to understand flow
- **Flow logs** - Use Azure NSG flow logs or VNet flow logs to track traffic
- **Service mesh observability** - If using a service mesh, leverage its observability features

### State inspection

- **Kubernetes objects** - Examine services, endpoints, pods, and network policies
- **Node configuration** - Review node networking configuration and routing tables
- **Azure resources** - Inspect NSGs, route tables, VNets, and associated configurations

### Logging and monitoring

- **Container insights** - Query logs from various cluster components
- **Control plane logs** - Analyze kube-apiserver, kube-controller-manager logs
- **Metrics analysis** - Review network metrics, error rates, and latency patterns

## Contributors

*This article is maintained by Microsoft. It was originally written by the following contributors.*

Principal author:
- [Diego Casati](https://www.linkedin.com/in/ayobamiayodeji/) | Azure Global Black Belt

Other contributors:

- [Michael Walters](https://www.linkedin.com/in/mrwalters1988/) | Senior Consultant
- [Ayobami Ayodeji](https://www.linkedin.com/in/ayobamiayodeji) | Senior Program Manager
- [Bahram Rushenas](https://www.linkedin.com/in/bahram-rushenas-306b9b3) | Architect

## Next steps

- [Network concepts for applications in AKS](/azure/aks/concepts-network)
- [AKS connectivity troubleshooting](/troubleshoot/azure/azure-kubernetes/welcome-azure-kubernetes#connectivity)
- [Basic troubleshooting of DNS resolution problems in AKS](/troubleshoot/azure/azure-kubernetes/connectivity/dns/basic-troubleshooting-dns-resolution-problems)
- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Kubernetes Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking)
- [Choose the best networking plugin for AKS](/training/modules/choose-network-plugin-aks)

## Related resources

- [AKS architecture design](../../reference-architectures/containers/aks-start-here.md)
- [Baseline architecture for an AKS cluster](/azure/architecture/reference-architectures/containers/aks/baseline-aks)
- [AKS day-2 operations guide](../../operator-guides/aks/day-2-operations-guide.md)
- [Monitoring AKS with Azure Monitor](/azure/aks/monitor-aks)
- [AKS networking best practices](/azure/aks/operator-best-practices-network)
