---
title: Basic troubleshooting of DNS resolution problems in AKS
description: Learn how to create a troubleshooting workflow to fix DNS resolution problems in Azure Kubernetes Service (AKS).
author: dcasati
ms.author: dicasati
ms.date: 11/04/2025
ms.reviewer: 
editor:
ms.service: azure-kubernetes-service
ms.custom: sap:Connectivity
ms.topic: troubleshooting-general
#Customer intent: As an Azure Kubernetes user, I want to learn how to create a troubleshooting workflow so that I can fix DNS resolution problems in Azure Kubernetes Service (AKS).
---
# Troubleshoot DNS resolution problems in AKS

This article discusses how to create a troubleshooting workflow to fix Domain Name System (DNS) resolution problems in Microsoft Azure Kubernetes Service (AKS).

## Prerequisites

- The Kubernetes [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) command-line tool

   **Note:** To install kubectl by using [Azure CLI](/cli/azure/install-azure-cli), run the [az aks install-cli](/cli/azure/aks#az-aks-install-cli) command.

- The [dig](https://downloads.isc.org/isc/bind9/cur/9.19/doc/arm/html/manpages.html#dig-dns-lookup-utility) command-line tool for DNS lookup

- The [grep](https://linux.die.net/man/1/grep) command-line tool for text search

- The [Wireshark](https://www.wireshark.org) network packet analyzer

## Troubleshooting checklist

Troubleshooting DNS problems in AKS is typically a complex process. You can easily get lost in the many different steps without ever seeing a clear path forward. To help make the process simpler and more effective, use the "scientific" method to organize the work:

- Step 1. Collect the facts.

- Step 2. Develop a hypothesis.

- Step 3. Create and implement an action plan.

- Step 4. Observe the results and draw conclusions.

- Step 5. Repeat as necessary.

### Troubleshooting Step 1: Collect the facts

To better understand the context of the problem, gather facts about the specific DNS problem. By using the following baseline questions as a starting point, you can describe the nature of the problem, recognize the symptoms, and identify the scope of the problem.

| Question | Possible answers |
|--|--|
| Where does the DNS resolution fail? | <ul> <li>Pod</li> <li>Node</li> <li>Both pods and nodes</li> </ul> |
| What kind of DNS error do you get? | <ul> <li>Time-out</li> <li>No such host</li> <li>Other DNS error</li> </ul> |
| How often do the DNS errors occur? | <ul> <li>Always</li> <li>Intermittently</li> <li>In a specific pattern</li> </ul> |
| Which records are affected? | <ul> <li>A specific domain</li> <li>Any domain</li> </ul> |
| Do any custom DNS configurations exist? | <ul> <li>Custom DNS server configured on the virtual network</li> <li>Custom DNS on CoreDNS configuration</li> </ul> |
| What kind of performance problems are affecting the nodes? | <ul> <li>CPU</li> <li>Memory</li> <li>I/O throttling</li> </ul> |
| What kind of performance problems are affecting the CoreDNS pods? | <ul> <li>CPU</li> <li>Memory</li> <li>I/O throttling</li> </ul> |
| What causes DNS latency? | DNS queries that take too much time to receive a response (more the five seconds) |

To get better answers to these questions, follow this three-part process.

#### Part 1: Generate tests at different levels that reproduce the problem

The DNS resolution process for pods on AKS includes many layers. Review these layers to isolate the problem. The following layers are typical:

- CoreDNS pods
- CoreDNS service
- Nodes
- Virtual network DNS

To start the process, run tests from a test pod against each layer.

##### Test the DNS resolution at CoreDNS pod level

1. Deploy a test pod to run DNS test queries by running the following command:

   ```bash
   kubectl run -it --rm aks-test --image=mcr.microsoft.com/dotnet/runtime-deps:6.0 --restart=Never
   ```

1. Retrieve the IP addresses of the CoreDNS pods by running the following [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/) command:

   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
   ```

1. Connect to the test pod using the `kubectl exec -it aks-test -- bash` command and test the DNS resolution against each CoreDNS pod IP address by running the following commands:

   ```bash
   # Install dig
   apt-get update && apt-get install -y dnsutils
   
   # Test DNS resolution against CoreDNS pod IP
   dig @<coredns-pod-ip> kubernetes.default.svc.cluster.local
   dig @<coredns-pod-ip> <external-domain>
   ```

For more information about troubleshooting DNS resolution problems from the pod level, see [Troubleshoot DNS resolution failures from inside the pod](troubleshoot-dns-failure-from-pod-but-not-from-worker-node.md).

##### Test the DNS resolution at CoreDNS service level

1. Retrieve the CoreDNS service IP address by running the following `kubectl get` command:

   ```bash
   kubectl get svc -n kube-system kube-dns
   ```

1. On the test pod, run the following commands against the CoreDNS service IP address:

   ```bash
   # Test DNS resolution against CoreDNS service IP
   dig @<coredns-service-ip> kubernetes.default.svc.cluster.local
   dig @<coredns-service-ip> <external-domain>
   ```

##### Test the DNS resolution at node level

1. Connect to the node.

1. Run the following `grep` command to retrieve the list of upstream DNS servers that are configured:

   ```bash
   grep nameserver /etc/resolv.conf
   ```

1. Run the following text commands against each DNS that's configured in the node:

   ```bash
   # Test DNS resolution against upstream DNS servers
   dig @<upstream-dns-ip> <external-domain>
   ```

##### Test the DNS resolution at virtual network DNS level

Examine the DNS server configuration of the virtual network, and determine whether the servers can resolve the record in question.

#### Part 2: Review the health and performance of CoreDNS pods and nodes

##### Review the health and performance of CoreDNS pods

You can use `kubectl` commands to check the health and performance of CoreDNS pods. To do so, follow these steps:

1. Verify that the CoreDNS pods are running:

   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   ```

2. Check if the CoreDNS pods are overused:

   ```bash
   kubectl top pods -n kube-system -l k8s-app=kube-dns
   ```

3. Get the nodes that host the CoreDNS pods:

   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
   ```

4. Verify that the nodes aren't overused:

   ```bash
   kubectl top nodes
   ```

5. Verify the logs for the CoreDNS pods:

   ```bash
   kubectl logs -n kube-system -l k8s-app=kube-dns
   ```

> [!NOTE]
> To get more debugging information, enable verbose logs in CoreDNS. To do so, see [Troubleshooting CoreDNS customization in AKS](/azure/aks/coredns-custom#troubleshooting).

##### Review the health and performance of nodes

You might first notice DNS resolution performance problems as intermittent errors, such as time-outs. The main causes of this problem include resource exhaustion and I/O throttling within nodes that host the CoreDNS pods or the client pod.

To check whether resource exhaustion or I/O throttling is occurring, run the following [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/) command together with the `grep` command on your nodes. This series of commands lets you review the request count and compare it against the limit for each resource. If the limit percentage is more than 100 percent for a resource, that resource is overcommitted.

```bash
kubectl describe node | grep -A5 '^Name:\|^Allocated resources:' | grep -v '.kubernetes.io\|^Roles:\|Labels:'
```

The following snippet shows example output from this command:

```output
Name:               aks-nodepool1-17046773-vmss00000m
--
Allocated resources:
  (Total limits might be more than 100 percent.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                420m (9%)   1762m (41%)
  memory             420Mi (9%)  1762Mi (41%)
--
Name:               aks-nodepool1-17046773-vmss00000n
--
Allocated resources:
  (Total limits might be more than 100 percent.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                804m (18%)   6044m (140%)
  memory             804Mi (18%)  6044Mi (140%)
--
Name:               aks-nodepool1-17046773-vmss00000o
--
Allocated resources:
  (Total limits might be more than 100 percent.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                420m (9%)   1762m (41%)
  memory             420Mi (9%)  1762Mi (41%)
--
Name:               aks-ubuntu-16984727-vmss000008
--
Allocated resources:
  (Total limits might be more than 100 percent.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                420m (19%)   1762m (82%)
  memory             420Mi (19%)  1762Mi (82%)
```

To get a better picture of resource usage at the pod and node level, you can also use Container insights and other cloud-native tools in Azure. For more information, see [Monitor Kubernetes clusters using Azure services and cloud native tools](/azure/azure-monitor/containers/monitor-kubernetes).

#### Part 3: Analyze DNS traffic and review DNS resolution performance

Analyzing DNS traffic can help you understand how your AKS cluster handles the DNS queries. Ideally, you should reproduce the problem on a test pod while you capture the traffic from this test pod and on each of the CoreDNS pods.

There are two main ways to analyze DNS traffic:

- Using real-time DNS analysis tools, such as [Inspektor Gadget](../../logs/capture-system-insights-from-aks.md#what-is-inspektor-gadget), to analyze the DNS traffic in real time.
- Using traffic capture tools, such as [Retina Capture](https://retina.sh/docs/Troubleshooting/capture) and [Dumpy](https://github.com/larryTheSlap/dumpy), to collect the DNS traffic and analyze it with a network packet analyzer tool, such as Wireshark.

Both approaches aim to understand the health and performance of DNS responses using DNS response codes, response times, and other metrics. Choose the one that fits your needs best.

##### Real-time DNS traffic analysis

You can use [Inspektor Gadget](../../logs/capture-system-insights-from-aks.md#what-is-inspektor-gadget) to analyze the DNS traffic in real time. To install Inspektor Gadget to your cluster, see [How to install Inspektor Gadget in an AKS cluster](../../logs/capture-system-insights-from-aks.md#how-to-install-inspektor-gadget-in-an-aks-cluster).

To trace DNS traffic across all namespaces, use the following command:

```bash
# Get the version of Gadget
GADGET_VERSION=$(kubectl gadget version | grep Server | awk '{print $3}')
# Run the trace_dns gadget
kubectl gadget run trace_dns:$GADGET_VERSION --all-namespaces --fields "src,dst,name,qr,qtype,id,rcode,latency_ns"
```

Where `--fields` is a comma-separated list of fields to be displayed. The following list describes the fields that are used in the command:

- `src`: The source of the request with Kubernetes information (`<kind>/<namespace>/<name>:<port>`).
- `dst`: The destination of the request with Kubernetes information (`<kind>/<namespace>/<name>:<port>`).
- `name`: The name of the DNS request.
- `qr`: The query/response flag.
- `qtype`: The type of the DNS request.
- `id`: The ID of the DNS request, which is used to match the request and response.
- `rcode`: The response code of the DNS request.
- `latency_ns`: The latency of the DNS request.

The command output looks like the following:

```output
SRC                                  DST                                  NAME                        QR QTYPE          ID             RCODE           LATENCY_NS
p/default/aks-test:33141             p/kube-system/coredns-57d886c994-r2… db.contoso.com.             Q  A              c215                                  0ns
p/kube-system/coredns-57d886c994-r2… 168.63.129.16:53                     db.contoso.com.             Q  A              323c                                  0ns
168.63.129.16:53                     p/kube-system/coredns-57d886c994-r2… db.contoso.com.             R  A              323c           NameErr…           13.64ms
p/kube-system/coredns-57d886c994-r2… p/default/aks-test:33141             db.contoso.com.             R  A              c215           NameErr…               0ns
p/default/aks-test:56921             p/kube-system/coredns-57d886c994-r2… db.contoso.com.             Q  A              6574                                  0ns
p/kube-system/coredns-57d886c994-r2… p/default/aks-test:56921             db.contoso.com.             R  A              6574           NameErr…               0ns
```

You can use the `ID` field to identify whether a query has a response. The `RCODE` field shows you the response code of the DNS request. The `LATENCY_NS` field shows you the latency of the DNS request in nanoseconds. These fields can help you understand the health and performance of DNS responses.
For more information about real-time DNS analysis, see [Troubleshoot DNS failures across an AKS cluster in real time](troubleshoot-dns-failures-across-an-aks-cluster-in-real-time.md).

##### Capture DNS traffic

This section demonstrates how to use Dumpy to collect DNS traffic captures from each CoreDNS pod and a client DNS pod (in this case, the `aks-test` pod).

To collect the captures from the test client pod, run the following command:

```bash
kubectl dumpy capture pod aks-test -f "-i any port 53" --name dns-cap1-aks-test
```

To collect captures for the CoreDNS pods, run the following Dumpy command:

```bash
kubectl dumpy capture deploy coredns \
    -n kube-system \
    -f "-i any port 53" \
    --tcpdump-image nicolaka/netshoot:latest \
    --name dns-cap1-coredns
```

Ideally, you should be running captures while the problem reproduces. This requirement means that different captures might be running for different amounts of time, depending on how often you can reproduce the problem. To collect the captures, run the following commands:

```bash
mkdir dns-captures
kubectl dumpy export dns-cap1-aks-test ./dns-captures
kubectl dumpy export dns-cap1-coredns ./dns-captures -n kube-system
```

To delete the Dumpy pods, run the following Dumpy command:

```bash
kubectl dumpy delete dns-cap1-coredns -n kube-system
kubectl dumpy delete dns-cap1-aks-test
```

To merge all the CoreDNS pod captures, use the [mergecap](https://www.wireshark.org/docs/man-pages/mergecap.html) command line tool for merging capture files. The `mergecap` tool is included in the Wireshark network packet analyzer tool. Run the following `mergecap` command:

```bash
mergecap -w coredns-cap1.pcap dns-cap1-coredns-<coredns-pod-name-1>.pcap dns-cap1-coredns-<coredns-pod-name-2>.pcap [...]
```

##### DNS packet analysis for an individual CoreDNS pod

After you generate and merge your traffic capture files, you can do a DNS packet analysis of the capture files in Wireshark. Follow these steps to view the packet analysis for the traffic of an individual CoreDNS pod:

1. Select **Start**, enter **Wireshark**, and then select **Wireshark** in the search results.
1. In the **Wireshark** window, select the **File** menu, and then select **Open**.
1. Navigate to the folder that contains your capture files, select *dns-cap1-db-check-\<db-check-pod-name>.pcap* (the client-side capture file for an individual CoreDNS pod), and then select the **Open** button.
1. Select the **Statistics** menu, and then select **DNS**. The **Wireshark - DNS** dialog box appears and displays an analysis of the DNS traffic. The contents of the dialog box are shown in the following table.

   | Topic / Item                                 | Count       | Average | Min Val  | Max Val          | Rate (ms) | Percent    | Burst Rate | Burst Start |
   |----------------------------------------------|-------------|---------|----------|------------------|-----------|------------|------------|-------------|
   | &blacktriangledown; **Total Packets**        | **1066**    |         |          |                  | 1.7224    | 100%       | 0.1200     | 0.000       |
   | &emsp;&blacktriangledown; rcode              | 1066        |         |          |                  | 1.7224    | 100.00%    | 0.1200     | 0.000       |
   | &emsp;&emsp;&emsp;**Server failure**         | **17**      |         |          |                  | 0.0275    | **1.59%**  | 0.0100     | 600.000     |
   | &emsp;&emsp;&emsp;No such name               | 518         |         |          |                  | 0.8370    | 48.59%     | 0.0600     | 0.000       |
   | &emsp;&emsp;&emsp;No error                   | 531         |         |          |                  | 0.8579    | 49.81%     | 0.0600     | 0.000       |
   | &blacktriangledown; **Query/Response**       | **1066**    |         |          |                  | 1.7224    | 100%       | 0.1200     | 0.000       |
   | &emsp;&blacktriangledown; Response           | 531         |         |          |                  | 0.8579    | 49.81%     | 0.0600     | 0.000       |
   | &emsp;&emsp;&blacktriangledown; Question     | 531         |         |          |                  | 0.8579    | 100.00%    | 0.0600     | 0.000       |
   | &emsp;&emsp;&emsp;A (IPv4)                   | 531         |         |          |                  | 0.8579    | 100.00%    | 0.0600     | 0.000       |
   | &emsp;&emsp;&blacktriangledown; Answer RRs   | 531         | 1.00    | 1        | 1                | 0.8579    | 100.00%    | 0.0600     | 0.000       |
   | &emsp;&emsp;&emsp;Total RRs                  | 531         |         |          |                  | 0.8579    | 100.00%    | 0.0600     | 0.000       |
   | &emsp;&emsp;&emsp;&blacktriangledown; A      | 531         |         |          |                  | 0.8579    | 100.00%    | 0.0600     | 0.000       |
   | &emsp;&emsp;&emsp;&emsp;TTL                  | 531         | 5.02    | 5        | 30               | 0.8579    | 100.00%    | 0.0600     | 0.000       |
   | &emsp;&blacktriangledown; Query              | 535         |         |          |                  | 0.8645    | 50.19%     | 0.0600     | 0.000       |
   | &emsp;&emsp;&blacktriangledown; Question     | 535         |         |          |                  | 0.8645    | 100.00%    | 0.0600     | 0.000       |
   | &emsp;&emsp;&emsp;A (IPv4)                   | 535         |         |          |                  | 0.8645    | 100.00%    | 0.0600     | 0.000       |
   | &blacktriangledown; **Question type**        | **1066**    |         |          |                  | 1.7224    | 100%       | 0.1200     | 0.000       |
   | &emsp;A (IPv4)                               | 1066        |         |          |                  | 1.7224    | 100.00%    | 0.1200     | 0.000       |
   | &blacktriangledown; **Response time (secs)** | **531**     | 0.0091  | 0.000061 | **6.308383**     | 0.8579    | 100%       | 0.0600     | 0.000       |
   | &emsp;0.0-0.1 secs                           | 514         | 0.0015  | 0.000061 | 0.099966         | 0.8304    | 96.80%     | 0.0600     | 0.000       |
   | &emsp;**6.0-7.0 secs**                       | **17**      | 6.0925  | 6.007022 | **6.308383**     | 0.0275    | **3.20%**  | 0.0100     | 600.000     |
   | &blacktriangledown; **Service stats**        | **1066**    |         |          |                  | 1.7224    | 100%       | 0.1200     | 0.000       |
   | &emsp;Unsolicited                            | 0           |         |          |                  | 0.0000    | 0.00%      | 0.0000     | 0.000       |
   | &emsp;Retransmission                         | 0           |         |          |                  | 0.0000    | 0.00%      | 0.0000     | 0.000       |
   | &emsp;**Request not found**                  | **4**       |         |          |                  | 0.0065    | **0.38%**  | 0.0000     | 225.000     |
   | &blacktriangledown; **Payload size**         | **1066**    |         |          |                  | 1.7224    | 100%       | 0.1200     | 0.000       |
   | &emsp;0-63 bytes                             | 535         | 32.00   | 32       | 32               | 0.8645    | 50.19%     | 0.0600     | 0.000       |
   | &emsp;64-127 bytes                           | 531         | 92.87   | 32       | 194              | 0.8579    | 49.81%     | 0.0600     | 0.000       |

The DNS analysis dialog box in Wireshark shows a total of 1,066 packets. Of these packets, 17 (1.59 percent) caused a server failure. Additionally, the maximum response time was 6,308 milliseconds (6.3 seconds), and no response was received for 0.38 percent of the queries. (This total was calculated by subtracting the 49.81 percent of packets that contained responses from the 50.19 percent of packets that contained queries.)

If you enter `(dns.flags.response == 0) && ! dns.response_in` as a display filter in Wireshark, this filter displays DNS queries that didn't receive a response, as shown in the following table.

| No. | Time                       | Source    | Destination | Protocol | Length | Info                                                                                       |
|----:|----------------------------|-----------|-------------|----------|-------:|--------------------------------------------------------------------------------------------|
| 225 | 2024-04-01 16:50:40.000520 | 10.0.0.21 | 172.16.0.10 | DNS      | 80     | Standard query 0x2c67 AAAA db.contoso.com                                                  |
| 426 | 2024-04-01 16:52:47.419907 | 10.0.0.21 | 172.16.0.10 | DNS      | 132    | Standard query 0x8038 A db.contoso.com.iosffyoulcwehgo1g3egb3m4oc.jx.internal.cloudapp.net |
| 693 | 2024-04-01 16:55:23.105558 | 10.0.0.21 | 172.16.0.10 | DNS      | 132    | Standard query 0xbcb0 A db.contoso.com.iosffyoulcwehgo1g3egb3m4oc.jx.internal.cloudapp.net |
| 768 | 2024-04-01 16:56:06.512464 | 10.0.0.21 | 172.16.0.10 | DNS      | 80     | Standard query 0xe330 A db.contoso.com                                                     |

Additionally, the Wireshark status bar displays the text **Packets: 1066 - Displayed: 4 (0.4%)**. This information means that four of the 1,066 packets, or 0.4 percent, were DNS queries that never received a response. This percentage essentially matches the 0.38 percent total that you calculated earlier.

If you change the display filter to `dns.time >= 5`, the filter shows query response packets that took five seconds or more to be received, as shown in the updated table.

| No. | Time                       | Source      | Destination | Protocol | Length | Info                                                                                                      | SourcePort | Additional RRs | dns resp time |
|----:|----------------------------|-------------|-------------|----------|-------:|-----------------------------------------------------------------------------------------------------------|-----------:|---------------:|--------------:|
| 213 | 2024-04-01 16:50:32.644592 | 172.16.0.10 | 10.0.0.21   | DNS      | 132    | Standard query 0x9312 Server failure A db.contoso.com.iosffyoulcwehgo1g3egb3m4oc.jx.internal.cloudapp.net | 53         | 0              | 6.229941      |
| 320 | 2024-04-01 16:51:55.053896 | 172.16.0.10 | 10.0.0.21   | DNS      | 80     | Standard query 0xe5ce Server failure A db.contoso.com                                                     | 53         | 0              | 6.065555      |
| 328 | 2024-04-01 16:51:55.113619 | 172.16.0.10 | 10.0.0.21   | DNS      | 132    | Standard query 0x6681 Server failure A db.contoso.com.iosffyoulcwehgo1g3egb3m4oc.jx.internal.cloudapp.net | 53         | 0              | 6.029641      |
| 335 | 2024-04-01 16:52:02.553811 | 172.16.0.10 | 10.0.0.21   | DNS      | 80     | Standard query 0x6cf6 Server failure A db.contoso.com                                                     | 53         | 0              | 6.500504      |
| 541 | 2024-04-01 16:53:53.423838 | 172.16.0.10 | 10.0.0.21   | DNS      | 80     | Standard query 0x07b3 Server failure AAAA db.contoso.com                                                  | 53         | 0              | 6.022195      |
| 553 | 2024-04-01 16:54:05.165234 | 172.16.0.10 | 10.0.0.21   | DNS      | 132    | Standard query 0x1ea0 Server failure A db.contoso.com.iosffyoulcwehgo1g3egb3m4oc.jx.internal.cloudapp.net | 53         | 0              | 6.007022      |
| 774 | 2024-04-01 16:56:17.553531 | 172.16.0.10 | 10.0.0.21   | DNS      | 80     | Standard query 0xa20f Server failure AAAA db.contoso.com                                                  | 53         | 0              | 6.014926      |
| 891 | 2024-04-01 16:56:44.442334 | 172.16.0.10 | 10.0.0.21   | DNS      | 132    | Standard query 0xa279 Server failure A db.contoso.com.iosffyoulcwehgo1g3egb3m4oc.jx.internal.cloudapp.net | 53         | 0              | 6.044552      |

After you change the display filter, the status bar is updated to show the text **Packets: 1066 - Displayed: 8 (0.8%)**. Therefore, eight of the 1,066 packets, or 0.8 percent, were DNS responses that took five seconds or more to be received. However, on most clients, the default DNS time-out value is expected to be five seconds. This expectation means that, although the CoreDNS pods processed and delivered the eight responses, the client already ended the session by issuing a "timed out" error message. The **Info** column in the filtered results shows that all eight packets caused a server failure.

##### DNS packet analysis for all CoreDNS pods

In Wireshark, open the capture file of the CoreDNS pods that you merged earlier (*coredns-cap1.pcap*), and then open the DNS analysis, as described in the previous section. A Wireshark dialog box appears that displays the following table.

| Topic / Item                                 | Count       | Average | Min Val  | Max Val          | Rate (ms) | Percent    | Burst Rate | Burst Start |
|----------------------------------------------|-------------|---------|----------|------------------|-----------|------------|------------|-------------|
| &blacktriangledown; **Total Packets**        | **4540233** |         |          |                  | 7.3387    | 100%       | 84.7800    | 592.950     |
| &emsp;&blacktriangledown; rcode              | 4540233     |         |          |                  | 7.3387    | 100.00%    | 84.7800    | 592.950     |
| &emsp;&emsp;&emsp;**Server failure**         | **121781**  |         |          |                  | 0.1968    | **2.68%**  | 8.4600     | 599.143     |
| &emsp;&emsp;&emsp;No such name               | 574658      |         |          |                  | 0.9289    | 12.66%     | 10.9800    | 592.950     |
| &emsp;&emsp;&emsp;No error                   | 3843794     |         |          |                  | 6.2130    | 84.66%     | 73.2500    | 592.950     |

## Common DNS resolution issues

### Client can't reach the API server

These errors involve connection problems that occur when you can't reach an Azure Kubernetes Service (AKS) cluster's API server through the Kubernetes cluster command-line tool (kubectl) or any other tool, like the REST API via a programming language.

**Error**

You might see errors that look like these:

```output
Unable to connect to the server: dial tcp <API-server-IP>:443: i/o timeout 
```

```output
Unable to connect to the server: dial tcp <API-server-IP>:443: connectex: A connection attempt
failed because the connected party did not properly respond after a period, or established 
connection failed because connected host has failed to respond. 
```

**Cause 1** 

It's possible that IP ranges authorized by the API server are enabled on the cluster's API server, but the client's IP address isn't included in those IP ranges. To determine whether IP ranges are enabled, use the following `az aks show` command in Azure CLI. If the IP ranges are enabled, the command will produce a list of IP ranges. 

```azurecli
az aks show --resource-group <cluster-resource-group> \ 
    --name <cluster-name> \ 
    --query apiServerAccessProfile.authorizedIpRanges 
```

**Solution 1**

Ensure that your client's IP address is within the ranges authorized by the cluster's API server:

1. Find your local IP address. For information on how to find it on Windows and Linux, see [How to find my IP](/azure/aks/api-server-authorized-ip-ranges#how-to-find-my-ip-to-include-in---api-server-authorized-ip-ranges).

1. Update the range that's authorized by the API server by using the `az aks update` command in Azure CLI. Authorize your client's IP address. For instructions, see [Update a cluster's API server authorized IP ranges](/azure/aks/api-server-authorized-ip-ranges#update-a-clusters-api-server-authorized-ip-ranges).  

**Cause 2**

If your AKS cluster is a private cluster, the API server endpoint doesn't have a public IP address. You need to use a VM that has network access to the AKS cluster's virtual network.  

**Solution 2**

For information on how to resolve this problem, see [options for connecting to a private cluster](/azure/aks/private-clusters#options-for-connecting-to-the-private-cluster).

### Pod fails to allocate the IP address

**Error**

The Pod is stuck in the `ContainerCreating` state, and its events report a `Failed to allocate address` error:

```output
Normal   SandboxChanged          5m (x74 over 8m)    kubelet, k8s-agentpool-00011101-0 Pod sandbox
changed, it will be killed and re-created. 

  Warning  FailedCreatePodSandBox  21s (x204 over 8m)  kubelet, k8s-agentpool-00011101-0 Failed 
create pod sandbox: rpc error: code = Unknown desc = NetworkPlugin cni failed to set up pod 
"deployment-azuredisk6-874857994-487td_default" network: Failed to allocate address: Failed to 
delegate: Failed to allocate address: No available addresses 
```

Or a `not enough IPs available` error:

```output
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox 
'ac1b1354613465324654c1588ac64f1a756aa32f14732246ac4132133ba21364': plugin type='azure-vnet' 
failed (add): IPAM Invoker Add failed with error: Failed to get IP address from CNS with error: 
%w: AllocateIPConfig failed: not enough IPs available for 9c6a7f37-dd43-4f7c-a01f-1ff41653609c, 
waiting on Azure CNS to allocate more with NC Status: , IP config request is [IPConfigRequest: 
DesiredIPAddress , PodInterfaceID a1876957-eth0, InfraContainerID 
a1231464635654a123646565456cc146841c1313546a515432161a45a5316541, OrchestratorContext 
{'PodName':'a_podname','PodNamespace':'my_namespace'}]
```

Check the allocated IP addresses in the plugin IPAM store. You might find that all IP addresses are allocated, but the number is much less than the number of running Pods:

**If using [kubenet](/azure/aks/configure-kubenet):**

```bash
# Kubenet, for example. The actual path of the IPAM store file depends on network plugin implementation. 
chroot /host/
ls -la "/var/lib/cni/networks/$(ls /var/lib/cni/networks/ | grep -e "k8s-pod-network" -e "kubenet")" | grep -v -e "lock\|last\|total" -e '\.$' | wc -l
244
```

> [!NOTE]
> For kubenet without Calico, the path is `/var/lib/cni/networks/kubenet`. For kubenet with Calico, the path is `/var/lib/cni/networks/k8s-pod-network`. The script above will auto select the path while executing the command.

```bash
# Check running Pod IPs
kubectl get pods --field-selector spec.nodeName=<your_node_name>,status.phase=Running -A -o json | jq -r '.items[] | select(.spec.hostNetwork != 'true').status.podIP' | wc -l
7 
```

**If using [Azure CNI for dynamic IP allocation](/azure/aks/configure-azure-cni-dynamic-ip-allocation):**

```bash
kubectl get nnc -n kube-system -o wide
```

```output
NAME                               REQUESTED IPS  ALLOCATED IPS  SUBNET  SUBNET CIDR   NC ID                                 NC MODE  NC TYPE  NC VERSION
aks-agentpool-12345678-vmss000000  32             32             subnet  10.18.0.0/15  559e239d-f744-4f84-bbe0-c7c6fd12ec17  dynamic  vnet     1
```

```bash
# Check running Pod IPs
kubectl get pods --field-selector spec.nodeName=aks-agentpool-12345678-vmss000000,status.phase=Running -A -o json | jq -r '.items[] | select(.spec.hostNetwork != 'true').status.podIP' | wc -l
21
```

**Cause 1**

This error can be caused by a bug in the network plugin. The plugin can fail to deallocate the IP address when a Pod is terminated.  

**Solution 1**

Contact Microsoft for a workaround or fix.  

**Cause 2**

Pod creation is much faster than garbage collection of terminated Pods.

**Solution 2**

Configure fast garbage collection for the kubelet. For instructions, see [the Kubernetes garbage collection documentation](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#containers-images).  

### Service not accessible within Pods

The first step to resolving this problem is to check whether endpoints have been created automatically for the service:

```bash
kubectl get endpoints <service-name> 
```

If you get an empty result, your service's label selector might be wrong. Confirm that the label is correct:

```bash
# Query Service LabelSelector. 
kubectl get svc <service-name> -o jsonpath='{.spec.selector}' 

# Get Pods matching the LabelSelector and check whether they're running. 
kubectl get pods -l key1=value1,key2=value2 
```

If the preceding steps return expected values:

- Check whether the Pod `containerPort` is the same as the service `containerPort`. 
- Check whether `podIP:containerPort` is working:

   ```bash
   # Testing via cURL. 
   curl -v telnet://<Pod-IP>:<containerPort>

   # Testing via Telnet. 
   telnet <Pod-IP>:<containerPort> 
   ```

These are some other potential causes of service problems:

- The container isn't listening to the specified `containerPort`. (Check the Pod description.)
- A CNI plugin error or network route error is occurring.
- kube-proxy isn't running or iptables rules aren't configured correctly.
- Network Policies is dropping traffic. For information on applying and testing Network Policies, see [Azure Kubernetes Network Policies overview](/azure/virtual-network/kubernetes-network-policies).
  - If you're using Calico as your network plugin, you can capture network policy traffic as well. For more information about configuration, see the [Calico site](https://projectcalico.docs.tigera.io/security/calico-network-policy#generate-logs-for-specific-traffic).

### Nodes can't reach the API server

Many add-ons and containers need to access the Kubernetes API (for example, kube-dns and operator containers). If errors occur during this process, the following steps can help you determine the source of the problem.  

First, confirm whether the Kubernetes API is accessible within Pods:

```bash
kubectl run curl --image=mcr.microsoft.com/azure-cli -i -t --restart=Never --overrides='[{"op":"add","path":"/spec/containers/0/resources","value":{"limits":{"cpu":"200m","memory":"128Mi"}}}]' --override-type json --command -- sh
```

Then execute the following from within the container that you now are shelled into.

```bash
# If you don't see a command prompt, try selecting Enter. 
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) 
curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api/v1/namespaces/default/pods
```

Healthy output will look similar to the following.

```output
{ 
  "kind": "PodList", 
  "apiVersion": "v1", 
  "metadata": { 
    "selfLink": "/api/v1/namespaces/default/pods", 
    "resourceVersion": "2285" 
  }, 
  "items": [ 
   ... 
  ] 
} 
```

If an error occurs, check whether the `kubernetes-internal` service and its endpoints are healthy:

```bash
kubectl get service kubernetes-internal
```

```output
NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE 
kubernetes-internal ClusterIP   10.96.0.1    <none>        443/TCP   25m 
```

```bash
kubectl get endpoints kubernetes-internal
```

```output
NAME                ENDPOINTS          AGE 
kubernetes-internal 172.17.0.62:6443   25m 
```

If both tests return responses like the preceding ones, and the IP and port returned match the ones for your container, it's likely that kube-apiserver isn't running or is blocked from the network.

There are four main reasons why the access might be blocked:

- Your network policies. They might be preventing access to the API management plane. For information on testing Network Policies, see [Network Policies overview](/azure/virtual-network/kubernetes-network-policies).
- Your API's allowed IP addresses. For information about resolving this problem, see [Update a cluster's API server authorized IP ranges](/azure/aks/api-server-authorized-ip-ranges#update-a-clusters-api-server-authorized-ip-ranges).
- Your private firewall. If you route the AKS traffic through a private firewall, make sure there are outbound rules as described in [Required outbound network rules and FQDNs for AKS clusters](/azure/aks/limit-egress-traffic#required-outbound-network-rules-and-fqdns-for-aks-clusters).  
- Your private DNS. If you're hosting a private cluster and you're unable to reach the API server, your DNS forwarders might not be configured properly. To ensure proper communication, complete the steps in [Hub and spoke with custom DNS](/azure/aks/private-clusters#hub-and-spoke-with-custom-dns).  

You can also check kube-apiserver logs by using Container insights. For information on querying kube-apiserver logs, and many other queries, see [How to query logs from Container insights](/azure/azure-monitor/containers/container-insights-log-query#resource-logs).

Finally, you can check the kube-apiserver status and its logs on the cluster itself:

```bash
# Check kube-apiserver status. 
kubectl -n kube-system get pod -l component=kube-apiserver 

# Get kube-apiserver logs. 
PODNAME=$(kubectl -n kube-system get pod -l component=kube-apiserver -o jsonpath='{.items[0].metadata.name}')
kubectl -n kube-system logs $PODNAME --tail 100
```

If a `403 - Forbidden` error returns, kube-apiserver is probably configured with role-based access control (RBAC) and your container's `ServiceAccount` probably isn't authorized to access resources. In this case, you should create appropriate `RoleBinding` and `ClusterRoleBinding` objects. For information about roles and role bindings, see [Access and identity](/azure/aks/concepts-identity#roles-and-clusterroles). For examples of how to configure RBAC on your cluster, see [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac).

## Contributors

*This article is maintained by Microsoft. It was originally written by the following contributors.*

Principal author:

- [Michael Walters](https://www.linkedin.com/in/mrwalters1988/) | Senior Consultant

Other contributors:

- [Ayobami Ayodeji](https://www.linkedin.com/in/ayobamiayodeji) | Senior Program Manager
- [Bahram Rushenas](https://www.linkedin.com/in/bahram-rushenas-306b9b3) | Architect

## Next steps

- [Network concepts for applications in AKS](/azure/aks/concepts-network)
- [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Kubernetes Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking)
- [Choose the best networking plugin for AKS](/training/modules/choose-network-plugin-aks)

## Related resources

- [AKS architecture design](../../reference-architectures/containers/aks-start-here.md)
- [Lift and shift to containers with AKS](/azure/cloud-adoption-framework/migrate/)
- [Baseline architecture for an AKS cluster](/azure/architecture/reference-architectures/containers/aks/baseline-aks)
- [AKS baseline for multiregion clusters](../../reference-architectures/containers/aks-multi-region/aks-multi-cluster.yml)
- [AKS day-2 operations guide](../../operator-guides/aks/day-2-operations-guide.md)
  - [Triage practices](../../operator-guides/aks/aks-triage-practices.md)
  - [Patching and upgrade guidance](../../operator-guides/aks/aks-upgrade-practices.md)
  - [Monitoring AKS with Azure Monitor](/azure/aks/monitor-aks)
