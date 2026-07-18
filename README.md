# Trace the Fault #1 — 60% Packet Loss in a Network

A Packet Tracer troubleshooting lab. Download `broken.pkt`, reproduce the symptoms, and diagnose the two faults hiding inside. Full solution at the bottom.

Recreated from the scenario **"60% Packet Loss in a Network"** by **Praphul Mishra (PM Networking)** — original video: `<YouTube link>`.

## Topology

```
 10.1.1.0/24                                              20.1.1.0/24
     |                                                          |
   [ R1 ]--1.1.1.1 --- 1.1.1.2--[ MPLS ]--2.2.2.2 --- 2.2.2.1--[ R2 ]
             (WAN)               (provider)             (WAN)
```

- R1 LAN 10.1.1.1/24 | R1 WAN 1.1.1.1/24
- MPLS 1.1.1.2/24 (to R1), 2.2.2.2/24 (to R2)
- R2 WAN 2.2.2.1/24 | R2 LAN 20.1.1.1/24

LANs are configured as loopbacks so they stay up/up. All interfaces up. Nothing shut down.

> **Why a router for "MPLS" and not PT's Cloud object?**
> The whole bug lives in a routing table — a route that should exist, doesn't. Packet Tracer's Cloud device has no routing table to break, so a plain router is the only thing that reproduces it. The cloud in the diagram just means "the provider" — behind it are real routers anyway.

## The symptoms

1. `R2# ping 10.1.1.1` → **60% loss** (2/5), pattern `.!.!.`
2. `R1# ping 20.1.1.1` → **0%**, shows `U.U.U`
3. `R2# ping 10.1.1.1 source 20.1.1.1` → **100% loss** (`.....`)

All links up. Nothing shut down. So where do you start?

## Your job

- Why does a ping work one way but fail the other?
- What does `U` mean, and who is sending it?
- Why does adding `source` change the result?
- There are **two** separate faults. Find both, and fix both.

Try it before scrolling. Drop your reasoning in the LinkedIn comments.

---

## SOLUTION (spoilers)

Both faults are **return-path** problems. A ping is a round trip — always ask: *can the reply get home?* Start where the network is loudest.

### Fault 1 — the 0% / `U.U.U` (the network confesses)

`U` = "no route." The network is telling you something's unreachable, so follow it.

```
R1# debug ip icmp
R2# ping 10.1.1.1 source 20.1.1.1
```
R1 shows `echo reply sent, src 10.1.1.1, dst 20.1.1.1` **and** `host unreachable rcv from 1.1.1.2`.

Read that: R1 **is** replying — MPLS is throwing the reply away. So the fault is on MPLS, not R1. Check it:
```
MPLS# show ip route 20.1.1.0     ->  no route
```
The provider has no route back to that LAN. Forward path fine, return path dead.

**Fix — on MPLS:**
```
configure terminal
ip route 20.1.1.0 255.255.255.0 2.2.2.1
end
```
Sourced ping → 5/5. The `U` disappears instantly.

### Fault 2 — the 60% loss (the quiet one, you hunt it)

After fixing Fault 1, the default `R2# ping 10.1.1.1` is **still** lossy — and this time there's **no error**. No `U`, nothing confesses. That silence is the clue.

Partial, alternating loss (`.!.!.`) is the fingerprint of **load-sharing over two paths where one is a black hole.** Look at R1's paths back to R2's WAN:
```
R1# show ip route 2.2.2.0
```
Two equal-cost routes appear — one via `1.1.1.2` (real MPLS), one via `1.1.1.100` (?). Test the suspect:
```
R1# ping 1.1.1.100     ->  0%, nobody there
R1# show ip arp        ->  no ARP entry for 1.1.1.100
```
No ARP entry = no Layer-2 next-hop. Half the replies get hashed onto a dead next-hop and dropped silently.

**Fix — on R1, remove the bogus path:**
```
configure terminal
no ip route 2.2.2.0 255.255.255.0 1.1.1.100
end
```
Default ping → 5/5.

### Verify everything
```
R1#  ping 20.1.1.1                       ->  !!!!!
R2#  ping 10.1.1.1 source 20.1.1.1       ->  !!!!!
R2#  ping 10.1.1.1                        ->  !!!!!
```

## The through-line

Both faults were invisible from the WAN side and only showed up when traffic depended on a subnet the return path couldn't reach. One was a missing route, one was a blackhole in a load-share — but the diagnostic move was identical both times: **follow the reply, not the request.**

The network hands you the loud fault (the `U`). You have to hunt the quiet one (the 60% loss) yourself.

### Real-world mapping
In a true MPLS L3VPN, Fault 1 = the PE missing the customer route (VRF route not learned, or the CE LAN not redistributed into BGP). Same lesson: make the far subnet reachable on the *return* path. Don't stop at "the link is up."

---
*Recreated in Packet Tracer as a troubleshooting exercise. Original scenario by Praphul Mishra (PM Networking).*
