---
title: Systematic approach to troubleshooting network problems in AKS clusters
description: Learn about systematic approaches and methodologies to troubleshoot network problems in Azure Kubernetes Service (AKS) clusters. For detailed troubleshooting guides, see the AKS connectivity troubleshooting documentation.
author: dcasati
ms.author: dicasati
ms.date: 04/02/2026
ms.topic: conceptual
ms.subservice: architecture-guide
ms.custom:
  - e2e-aks
  - arb-containers
---

# Systematic approach to troubleshooting network problems in AKS clusters

Network problems in Azure Kubernetes Service (AKS) clusters can manifest in various ways, from API server connectivity issues to ingress/egress challenges to pod-to-pod communication failures. This guide is intended to help you identify the most likely layer of failure and apply a structured approach to isolate root causes in AKS networking scenarios, rather than follow prescriptive troubleshooting steps.

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

### Applying the golden signals to network troubleshooting

In addition to the USE method, the **golden signals** (latency, traffic, errors, saturation) provide another effective lens for analyzing network behavior in AKS environments.

| Signal | What to evaluate | Example indicators in AKS networking |
|--------|------------------|-------------------------------------|
| **Latency** | Time taken for requests to complete | Increased DNS resolution time, high service response times, slow API server responses |
| **Traffic** | Volume of requests or data flow | Sudden spikes in network throughput, increased DNS query rate, load balancer connection count |
| **Errors** | Failed requests or operations | Connection timeouts, DNS failures, dropped packets, failed health probes |
| **Saturation** | Resource capacity limits | Network interface congestion, connection queue buildup, CoreDNS or load balancer limits |

These signals help identify whether issues are caused by:

- **Performance degradation** (latency, saturation)  
- **Capacity constraints** (traffic, saturation)  
- **Configuration or reliability problems** (errors)

When used together with the USE method, the golden signals provide both:

- A **resource-oriented view** (USE), and  
- A **user-impact view** (golden signals)

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

## Mapping common failure patterns to network layers

When troubleshooting, symptoms often provide strong signals about which layer is most likely responsible. The following table maps common failure patterns to the most relevant AKS network layer to investigate first.

| Failure pattern | Likely layer(s) | Description |
|----------------|-----------------|-------------|
| Cannot reach API server from client | Control plane, Platform | Often related to authorized IP ranges, private cluster access, DNS resolution, or NSG/firewall rules |
| Intermittent API server errors (429, timeouts) | Control plane | Typically caused by rate limiting, excessive API calls, or client behavior |
| Pod-to-pod communication fails | Pod network, Data plane | Usually indicates CNI, routing, or IP addressing issues. Could also indicate the presence of a networking policy blocking traffic (Network Security Group or in-cluster by a network policy like Calico or Cillium) |
| Pod can reach IP but not service | Service layer | Suggests service misconfiguration, kube-proxy/eBPF issues, or port mismatches |
| Service resolves but connection fails | Service, Network policy, Platform | May indicate blocked traffic (Network Policies, NSGs) or backend pod issues |
| DNS resolution fails inside cluster | DNS layer | Typically CoreDNS issues, misconfiguration, or upstream DNS problems |
| External access to service fails (Ingress/LoadBalancer) | Service, Egress, Platform | Often related to load balancer configuration, NSGs, or routing |
| Pod cannot reach external endpoints | Egress, Platform | Common causes include UDR, firewall restrictions, or missing outbound rules |
| Node cannot reach API server | Control plane, Platform | Often due to routing, DNS, firewall, or private endpoint misconfiguration |
| IP allocation failures for pods | Pod network | Usually subnet exhaustion, IPAM inconsistencies, or CNI configuration issues |
| Works via TCP/UDP but fails via ICMP (ping) | Platform | Often due to Azure load balancer and SNAT behavior affecting ICMP |

> Start with the most likely layer, but always validate assumptions across adjacent layers to avoid false conclusions.

> [!IMPORTANT]
> AKS networking issues often span multiple layers. Always validate findings across adjacent layers to avoid misattributing the root cause.

## Approach to API server connectivity issues

When clients cannot reach the API server, consider these investigation paths:

### Network access control

The API server may have network restrictions that prevent access:

- **Authorized IP ranges** - Verify whether the cluster has API server authorized IP ranges enabled and if the client's IP is included
- **Private cluster configuration** - Determine if the cluster is private and whether access is being attempted from an appropriate network location
- **Network Security Group rules** - Check if Azure Network Security Group (NSG) rules are blocking access to the control plane

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
- **HTTP status codes** - Analyze HTTP response codes for clues (e.g., 401 Unauthorized, 403 Forbidden)

### Rate limiting and throttling

The API server implements rate limiting to protect against overload:

- **429 Too Many Requests errors** - Indicate that the client has exceeded the allowed request rate to the API server
- **Request patterns** - Review application behavior to identify excessive API calls or polling loops
- **Client configuration** - Check if multiple clients or controllers are making redundant requests
- **Informer caching** - Ensure Kubernetes clients are using informers with proper caching rather than repeated LIST operations

High request rates can originate from misconfigured applications, excessive reconciliation loops in controllers, or poorly designed controllers or scripts performing frequent LIST operations instead of using watches/informers. For more information, see [Troubleshoot 429 Too Many Requests errors](/troubleshoot/azure/azure-kubernetes/create-upgrade-delete/429-too-many-requests-errors).

## Approach to pod IP allocation failures

When pods fail to obtain IP addresses, investigate these areas:

### IP address exhaustion

The most common cause is running out of available IP addresses:

- **Node surge** - During maintenance operations, you can decrease the node surge to a lower number from its default value. Decreasing node surge can to a lower count can help minimize the IP address consuption during these operations.
- **Subnet sizing** - With Azure CNI in flat networking mode (non-Overlay), evaluate whether the subnet has sufficient IP addresses for the cluster's scale. Overlay mode reduces subnet pressure but introduces encapsulation and different routing characteristics
- **IP allocation efficiency** - Understand how your CNI plugin allocates IPs (per-pod vs. per-node)
- **IP address collision** - Investigate whether the AKS cluster VNet is learning new network addresses from other sources (for example, from on-premises) and whether this is causing collisions in routing. One example would be when a cluster connects back to an on-premises datacenter and is learning new network address spaces through Border Gateway Protocol (BGP)

### CNI plugin issues

The Container Network Interface plugin is responsible for IP allocation:

- **Plugin health** - Verify the CNI plugin is functioning correctly
- **IPAM store consistency** - Check if the IPAM (IP Address Management) store reflects reality
- **Plugin configuration** - Ensure CNI plugin settings align with network architecture

## Approach to service accessibility issues

When services are not accessible from pods, systematically evaluate:

### Service discovery layer

Ensure the service is properly registered and discoverable:

- **Label selectors** - Confirm that service label selectors correctly match target pods
- **Endpoint creation** - Verify that endpoints are automatically created for the service and the endpoint IPs correspond to healthy target pods
- **Pod readiness** - Check that backend pods are in a ready state

### Network connectivity layer

Test connectivity at different levels starting from a pod inside the cluster and then from a client outside the cluster:

- **Pod IP reachability** - Verify direct pod IP connectivity bypassing the service
- **Application listening ports** - Ensure the application runtime is listening on the expected ports
- **Container port configuration** - Ensure containers are listening on the port matching the application runtime port
- **Service port mapping** - Confirm service port configuration matches container ports

Access to the pod logs may help verify requests are reaching the application during the above testing.

### Service proxy layer

The kube-proxy component handles service load balancing:

- **kube-proxy health** - Verify kube-proxy pods are running and healthy. In clusters using eBPF dataplanes (e.g., Cilium), kube-proxy may not be present or responsible for service routing. See [Configure Azure CNI Powered by Cilium in Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium) for more information.
- **Service mode** - Understand which proxy mode is in use (iptables, NFTables, etc.)
- **iptables/NFTables rules** - Examine whether proper forwarding rules are configured

Note that NFTables support is evolving and may depend on Kubernetes version and cluster configuration.

### Network policies and security

Access may be blocked by security controls:

- **Network policy impact** - Determine if Kubernetes Network Policies are restricting traffic
- **CNI plugin policies** - Check if the CNI plugin (e.g., Calico, Cilium) has additional policies
- **Platform-level controls** - Consider Azure NSGs or other platform security controls
- **NSG mismatches** - AKS manages a Network Security Group attached to the cluster's subnet. However, customers may also apply an additional NSG at the VNet subnet level or on individual node network interfaces, which may have rules that conflict with the AKS-managed NSG

## Approach to node-to-API server connectivity

When nodes or pods cannot reach the API server, investigate from multiple angles:

### Internal service functionality

The kubernetes service in the default namespace provides API server access within the cluster:

- **Service existence** - Confirm the `kubernetes` service exists in the default namespace and has correct endpoints
- **DNS resolution** - Verify the service DNS name (`kubernetes.default.svc.cluster.local`) resolves correctly
- **Endpoint health** - Ensure the endpoint points to a reachable API server

### Network path validation

Trace the path from node/pod to API server:

- **Routing configuration** - Check that routes exist to reach the API server
- **Azure Virtual Network configuration** - Verify the following Azure VNet configuration items:
  - Subnet address space is correctly defined and non-overlapping
  - Route tables associated with the subnet have appropriate routes to the API server
  - VNet peering (if used) is properly configured with gateway transit and forwarded traffic settings
  - Private DNS zones (for private clusters) are linked to the VNet
  - NSG rules on subnets permit traffic to the API server endpoint
- **Firewall traversal** - Ensure outbound rules permit API server communication

### Configuration-specific issues

Different cluster configurations present different challenges:

- **Private clusters** - Validate that private DNS zones are configured correctly and that DNS forwarders work
- **Custom DNS** - If using custom DNS servers, ensure they can resolve cluster-internal names
- **User-Defined Routes (UDR) with egress control** - When using UDR outbound type with restricted egress (such as through Azure Firewall or a network virtual appliance), ensure all required FQDNs and network rules are properly configured. Missing rules can prevent nodes from reaching the API server, container registries, or other essential services. Review both outbound firewall restrictions and UDR configuration to ensure they align. For detailed requirements, see [Control egress traffic for cluster nodes in AKS](/azure/aks/outbound-rules-control-egress)

### Authorization layer

Network connectivity is necessary but not sufficient:

- **Workload Identity configuration** - When using Workload Identity for Azure service authentication, verify that:
  - The managed identity associated with the workload has appropriate Azure RBAC role assignments
  - The federated identity credential is correctly configured
  - The Kubernetes service account is properly annotated with the client ID
  - Ensure that the deployment file is using the proper annotations for workload identiy (`azure.workload.identity/use: "true"`). See [Use Microsoft Entra Workload ID with Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview?tabs=dotnet) for more details.
- **Role bindings** - Verify that necessary RoleBinding and ClusterRoleBinding objects exist

## Diagnostic tools and techniques

Choose appropriate tools based on the specific problem you're investigating. The following categories represent common diagnostic approaches for network troubleshooting in AKS:

### Connectivity testing

- **Test pods** - Deploy ephemeral test pods with networking tools
- **Direct IP testing** - Use curl, telnet, or netcat to test specific endpoints
- **DNS queries** - Use dig or nslookup to validate name resolution

Note that ICMP (ping) may fail even when TCP/UDP connectivity works due to Azure load balancer and SNAT behavior. Always validate with protocol-specific tools like curl or nc.

### Traffic analysis

- **Packet captures** - Capture traffic at various points to understand flow ([Container Network Observability with Hubble](https://learn.microsoft.com/en-us/azure/aks/container-network-observability-how-to?tabs=cilium), [Inspektor Gadget](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/logs/capture-system-insights-from-aks?tabs=azurelinux30#what-is-inspektor-gadget))
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
- [Diego Casati](https://www.linkedin.com/in/dcasati/) | Azure Global Black Belt

Other contributors:

- [Michael Walters](https://www.linkedin.com/in/mrwalters1988/) | Senior Consultant
- [Ayobami Ayodeji](https://www.linkedin.com/in/ayobamiayodeji) | Senior Program Manager
- [Bahram Rushenas](https://www.linkedin.com/in/bahram-rushenas-306b9b3) | Architect
- [Steve Griffith](https://www.linkedin.com/in/stevewgriffith/) | Principal Product Manager

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
