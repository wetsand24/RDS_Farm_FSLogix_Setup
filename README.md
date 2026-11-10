# RDS farm with Active Directory, FSLogix, RD Gateway and Let's Encrypt

Everything from the video, values and steps, so you can follow on your own network. Lab only: read "Before you use this for real" first.


▶️ Video: https://youtu.be/W8o6kdQXKHY

## What to change, what to keep
| Value | Change it? |
|---|---|
| Server names DC01, FS01, RDSCB01, RDSGW01, RDS01, RDS02, CLIENT | Keep |
| 10.37.0.80 to .86 | Change to free addresses on your network, outside your router's DHCP range |
| Gateway 10.37.0.1, mask 255.255.255.0 | Change to your router and your mask |
| ad.itera.ca | **Must change** to a subdomain of a domain you own, such as ad.yourdomain.com |
| ITERA | Any short name. The wizard suggests the first part of the domain, so type yours in |
| Profiles$, Desktops, RDS Users, RDS Servers, RDS Farm | Keep |
| alice, bob, passwords | Any names. Your own long passwords |
| rds.itera.ca | **Must change** to your own public name, everywhere. Keep it outside the AD subdomain |
| Router rule target | **Must change** to your RDSGW01's LAN address |

## Servers
| Name | Job | RAM | IP | Gateway | DNS |
|---|---|---|---|---|---|
| DC01 | Domain controller, DNS | 4 GB | 10.37.0.80 | 10.37.0.1 | 10.37.0.80 |
| FS01 | File server, the profile share | 4 GB | 10.37.0.84 | 10.37.0.1 | 10.37.0.80 |
| RDSCB01 | Connection broker, RD Licensing | 4 GB | 10.37.0.85 | 10.37.0.1 | 10.37.0.80 |
| RDSGW01 | RD Gateway, RD Web Access, the only internet-facing box | 4 GB | 10.37.0.86 | 10.37.0.1 | 10.37.0.80 |
| RDS01 | Session host | 8 GB | 10.37.0.81 | 10.37.0.1 | 10.37.0.80 |
| RDS02 | Session host | 8 GB | 10.37.0.82 | 10.37.0.1 | 10.37.0.80 |
| CLIENT | Test client, Windows 11 with TPM on, not domain joined | 4 GB | 10.37.0.83 | 10.37.0.1 | 10.37.0.80 |

The six servers are Windows Server 2025 Standard Evaluation (Desktop Experience). All Generation 2, on a Hyper-V external switch. In the video the host is a Windows Server 2025 box with the Hyper-V role and two network adapters: one for managing the host, one for the external switch. Physical servers or VMs on another hypervisor work the same from step 2 on. A farm can have as many session hosts as you want, two is the minimum to see the profile follow the user. Use a wired adapter if you can. Install each VM fresh. If you clone, Sysprep first, because duplicate SIDs cause authentication failures on Server 2025.

## Profile share (FS01, C:\Profiles, shared as Profiles$)
The dollar sign hides the share from browsing. Path: `\\fs01.ad.itera.ca\Profiles$`.

Share permissions: Everyone removed, `RDS Users` Change, `Domain Admins` Full.

Security, Advanced, Disable inheritance, Convert, then remove the two Users entries:
| Entry | Access | Applies to |
|---|---|---|
| Administrators | Full | This folder, subfolders and files |
| SYSTEM | Full | This folder, subfolders and files |
| CREATOR OWNER | Modify | Subfolders and files only |
| RDS Users | Modify | This folder only |

## Deployment (run from RDSCB01)
Add Servers: RDSGW01, RDS01, RDS02. Add Roles and Features, Remote Desktop Services installation, Standard deployment, Session-based desktop. Connection broker RDSCB01, Web Access RDSGW01, session hosts RDS01 and RDS02. Then add RD Licensing on RDSCB01, and Edit Deployment Properties, licensing mode Per User.

Collection `Desktops`: both session hosts, `RDS Users` instead of Domain Users, user profile disks unticked. Session page: end a disconnected session after 2 hours, idle limit 1 hour.

## FSLogix
Install `x64\Release\FSLogixAppsSetup.exe` on both session hosts. It didn't ask for a restart on my build, but Microsoft's steps end with a reboot, so do one.

Group policy is the way for a farm. On DC01: `fslogix.admx` to `C:\Windows\PolicyDefinitions`, `fslogix.adml` to `C:\Windows\PolicyDefinitions\en-US`. New OU `RDS Servers`, move RDS01 and RDS02 into it, then a new GPO linked there called `FSLogix`: Computer Configuration, Policies, Administrative Templates, FSLogix.

| Policy setting | Value | Folder | Registry value it writes |
|---|---|---|---|
| Enabled | Enabled | Profile Containers | Enabled = 1 |
| VHD Locations | \\\\fs01.ad.itera.ca\\Profiles$ | Profile Containers | VHDLocations |
| Size In MBs | 30000 | Profile Containers | SizeInMBs = 30000 |
| Delete Local Profile When VHD Should Apply | Enabled | Profile Containers | DeleteLocalProfileWhenVHDShouldApply = 1 |
| Profile Type | Normal | Profile Containers | ProfileType = 0 |
| Volume Type (VHD or VHDX) | VHDX | Container and Directory Naming | VolumeType = VHDX |
| Flip Flop Profile Directory Name | Enabled | Container and Directory Naming | FlipFlopProfileDirectoryName = 1 |

Exclude the admins in the same GPO, because the delete setting would otherwise remove their local profiles: Computer Configuration, Preferences, Control Panel Settings, Local Users and Groups, New, Local Group. Action **Update** (Replace would recreate the group with a new SID). Group name **FSLogix Profile Exclude List**, member `ITERA\Domain Admins`, Add to this group.

On a session host the values then show up under `HKLM\SOFTWARE\FSLogix\Profiles`. You can change them there too, but group policy is preferred, because every host gets the same settings and a change in the registry is overwritten at the next refresh. Restart the session hosts so they pick the policy up. Check what FSLogix did in `C:\ProgramData\FSLogix\Logs\Profile`.

ProfileType: 0 normal, 1 this machine always read-write, 2 this machine always read-only, 3 tries read-write and falls back to read-only. Don't mix 0 with other values.

## Public name
Public DNS: an A record for your name to your public IP (on Cloudflare, set Proxy status to DNS only). Inside: on DC01, DNS Manager, a new primary zone named the full public name, with a blank-name host record to RDSGW01 (10.37.0.86). Don't create a zone for the whole domain.

## Certificate (RDSGW01)
1. win-acme `settings_default.json`: set `PrivateKeyExportable` to `true` everywhere (Replace All) before the first request.
2. `Scripts\ImportRDSFull.ps1` calls WMIC on one line, and Microsoft deprecated WMIC. Search the file for `wmic` and replace that line with this one. Check the file again if you update win-acme, because an update can put the old line back:
```
Get-CimInstance -Namespace root\cimv2\TerminalServices -ClassName Win32_TSGeneralSetting | Set-CimInstance -Property @{SSLCertificateSHA1Hash=$CertInStore.Thumbprint}
```
3. PowerShell in the win-acme folder:
```
.\wacs.exe --source manual --host rds.itera.ca --certificatestore My --installation iis,script --installationsiteid 1 --script "Scripts\ImportRDSFull.ps1" --scriptparameters "{CertThumbprint} rdscb01.ad.itera.ca" --accepttos --emailaddress you@yourdomain.com
```
Keep your broker's full name in the script parameters. HTTP validation needs port 80 reaching RDSGW01 at every renewal.

**The DNS method, with no open ports.** With DNS validation Let's Encrypt checks a record in your public DNS instead of connecting to your server, so it also works for a farm that stays inside your network or sits behind CGNAT. Use a win-acme plugin for your DNS provider, such as Cloudflare, so renewals can create the record by themselves. The Manual plugin shows you the record to create by hand, and win-acme's docs say it's not supported for unattended renewal, so it's only good for a first try.
- DNS validation and the provider list: https://www.win-acme.com/reference/plugins/validation/dns/
- Cloudflare plugin: https://www.win-acme.com/reference/plugins/validation/dns/cloudflare
- Manual plugin: https://www.win-acme.com/reference/plugins/validation/dns/manual

## Gateway policies (RD Gateway Manager on RDSGW01)
- Connection policy `RDG_CAP_AllUsers`: Requirements, remove Domain Users, add `RDS Users`.
- Resource policy `RDG_AllDomainComputers` (I renamed it RDG_RDSFarm): User Groups `RDS Users`; Network Resource, new RD Gateway-managed group `RDS Farm` with rdscb01.ad.itera.ca, rds01.ad.itera.ca, rds02.ad.itera.ca.
- Disable `RDG_RDConnectionBrokers`.
- Later changes to the deployment's gateway settings can overwrite these, so re-check after.

## Firewall (group policy "RDS Firewall", linked to the RDS Servers OU)
Computer Configuration, Policies, Windows Settings, Security Settings, Windows Defender Firewall with Advanced Security, Inbound Rules. Two block rules:
| Rule | Protocol and ports | Action | Remote IP address (Scope) |
|---|---|---|---|
| Block RDP except gateway TCP | TCP 3389, 3387, 3392 | Block | 0.0.0.0 to 10.37.0.85, 10.37.0.87 to 255.255.255.255, :: to ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff |
| Block RDP except gateway UDP | UDP 3389 | Block | The same three ranges |

- Leave out your gateway's address, so use your own ranges.
- Why block rules: a policy's allow rule merges with the host's local rules. I tested an allow rule scoped to the gateway, with the same rule ID as the local one, and the client still connected because the local rule stayed open. A block rule beats an allow rule.
- Ports: TCP 443 in to the gateway. TCP 80 in for HTTP validation only. TCP 3389 gateway to hosts only.

## Taking a server out of rotation
Collections, Desktops, Host Servers, right-click the server, Do not allow new connections. A user with a session open there can still reconnect to it, so sign out before you test.

## If it doesn't work
- **Roles Not Configured after win-acme:** check that the WMIC line in `ImportRDSFull.ps1` was replaced (see Certificate), then run it again.
- **A win-acme update put the WMIC line back:** renewals can stop reaching the Remote Desktop roles. Replace the line again, or install WMIC from Optional features.
- **Edge says ERR_SSL_KEY_USAGE_INCOMPATIBLE on RD Web before the real certificate:** expected. Test with Remote Desktop Connection straight to the server.
- **Red XSLT banner on RD Web:** the page is old and browsers are removing XSLT. It worked at recording, but RD Web may stop displaying in a future browser.
- **Prompt shows the last user:** More choices, Use a different account. The connection remembers credentials until you sign out of the client or restart it.
- **Port forwarding fails:** check for CGNAT (WAN IP differs from your public IP, or starts 100.64 to 100.127), or the provider blocking 80 or 443.
- **Domain join fails:** DNS on that server must be the domain controller.
- **Sign-in fails:** use `ITERA\username`, and untick must-change-password.
- **User stays on the same server:** they closed the window instead of signing out.
- **Profile problems:** read the FSLogix profile log first.
- The title bar always shows the broker's name. Check the server under Settings, System, About.

## Before you use this for real
This was a lab. MFA, for example Entra through the NPS extension. Patch before exposing anything. A single broker, gateway and file server are single points of failure here, so plan high availability for them. Back up the file server that holds the profiles. No evaluation in production, an evaluation domain controller can't be converted. Apps go on every host, with `change user /install`. Antivirus needs the FSLogix exclusions. A failed profile disk gives a temporary profile by default; PreventLoginWithFailure and PreventLoginWithTempProfile block that. Server 2025 hosts need 2025 RDS CALs on a Server 2025 license server, and the farm stops accepting users after 120 days without one.

## Undo
Remove the router forwards. Delete the cloud test VM. Delete the seven VMs and the external switch. Remove the public DNS record if you added one for this.

## Links
- Server 2025 evaluation: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025
- Windows 11 Enterprise evaluation: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise
- Install Hyper-V: https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/install-hyper-v
- FSLogix download: https://aka.ms/fslogix_download
- FSLogix configuration settings: https://learn.microsoft.com/en-us/fslogix/reference-configuration-settings
- FSLogix storage permissions: https://learn.microsoft.com/en-us/fslogix/how-to-configure-storage-permissions
- FSLogix prerequisites and antivirus exclusions: https://learn.microsoft.com/en-us/fslogix/overview-prerequisites
- RDS CALs: https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/rds-client-access-license
- RDS MFA planning: https://learn.microsoft.com/en-in/windows-server/remote/remote-desktop-services/rds-plan-mfa
- win-acme: https://www.win-acme.com
- win-acme RDS guide: https://www.win-acme.com/manual/advanced-use/examples/rds
