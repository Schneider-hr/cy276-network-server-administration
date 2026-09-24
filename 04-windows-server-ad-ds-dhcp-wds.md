# Topic 4 — Windows Server: AD DS, DHCP, and WDS

**Goal:** stand up a Windows Server domain controller (AD DS + DNS), configure DHCP to hand out leases automatically, set up Windows Deployment Services for network-based OS installs, and join a Windows 10 client to the domain — all in a VirtualBox lab, GUI-first (matching how the actual exam was structured).

## Before any of this: getting the VM to actually install

The lab VM itself didn't come up cleanly on the first attempt, and the failures were worth tracking down rather than working around blind:

- **Boot loop back to the UEFI Boot Manager screen** on "I will install the operating system later" — that option creates a VM with nothing attached to boot from, so it correctly has nowhere to go; the fix is attaching the installer ISO to the virtual CD drive *before* first boot, not after.
- **"Windows cannot find the Microsoft Software License Terms"** on the manual ISO path, with VMware's Easy Install failing the same run in a different way — the common factor pointed at the ISO itself, but the file turned out to be a genuine Microsoft Evaluation Center download at the expected size, not a truncated one. The actual cause was narrowed down by testing the identical ISO in a different hypervisor: it installed cleanly under **VirtualBox**, isolating the fault to a VMware Workstation compatibility quirk with that particular ISO/Easy-Install combination rather than the media — a real lesson in not trusting "corrupted download" as the answer just because it's the most common cause, when a same-file cross-hypervisor test is cheap enough to just run.
- **Ctrl+Alt+Delete affecting the host, not the guest, even with focus inside the VM window** — a known VirtualBox behavior, not a bug: the combination has to be sent through VirtualBox's own **Input → Keyboard → Insert Ctrl+Alt+Del** menu action (or Host key + Del) rather than the physical key combo, since the OS intercepts that specific combination before it ever reaches guest capture.

Once installed, the "no product key" prompt was answered by leaving it blank and continuing unactivated — correct and expected for a lab/coursework build with no licensing requirement.

## The domain controller — and a networking bug that silently broke everything

After promoting the server to a domain controller (AD DS + DNS, both come together), the very first real problem wasn't AD-related at all — it was VM networking. The server had been left on VirtualBox's **default NAT** adapter, which gave it an address like `10.0.2.15` with gateway `10.0.2.2`. That's VirtualBox's classic default-NAT signature: every VM on default NAT gets its own **isolated** NAT instance. Two VMs both individually on default NAT can each reach the internet, but **cannot see or reach each other at all** — which would have made the eventual Windows 10 client's domain join impossible without ever throwing an obvious error pointing at the real cause.

**The fix — switch to NAT Network, not plain NAT:**
1. Shut the VM down completely (not just save state)
2. In VirtualBox Manager (not inside the VM): **File → Tools → Network Manager → NAT Networks → Create** (default name `NatNetwork`, e.g. `192.168.1.0/24` with its own DHCP)
3. VM → **Settings → Network → Adapter 1 → Attached to: NAT Network**, select `NatNetwork`
4. Boot back up, `ipconfig` — now on the shared network, reachable by any other VM on the same NAT Network

This is the same fix that would apply to a plain **Bridged Adapter** setup too, with a different tradeoff: Bridged puts both VMs on the real physical LAN (works fine, but depends on the physical network allowing device-to-device traffic — campus Wi-Fi with client isolation enabled would silently break it the same way default NAT did).

## Static IP — not optional in practice, even though the wizard calls it a warning

AD DS promotion completes with a yellow warning about the DC's IP not being static — technically non-blocking, but a real problem waiting to happen: every AD DS install also runs DNS, and if the DC's address changes after a reboot (entirely possible pulling from NAT Network's own DHCP), every client pointed at the old address for DNS fails to find the domain — with no obvious link back to "the DC's IP changed."

```
ipconfig                                    ! note current IP/mask/gateway first
```
Control Panel → Network and Sharing Center → Change adapter settings → right-click adapter → Properties → IPv4 Properties → **Use the following IP address** (same values `ipconfig` just showed, so nothing already pointed at it breaks) → **Preferred DNS server: 127.0.0.1** (itself, since it now is the DNS server).

## Active Directory Users and Computers — a real logon-name mismatch

Adding a user to a group ("ITstaff") by typing a guessed logon name (`maxrobin`) failed with "Name Not Found." The cause: the "Select Users" dialog resolves by **logon name (sAMAccountName)**, not display name — and a display name like "max robin" doesn't necessarily map to any predictable logon-name pattern. Checking the user's own **Account tab → User logon name** field directly (rather than guessing dot/no-dot/concatenated variants) revealed the real logon name was simply `max`. The more reliable method going forward: **Select Users → Advanced → Find Now**, then double-click the correct account directly from the results list — no typing, no risk of a wrong guess.

## DHCP — scope creation and the "greyed out" trap

DHCP console → right-click the server → **Authorize** (if not already) → expand → right-click **IPv4 → New Scope**:

| Setting | Value |
|---|---|
| Name | Lab PCs |
| Range | 192.168.1.50 – 192.168.1.150 |
| Exclusions | none needed |
| Router (gateway) | 192.168.1.1 |
| DNS server | 192.168.1.8 (the DC) |
| Activate scope | Yes |

A scope created but showing grey (not the green "active" arrow) simply hasn't been activated yet — **right-click the scope → Activate**. If Activate itself is greyed out, the more likely cause is the DHCP server not being authorized in AD yet; an unauthorized DHCP server refuses to lease addresses even with a fully configured, active-looking scope.

## Windows Deployment Services — the DHCP/WDS port conflict, and a security judgment call

WDS role install (Integrated with Active Directory, since this server is already a DC): remote install folder on a non-system path where available (`C:\RemoteInstall` is fine for a single-disk lab VM — the "not recommended, use a separate volume" warning is a production best-practice note, not a blocker), PXE response set to **"Respond to all client computers (known and unknown)"**.

**Real conflict, handled automatically:** running DHCP and WDS on the same server means both services want control over PXE boot handling on port 67/68. The WDS install wizard detected the existing DHCP role and pre-checked the correct fix itself — **"Do not listen on DHCP and DHCPv6 ports"** and **"Configure DHCP option 60 for PXEClient"** — handing port 67/68 listening entirely to DHCP, with WDS routing PXE clients via DHCP option 60 instead. Worth understanding why those boxes are checked, not just leaving them on a default.

**A deliberate refusal, not a technical block:** the exam-provided install image set included a listing for **"FBConan's Windows X-Lite 'Optimum'"** — a third-party, modified/debloated Windows build repackaged from an unofficial source. For a system meant to be joined to a domain, deploying an image of unknown provenance is a real integrity concern (unverifiable modifications, no way to confirm nothing extra was bundled in), independent of whether it would technically work. The call made here: don't import it, and use the ISO built from Microsoft's own Media Creation Tool instead, confirming both entries were genuinely **Windows 10 Pro** (domain-join-capable either way) before proceeding, and picking the Defender-off variant specifically for lab-VM performance reasons — a reasonable tradeoff *only* because this is a disposable lab environment, not a production machine.

## Windows 10 client — domain join

With the DC reachable and DNS pointed correctly, the join itself is standard: **System Properties → Change → Domain**, enter the domain name, authenticate, reboot. Verified from both sides — `whoami /fqdn` on the client confirming the fully-qualified domain identity, and the new computer object appearing in ADUC's Computers container on the server.

## Summary

| Component | Real issue hit | Root cause / decision |
|---|---|---|
| VM networking | Two VMs couldn't see each other | Default NAT isolates each VM individually; fixed with a shared NAT Network |
| DC's own IP | AD DS promotion warning | DNS depends on this IP staying fixed; set static explicitly rather than leaving it to DHCP luck |
| AD group membership | "Name Not Found" adding a user | Display name ≠ logon name; checked the Account tab directly instead of guessing |
| DHCP scope | Scope stayed grey after creation | Needed explicit Activation; a still-greyed Activate button pointed to server authorization instead |
| WDS + DHCP on one box | Both want PXE port control | Wizard's own auto-detected fix (don't listen on DHCP ports, use option 60) — understood, not just clicked through |
| Install image selection | An unofficial "Lite" Windows build was offered | Declined it on integrity grounds despite being readily available; used the Microsoft-sourced ISO instead |
