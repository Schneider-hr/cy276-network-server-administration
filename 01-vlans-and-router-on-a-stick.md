# Topic 1 — VLANs and Router-on-a-Stick

**Goal:** understand why VLANs need something to route between them, and configure the classic solution — a single trunked link between a switch and a router, with the router using subinterfaces to act as every VLAN's gateway at once.

## The problem VLANs solve, and the problem they create

A single switch, left alone, puts every connected device in the same broadcast domain — everyone can see everyone. Splitting departments (Sales, Engineering, and so on) into separate VLANs solves that: devices in VLAN 10 can't reach devices in VLAN 20 unless something routes between them.

That's the catch — switches can't route between VLANs by themselves. Instead of running a separate physical cable from the router to the switch for every VLAN, **router-on-a-stick** uses one single cable and lets the router logically split it using subinterfaces, each one tagged for a specific VLAN via 802.1Q trunking.

## Core vocabulary

- **Access port** — belongs to exactly one VLAN; where PCs plug in.
- **Trunk port** — carries traffic for multiple VLANs at once, tagging each frame with its VLAN ID; used for switch-to-switch or switch-to-router links.
- **Encapsulation dot1Q `<vlan>`** — the tagging standard a router subinterface uses to know which VLAN's traffic it's handling.

## Base configuration

Switch side:
```
vlan 10
 name Sales
exit
interface fa0/1
 switchport mode access
 switchport access vlan 10
exit
interface gig0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

Router side — note the physical interface itself gets **no IP address**, only the subinterfaces do:
```
interface fa0/0
 no shutdown
exit
interface fa0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit
interface fa0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```

## Why this actually works — the routing path

A router interface can hold exactly one IP address, belonging to one network. Since the switch-to-router link is a trunk, the single cable is physically carrying frames from both VLAN 10 and VLAN 20, each tagged with its VLAN number. The subinterface is what lets the router make sense of the tags — `interface fa0/0.10` isn't a new physical port, it's a logical slice of Fa0/0 that only opens for VLAN 10-tagged frames, with its own IP so it can act as that VLAN's gateway.

So when a PC in VLAN 10 sends a packet to a PC in VLAN 20: the switch tags it VLAN 10 and sends it up the trunk; the router receives it on **Fa0/0.10** (arrives); it looks up the destination network in its routing table; it re-tags the packet as VLAN 20 and sends it back out **Fa0/0.20** (leaves); the switch strips the tag and delivers it to the destination PC. That "arrive on one subinterface, leave on another" step is the actual routing happening.

**A genuine constraint worth knowing:** every subinterface needs its own unique, non-overlapping subnet. If two subinterfaces both claimed addresses in `192.168.10.0/24`, the router would have no way to know which one actually owns that network — Cisco IOS refuses to let you configure it at all (`% 192.168.10.0 overlaps with FastEthernet0/0.10`).

## Practicing on a real 3-site topology

Router0 + Switch0 (Site A: Sales/Engineering VLANs) ↔ Router2 + Switch1 (Data Center: DNS/Web/DHCP servers) ↔ Router3 + Switch2 (Site C: Finance/HR VLANs).

Local LAN cabling — straight-through, PCs into access switch ports, each site's router uplinked to its switch:
```
Router0 Fa0/0  -> Switch0 Gig0/1
Switch0 Fa0/1  -> PC0 ... Fa0/4 -> PC3
Router2 Fa0/0  -> Switch1 Gig0/1
Switch1 Fa0/1  -> Server0 (DNS), Fa0/2 -> Server1 (Web), Fa0/3 -> Server2 (DHCP)
Router3 Fa0/0  -> Switch2 Gig0/1
Switch2 Fa0/1  -> PC4 ... Fa0/4 -> PC7
```

## Real problem: running out of router ports

Both routers only shipped with two built-in FastEthernet ports, already used for the LAN uplink and one WAN leg — connecting Router2 to Router3 needed a third interface that didn't exist yet. Fix: power off the router (module changes only apply while off), drag a `PT-ROUTER-NM-1FFE` (adds one FastEthernet port) into an empty module slot, power back on, confirm the new interface (`FastEthernet1/0`) actually appears with `show ip interface brief`.

The first few connection attempts still failed with "the cable cannot be connected to that port" even after the module was installed — the actual cause was clicking the wrong item in the port-selection popup (Auxiliary and Console are management-only interfaces, not data ports, and are easy to click by mistake in a crowded menu). Once the connection tool was pointed specifically at `FastEthernet1/0` on both ends, the link connected cleanly.

## Real result: ended up with a serial WAN link instead

The Router2–Router3 connection actually came up as a **serial** link (Se2/0 on both ends) rather than Ethernet — a legitimate, arguably more conventional way to represent a WAN connection in these labs. The link showed red (unconfigured) until both ends had IP addresses, and only one side needed a clock rate:

```
! Router2 — the DCE end, supplies clocking for the link
interface serial2/0
 ip address 10.10.10.5 255.255.255.252
 clock rate 64000
 no shutdown
exit
```
```
! Router3 — the DTE end, no clock rate needed
interface serial2/0
 ip address 10.10.10.6 255.255.255.252
 no shutdown
exit
```

A serial link has one end acting as DCE (supplies the clock signal that paces the link) and one as DTE (follows it) — in a real deployment this would usually be dictated by which end connects to the actual carrier/ISP equipment; in Packet Tracer it's just whichever end the cable happened to assign as DCE, identified by a small clock icon next to that interface.

**Ethernet vs. serial for the other WAN leg (Router0–Router2):** left as-is once working, deliberately not converted to match the serial link elsewhere. Nothing about RIP or static routing cares whether the underlying link is Ethernet or Serial, and converting a working link just to make the topology description tidier isn't worth the rework unless an assignment specifically requires it.
