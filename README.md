# Infrastructure Security Audit

A read-only security audit of the self-hosted infrastructure I operate across two Proxmox VE sites. The work covered inventory reconciliation, host and container configuration, SSH trust, firewall and VPN policy, patching, credentials, monitoring and detection, and backups. It ends with a prioritized remediation plan.

This is phase 1: evidence collection and assessment. Remediation is a separate phase. The full report is in [report.md](report.md).

## Scale

- 2 sites: one remotely housed behind an edge firewall, one on premises, joined by a site-to-site VPN
- 33 running workloads: 16 and 17 per site, a mix of virtual machines and LXC containers
- 14 stopped workloads inventoried and excluded from the active audit

## Technologies

- **Virtualization:** Proxmox VE (VMs and unprivileged LXC containers, hypervisor firewall)
- **Operating systems and containers:** Linux (Debian-based), Docker and Docker Compose, including Docker inside LXC
- **Network:** an edge firewall with destination NAT, an admin VPN and a site-to-site VPN; reverse proxies at both sites; a host firewall on some hosts
- **Services:** mail, application and game servers, SQL and in-memory datastores, a DNS resolver, and a VPN gateway
- **Operations:** configuration-management automation with scheduled root-password rotation, and an active-check monitoring server
- **Observability and detection:** time-series metrics, a log store, a SIEM with agents across the fleet, and a passive network IDS

## What I assessed

- **Inventory:** reconciled hypervisor listings, the automation inventory, documentation and live reachability. The documentation no longer matched the environment; the reconciliation turned up migrated, stopped and undocumented workloads.
- **SSH:** effective daemon configuration (`sshd -T`, including per-source `Match` evaluation), authorized keys on every account, host-key identity across hosts, and brute-force throttling.
- **Privilege and credentials:** sudo rules, password state and rotation, and the effective readability of credential and TLS key files (file mode plus the full parent-directory chain; values never read).
- **Containers:** privileged mode, capabilities, AppArmor, Docker socket mounts, published ports and image age. For LXC: privilege mapping, nesting, mounts, device passthrough and raw overrides, checked from both the guest and the hypervisor side.
- **Network:** listeners, host firewall rulesets, hypervisor firewall enforcement, edge NAT and WAN rules, and cross-site VPN policy. Each exposure state was recorded separately, from a local listener up to Internet reachability.
- **Patching:** update mechanisms, package-index age, pending security updates, kernel update paths, and advisory-level checks for one host.
- **Monitoring and detection:** SIEM agent enrollment and activity, event ingestion, retention, IDS mode, scope and rule freshness.
- **Backups:** what the evidence could and could not establish.

## How it was done

- The audit was read-only. It made no configuration changes, refreshed no package indexes, restarted nothing, and wrote nothing to the audited hosts. Audit scripts were streamed over SSH.
- Checks followed versioned procedures for Linux, Docker, LXC, network and datastores. The work ran in steps, with a review checkpoint after each one.
- Network checks were single, justified connections between a known source and a known destination. There was no scanning, no enumeration and no brute force.
- Secret values were never collected, only metadata and key names.
- Every claim carries an evidence state: VERIFIED, NOT_VERIFIED, UNKNOWN or BLOCKED. NOT_VERIFIED means the state was not established, never that something is safe.
- Severity is Impact × Likelihood, where Likelihood is the minimum of exploitability and exposure. When an input couldn't be established, the finding is reported as provisional, with a range, instead of a guessed value.
- A root cause that repeats across hosts is counted as one finding, with its scope.
- A procedure command that did not behave as written stopped its check until the procedure was corrected.

## Results

- 17 distinct findings: 8 confirmed (7 MEDIUM, 1 LOW) and 9 provisional. Two of the provisional findings reach HIGH at the top of their range.
- No confirmed CRITICAL or HIGH finding.
- 65 open candidates and 125 observations.
- No service was verified as reachable from the Internet. End-to-end reachability was not tested, so exposure is reported as unverified rather than absent.

The main findings are systemic:

- A monitoring service's SSH key is authorized for administrative access on 25 hosts, with no source or command restrictions.
- VM templates were cloned without regenerating SSH host keys, so 27 hosts share host identities across three groups.
- Root login with password authentication is enabled on 30 of the 33 workloads.
- 8 containers on 5 hosts mount the host Docker socket, which is root-equivalent on the host.
- Six credential files, and TLS private keys on both reverse proxies, are readable by any local user.
- One VM template has no kernel update path (11 hosts), and several standalone workloads have no working update mechanism.
- Cross-site firewall policy is broad, hypervisor firewall enforcement is inconsistent, and most hosts have no host firewall.
- Detection coverage is only partly verified: SIEM agents are active, but per-agent log sources, IDS traffic visibility and the IDS-to-SIEM alert path are not established.

The report also records what held up: a key-only, non-root SSH baseline on the management server, working host firewalls on several hosts, verified weekly password rotation, application login gates that refused unauthenticated requests, and unprivileged LXC containers throughout.

## What was sanitized

This public version removes or generalizes anything that would help locate or target the live systems:

- IP addresses (IPv4 and IPv6), MAC addresses, network ranges and VPN addressing;
- hostnames, usernames and SSH key fingerprints;
- public endpoints, port mappings and firewall source addresses;
- exact trust-chain membership and the host membership of each systemic finding.

Hosts are referred to by role, for example "Management Server". Finding IDs that contained hostnames were replaced with neutral `PUB-F-` IDs. Finding statuses, severities and counts match the private report.

## Licence

This report is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0). You may share and adapt it, including commercially, with attribution.
