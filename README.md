# Network Troubleshooting Lab: 60% Packet Loss

> **Credits:** Recreated from the troubleshooting scenario *"60% Packet Loss in a Network"* by **Praphul Mishra (PM Networking)**. <br>
> 📺 [Watch the original video here](<[YouTube link](https://youtu.be/wcwAv8V09oM?si=IFzDBqkv0cD7i1G4)>)

This lab simulates a common real-world issue where two remote sites connected across a service provider network are unable to ping each other. While the physical and data-link layers look perfectly healthy, traffic between the sites fails in erratic, unexpected ways.

💾 **Download the Lab:** Grab [broken_net.pkt](broken_net.pkt) and try to solve it yourself before looking at the solution!

---

 ## The Scenario

Picture two corporate offices:
* **Site A:** Sits behind **R1** on the `10.1.1.0/24` LAN.
* **Site B:** Sits behind **R2** on the `20.1.1.0/24` LAN.

Neither router connects to the other directly. Instead, both hand their traffic to a service provider in the middle (represented by the MPLS cloud), which is responsible for routing packets end-to-end.

## Topology

### IP Addressing Architecture
* **R1:** LAN: `10.1.1.1/24` | WAN: `1.1.1.1/24`
* **MPLS Provider:** FastEthernet0/0: `1.1.1.2/24` | FastEthernet0/1: `2.2.2.2/24`
* **R2:** WAN: `2.2.2.1/24` | LAN: `20.1.1.1/24`

*Note: LANs are configured as Loopback interfaces so they remain permanently in an `up/up` state.*

> 💡 **NOTE: Why use a router for the "MPLS Cloud" instead of Packet Tracer's Cloud object?**
> The underlying issue in this scenario lives entirely within a routing table. Packet Tracer's generic Cloud object lacks a functional routing engine to break. Utilizing a standard Cisco router perfectly replicates a real-world provider edge (PE) environment.

---

## The Symptoms

When testing end-to-end reachability, three strange symptoms occur:

1. **Intermittent Drops:** `R2# ping 10.1.1.1` results in **60% packet loss** (2/5 success rate) with an alternating `.!.!.` pattern.
2. **Active Unreachability:** `R1# ping 20.1.1.1` results in **0% success**, but instead of standard timeouts (`.....`), it returns `U.U.U`. The network is actively telling us it's unreachable.
3. **Total Failure on Source Change:** `R2# ping 10.1.1.1 source 20.1.1.1` results in **100% packet loss** (`.....`). Simply altering the source address causes a partially working ping to die completely.

### Your Mission
* Why does the ping behave differently depending on direction and source?
* What does the `U` symbol indicate, and which device is generating it?
* Why does adding the `source` modifier completely kill the connection?
* **Identify and fix both independent faults hiding in this network.**

---

## Solution & Walkthrough

<details>
<summary> <b>Spoiler Warning: Click to expand full solution</b></summary>

Both underlying faults are **return-path** routing issues. When troubleshooting asymmetric behavior, always ask: *Can the reply packet find its way back home?* 

### Fault 1: The Active Unreachable (`U.U.U`)
The `U` character stands for **ICMP Destination Unreachable**. This means a router along the path actively looked at its routing table, found no matching route, dropped the packet, and sent an error message back to the source.

Let's isolate the traffic using debugging and a sourced ping:
```
R1# debug ip icmp
R2# ping 10.1.1.1 source 20.1.1.1
```

R1 output analysis:
```
ICMP: echo reply sent, src 10.1.1.1, dst 20.1.1.1
ICMP: host unreachable rcv from 1.1.1.2
```
R1 is successfully generating the reply, but the provider next-hop (1.1.1.2) is immediately throwing it away.

Checking the MPLS provider router's routing table confirms the missing network entry:

```
MPLS# show ip route 20.1.1.0
% Network not in table
```
