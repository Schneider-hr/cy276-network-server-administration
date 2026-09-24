# Enterprise Network Design & Windows Server Administration (CY276)

Coursework from CY276 — Systems and Network Administration — covering Cisco routing/switching in Packet Tracer and Windows Server administration, built and tested in my own home lab (VirtualBox + Packet Tracer) rather than a shared class environment.

This repo covers the real, hands-on technical work: VLAN design, two different methods of inter-VLAN routing (router-on-a-stick and multilayer Layer 3 switching), dynamic routing with RIP, a full enterprise network design exam scenario, and Windows Server 2019/2022 administration (Active Directory Domain Services, DHCP, and Windows Deployment Services). As with the rest of my portfolio, the real troubleshooting is kept in — including a switch-model misread I had to correct, a VM networking bug that silently isolated two VMs from each other, and a suspicious "modified" Windows image I declined to use.

## Topics covered

1. [VLANs and Router-on-a-Stick](01-vlans-and-router-on-a-stick.md) — why VLANs need something to route between them, how a single trunked link plus subinterfaces solves that, and the real config across a 3-site topology (including a serial WAN link with a DCE/DTE clock-rate gotcha).
2. [RIP Routing and Multilayer Switching](02-rip-routing-and-multilayer-switching.md) — dynamic routing so three sites learn each other's networks automatically, then the alternative to router-on-a-stick: doing inter-VLAN routing directly on a Layer 3 switch instead of sending every packet out to a router and back.
3. [Asterk LTD — Full Enterprise Network Design Exam](03-asterk-ltd-enterprise-network-exam.md) — a complete, real exam scenario: six departments, six VLANs, a separate Data Center site, DHCP relay across sites, RIP site-to-site routing, and the subnetting math behind all of it, including a full rebuild after an initial design turned out to not match the real grading checklist.
4. [Windows Server: AD DS, DHCP, and WDS](04-windows-server-ad-ds-dhcp-wds.md) — standing up a domain controller, a VM networking bug that made two VMs invisible to each other, DHCP scope configuration, Windows Deployment Services with the DHCP/WDS port conflict fix, and a deliberate refusal to deploy an untrusted "modified" Windows image even though it was faster to get working.

## Tools

Cisco Packet Tracer, VirtualBox, Windows Server 2019/2022, Windows 10.

## A note on scope

This is coursework against simulated/lab infrastructure (Packet Tracer's simulated devices, my own VirtualBox VMs) — no real production network or organization is involved. "Asterk LTD" is a fictional company used as the exam's scenario framing, not a real client.
