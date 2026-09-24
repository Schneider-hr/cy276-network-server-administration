# Topic 2 — RIP Routing and Multilayer Switching

**Goal:** get three separate sites automatically learning each other's networks with dynamic routing, then learn the alternative to router-on-a-stick — doing inter-VLAN routing directly on the switch instead of sending every packet out to a router and back.

## RIP — why it exists

Without dynamic routing, every router in a multi-site network needs a manually typed static route to every network it doesn't touch directly — and every time the topology changes, every affected router needs updating by hand. RIP (Routing Information Protocol) solves this: every router announces "here's the full list of every network I can personally touch." Neighbors add that to their own routing table and re-announce their own combined list onward. Eventually, every router in the topology knows how to reach every network — even ones it's not directly connected to — without a single static route typed anywhere.

## Real configuration across three sites

```
! Router0 — touches VLAN 10, VLAN 20, and the WAN link to Router2
router rip
 version 2
 network 192.168.10.0
 network 192.168.20.0
 network 10.0.0.0
 no auto-summary
```
```
! Router2 — touches the server subnet and both WAN links
router rip
 version 2
 network 192.168.30.0
 network 10.0.0.0
 no auto-summary
```
```
! Router3 — touches VLAN 40 and its WAN link to Router2
router rip
 version 2
 network 192.168.40.0
 network 10.0.0.0
 no auto-summary
```

Two details that matter:
- **`version 2`** forces RIPv2, which actually carries subnet mask information in its routing updates (RIPv1 doesn't, and would badly mishandle a VLAN-subnetted network like this one).
- **`no auto-summary`** turns off RIP's old habit of collapsing subnets back to their classful boundaries when advertising to neighbors — without it, the VLAN 10/VLAN 20 split would effectively get erased in what other routers hear about. Always include this on any modern subnetted network.

**Why `network 10.0.0.0` alone covers two separate WAN links:** RIP needs a `network` line for every network a router directly touches, not just the "most important" one. Router2's two WAN legs (`10.10.10.1/.2` and `10.10.10.5/.6`) are both inside the same classful `10.x.x.x` range, so the single `network 10.0.0.0` line advertises both of them at once — RIP's `network` command works at the classful level regardless of how the addresses are actually subnetted underneath.

Verified with `show ip route` on each router (checking for `R`-flagged routes to the networks that router doesn't directly touch) and with real ping tests, confirming end-to-end reachability across all three sites through two RIP-learned hops.

## Multilayer switching — the alternative to router-on-a-stick

**The weakness router-on-a-stick has:** every single packet moving between VLANs has to travel up to the router and back down, even if the router is sitting right next to the switch. That one link becomes a bottleneck, and the router processes every inter-VLAN packet in software.

**The fix — a Layer 3 (multilayer) switch:** a normal switch only understands MAC addresses and has no concept of an IP address. A multilayer switch adds routing capability directly into the switch hardware, so inter-VLAN traffic gets routed internally at much higher speed, with no trunk-to-a-router bottleneck.

New vocabulary:
- **SVI (Switched Virtual Interface)** — a virtual Layer 3 interface configured directly on a VLAN, on the switch itself. Instead of a router subinterface (`fa0/0.10`), you configure `interface vlan 10` directly and give *that* an IP.
- **`ip routing`** — a global command that must be explicitly enabled for the switch to actually forward traffic between VLANs at all. Without it, SVIs can hold IP addresses but the switch still won't route between them.

| Router-on-a-Stick | Multilayer Switch |
|---|---|
| Router has subinterfaces (`fa0/0.10`, `fa0/0.20`) | Switch has SVIs (`interface vlan 10`, `interface vlan 20`) |
| Needs `encapsulation dot1Q` on the router | No encapsulation needed — routing happens inside the switch |
| Trunk to a router carries all inter-VLAN traffic | No trunk-to-router needed for inter-VLAN routing — it's internal |
| Software-based routing over one shared link | Hardware-based routing, no single-link bottleneck |

**One genuine limitation worth knowing:** a multilayer switch typically can't replace a router entirely. It's excellent at fast internal VLAN-to-VLAN routing, but usually can't handle WAN connections (serial links), NAT, or more advanced routing/VPN functionality. In a real design the router's role shifts to purely WAN/external connectivity, while the multilayer switch handles all local inter-VLAN traffic.

**A real constraint hit directly:** not every switch model supports this. The 2960 series (even PoE variants) is Layer 2 only in Packet Tracer — `ip routing` and SVIs simply don't exist as configurable options on it. A **3560** or higher is required. This came up for real during the Asterk LTD exam build (see [Topic 3](03-asterk-ltd-enterprise-network-exam.md)), where an initial misread of the switch model in a topology diagram (2960 vs. 3560) had to be corrected before the SVI-based design could work at all.

The working multilayer switch configuration — VLANs, SVIs, `ip routing`, and RIP running on top of it — is the real SWITCH-MAIN configuration built for that exam, covered in full in Topic 3.
