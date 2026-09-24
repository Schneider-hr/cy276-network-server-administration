# Topic 3 — Asterk LTD: Full Enterprise Network Design Exam

**The brief:** Asterk LTD, a fictional Ghanaian company, is opening a new branch in Tarkwa and needs a complete network design. Six departments (Admin, IT, Finance, HR, Reception, Sales/Marketing), each its own VLAN with both wired and wireless access, all able to communicate with each other and with automatic (DHCP) addressing. A separate Data Center site hosts DHCP, DNS, HTTP, and FTP servers for the whole network — the DHCP server must be the only address-assigning service anywhere. The ISP-provided network for Tarkwa is `220.168.1.0`; the Data Center uses `10.200.0.0`; site-to-site connectivity between them uses `10.0.0.0`, via RIP or static routing.

This is real, complete exam work — including a full rebuild partway through once the actual grading checklist turned up requirements the first design had gotten wrong.

## The subnetting math, from first principles

An IPv4 address is 32 bits. The slash number tells you how many of those bits are locked as the network portion — the rest are free to identify individual devices.

- **`/24`**: 32 − 24 = 8 free bits → 2⁸ = **256** addresses. The last of the four numbers is the entire host portion (0–255).
- **`/27`**: 32 − 27 = 5 free bits → 2⁵ = **32** addresses per block.
- **256 ÷ 32 = 8** — one `/24` splits cleanly into 8 equal `/27` blocks, each starting at a multiple of 32:

```
220.168.1.0   – .31    (block 1)
220.168.1.32  – .63    (block 2)
220.168.1.64  – .95    (block 3)
220.168.1.96  – .127   (block 4)
220.168.1.128 – .159   (block 5)
220.168.1.160 – .191   (block 6)
220.168.1.192 – .223   (block 7)
220.168.1.224 – .255   (block 8)
```

Six departments need six blocks; a `/27` (30 usable addresses) comfortably covers a department's wired PCs plus a wireless network, with headroom, without wasting a whole `/24` on each one the way a `/26` would. That sizing choice — not the block boundaries themselves, which are fixed by the math — is the one genuine judgment call in the whole plan.

**The general version of this, for any starting prefix:** free host bits = 32 − slash number; total addresses = 2^(free bits). A `/8` or `/10` just means a vastly bigger starting budget (2²⁴ or 2²² addresses) — the method for carving it up doesn't change, you'd just assign progressively smaller chunks (site → building → department) rather than jumping straight to `/27` in one step.

## The addressing plan

| VLAN | Department | Network | Gateway (SVI) |
|---|---|---|---|
| 10 | Admin | 220.168.1.0/27 | .1 |
| 20 | IT | 220.168.1.32/27 | .33 |
| 30 | Finance | 220.168.1.64/27 | .65 |
| 40 | HR | 220.168.1.96/27 | .97 |
| 50 | Reception | 220.168.1.128/27 | .129 |
| 60 | Sales/Marketing | 220.168.1.160/27 | .161 |

Router-Main ↔ Switch-Main link: `220.168.1.192/30` (router = .193, switch = .194). Data Center: `10.200.0.0/24` behind VLAN 100, router subinterface = .1, DHCP = .2, DNS = .3, WEB = .4, FTP = .5. Site-to-site WAN: `10.0.0.0/30` (Router-Main = .1, Router-DC = .2).

## The rebuild — a real grading checklist changed two design decisions

The first working draft used `interface fastEthernet 0/24` as a plain routed port for the Data Center uplink, and assumed FastEthernet router ports throughout. The actual grading checklist for this exam specified two things the first draft got wrong:

1. **The routers use Gigabit ports** (`GigabitEthernet0/0`/`0/1`), not FastEthernet.
2. **The Data Center router needs a subinterface** (`GigabitEthernet0/1.100`) — meaning the DC LAN sits behind a VLAN (100), routed via a tagged subinterface, the same router-on-a-stick pattern from Topic 1, rather than a plain routed port.

Rebuilding from the checklist rather than the original assumption was the right call — worth noting as a real lesson: verify against the actual requirement document before building, not just a reasonable-looking topology diagram.

## Final device configurations

**SWITCH-DC** (adds VLAN 100 for the Data Center LAN):
```
enable
configure terminal
hostname SWITCH-DC

vlan 100
 name DATACENTER
exit

interface range fastEthernet 0/1-4
 switchport mode access
 switchport access vlan 100
exit

interface gigabitEthernet 0/1
 switchport mode trunk
exit
```

**ROUTER-DC** (subinterface for the DC LAN, matching the checklist):
```
enable
configure terminal
hostname ROUTER-DC

interface gigabitEthernet 0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit

interface gigabitEthernet 0/1
 no shutdown
exit

interface gigabitEthernet 0/1.100
 encapsulation dot1Q 100
 ip address 10.200.0.1 255.255.255.0
exit

router rip
 version 2
 network 10.0.0.0
 network 10.200.0.0
 no auto-summary
```

**ROUTER-MAIN:**
```
enable
configure terminal
hostname ROUTER-MAIN

interface gigabitEthernet 0/1
 ip address 220.168.1.193 255.255.255.252
 no shutdown
exit

interface gigabitEthernet 0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit

router rip
 version 2
 network 220.168.1.0
 network 10.0.0.0
 no auto-summary
```

**SWITCH-MAIN** (3560-24PS — confirmed Layer 3 capable after an initial misread; see below):
```
enable
configure terminal
hostname SWITCH-MAIN

vlan 10
 name ADMIN
exit
vlan 20
 name IT
exit
vlan 30
 name FINANCE
exit
vlan 40
 name HR
exit
vlan 50
 name RECEPTION
exit
vlan 60
 name SALES
exit

interface fastEthernet 0/1
 switchport mode trunk
exit
interface fastEthernet 0/2
 switchport mode trunk
exit
interface fastEthernet 0/3
 switchport mode trunk
exit
interface fastEthernet 0/4
 switchport mode trunk
exit
interface fastEthernet 0/5
 switchport mode trunk
exit
interface fastEthernet 0/6
 switchport mode trunk
exit

interface vlan 10
 ip address 220.168.1.1 255.255.255.224
 ip helper-address 10.200.0.2
 no shutdown
exit
interface vlan 20
 ip address 220.168.1.33 255.255.255.224
 ip helper-address 10.200.0.2
 no shutdown
exit
interface vlan 30
 ip address 220.168.1.65 255.255.255.224
 ip helper-address 10.200.0.2
 no shutdown
exit
interface vlan 40
 ip address 220.168.1.97 255.255.255.224
 ip helper-address 10.200.0.2
 no shutdown
exit
interface vlan 50
 ip address 220.168.1.129 255.255.255.224
 ip helper-address 10.200.0.2
 no shutdown
exit
interface vlan 60
 ip address 220.168.1.161 255.255.255.224
 ip helper-address 10.200.0.2
 no shutdown
exit

ip routing

interface gigabitEthernet 0/1
 no switchport
 ip address 220.168.1.194 255.255.255.252
 no shutdown
exit

router rip
 version 2
 network 220.168.1.0
 no auto-summary
```

`ip helper-address 10.200.0.2` on every VLAN's SVI is what makes DHCP work across sites at all — it forwards each VLAN's DHCP broadcast, which normally can't cross a router boundary, directly to the real DHCP server sitting in the Data Center. Without it, only a DHCP server physically on the local subnet could ever respond, and the requirement that the Data Center be the *only* address-assigning service anywhere would be impossible to satisfy.

**Each of the six access switches** (identical pattern, only VLAN and hostname change) — example for SW-ADMIN:
```
enable
configure terminal
hostname SW-ADMIN

vlan 10
 name ADMIN
exit

interface range fastEthernet 0/1-23
 switchport mode access
 switchport access vlan 10
exit

interface gigabitEthernet 0/1
 switchport mode trunk
exit
```

**Wireless access points:** one per department, plugged into an access port on that department's switch (same VLAN as the wired PCs — that's what makes wireless devices land in the same subnet/DHCP scope automatically, no separate VLAN tagging needed on the AP itself).

**Servers** — every server got a static IP (infrastructure shouldn't self-assign via the DHCP service it's part of):

| Server | IP | Gateway | DNS |
|---|---|---|---|
| DHCP-SERVER | 10.200.0.2 /24 | 10.200.0.1 | 10.200.0.3 |
| DNS-SERVER | 10.200.0.3 /24 | 10.200.0.1 | 10.200.0.3 (itself) |
| WEB-SERVER | 10.200.0.4 /24 | 10.200.0.1 | 10.200.0.3 |
| FTP-SERVER | 10.200.0.5 /24 | 10.200.0.1 | 10.200.0.3 |

DHCP-SERVER: six pools, one per department, each with the matching VLAN's gateway and a start address just past it. DNS-SERVER: five A records matching the checklist exactly — `asterk.com` and `www.asterk.com` → 10.200.0.4, `dhcp.asterk.com` → 10.200.0.2, `dns.asterk.com` → 10.200.0.3, `ftp.asterk.com` → 10.200.0.5. FTP-SERVER: one `admin` user account with full read/write/delete/rename/list permissions.

## Two real mistakes worth keeping

**Misreading the core switch model.** Early in planning, the topology's core switch was read as a 2960 (Layer 2 only, no SVI/`ip routing` support), which would have required swapping the device entirely. On closer inspection of the actual image it was a **3560-24PS** — genuinely Layer 3 capable — so no swap was needed after all. Worth double-checking a device model directly rather than assuming from a quick glance, in either direction: assuming *less* capability than a device actually has costs just as much rework as assuming more.

**Guessing port numbers instead of reading them.** Early config drafts assumed which physical port faced which device (`Fa0/24 is whichever port faces ROUTER-DC — adjust to match your actual cabling`) rather than checking. This led to a full reset (`erase startup-config` + `reload` on every device) and rebuilding the port map properly using `show cdp neighbor` on each switch — which lists every directly-connected Cisco device along with the exact local and remote port for each one, removing all guesswork in a single command. Once every access switch had a real hostname set (before that, all six showed up identically as generic "Switch" in CDP output, making them indistinguishable), the full port map resolved cleanly:

```
show cdp neighbor
```
```
Device ID       Local Intrfce   Port ID
SW-ADMIN        Fas 0/1         GigabitEthernet0/1
SW-IT           Fas 0/2         GigabitEthernet0/1
SW-FINANCE      Fas 0/3         GigabitEthernet0/1
SW-HR           Fas 0/4         GigabitEthernet0/1
SW-RECEPTION    Fas 0/5         GigabitEthernet0/1
SW-SALES        Fas 0/6         GigabitEthernet0/1
ROUTER-MAIN     Gig 0/1         GigabitEthernet0/1
```

The lesson that generalizes: `show cdp neighbor` beats reading a crowded topology diagram or a hovering-over-cables approach every time — it's authoritative, and it's one command instead of a dozen manual checks.

## Verification

```
show ip route          ! on SWITCH-MAIN, ROUTER-MAIN, ROUTER-DC — confirms every subnet is reachable
ipconfig                ! on a department PC — correct leased address, gateway matching its VLAN's SVI, DNS = 10.200.0.3
ping 10.200.0.4          ! from any department PC to the web server — confirms full end-to-end reachability
```
A browser request to `http://10.200.0.4` loading the page, and a phone joining a department's Wi-Fi landing in that same department's address range, were the final confirmations that the whole design — VLANs, wireless, DHCP relay, RIP, and the Data Center — actually worked together as one system, not just as individually-verified pieces.
