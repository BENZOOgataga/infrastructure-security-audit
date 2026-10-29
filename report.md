# Infrastructure Security Audit

A two-site Proxmox environment. Public, sanitized version of a read-only infrastructure audit (phase 1: evidence collection and assessment).

Addresses, hostnames, account names, key identifiers, network ranges, port mappings and deployment-specific details are removed or generalized. Finding statuses, severities and counts match the private report this version was derived from. The audit collected no secret values, and none appear here.

**Assessment date: September 2026.** This is a point-in-time snapshot of phase 1. Remediation is a separate phase, tracked privately; its progress is not reflected in the findings below, and no finding here should be read as a description of the environment today.

## Contents

1. [Summary](#1-summary)
2. [Scope](#2-scope)
3. [Methodology](#3-methodology)
4. [Architecture](#4-architecture)
5. [Results at a glance](#5-results-at-a-glance)
6. [Systemic findings](#6-systemic-findings)
7. [Segmentation and exposure](#7-segmentation-and-exposure)
8. [Identity and trust concentration](#8-identity-and-trust-concentration)
9. [Patch management](#9-patch-management)
10. [Credentials and secrets](#10-credentials-and-secrets)
11. [Containers and virtualization](#11-containers-and-virtualization)
12. [Monitoring and detection](#12-monitoring-and-detection)
13. [Backups](#13-backups)
14. [Controls that held up](#14-controls-that-held-up)
15. [Remediation roadmap](#15-remediation-roadmap)
16. [Notes on method](#16-notes-on-method)
17. [Conclusion](#17-conclusion)
18. [Appendix: finding register](#18-appendix-finding-register)

## 1. Summary

The audit covered a self-hosted, two-site virtualization environment built on Proxmox VE: 33 running workloads, 16 at a remotely housed site (Site A, which also hosts the edge firewall) and 17 at an on-premises site (Site B). It was read-only throughout. Every workload's actual state was checked; documentation was treated as a set of claims to verify.

Evidence collection is complete. Every running workload has sufficient coverage, and none is left blocked or unknown. What remains open is recorded explicitly as unverified items, candidates, provisional findings and policy questions for the owner.

Results:

- 17 distinct findings: 8 confirmed (7 MEDIUM, 1 LOW) and 9 provisional. Provisional findings carry a severity range because at least one input is unresolved; two of them reach HIGH at the top of their range.
- No confirmed CRITICAL or HIGH finding.
- 65 open finding candidates and 125 observations.

The main concern is how far a few identities and templates reach across the fleet, more than any single exploitable service:

- one monitoring identity is authorized for administrative SSH on 25 hosts;
- VM templates were cloned with their SSH host keys, so hosts share identities;
- root login with password authentication is enabled on 30 of the 33 workloads;
- firewall policy between and within the sites is broad rather than service-specific.

Controls that were checked and held up include:

- a key-only, non-root SSH baseline on the management server;
- SSH disabled on the edge firewall;
- weekly automated root-password rotation on managed hosts;
- effective host-level filtering on several hosts;
- application login gates that refused unauthenticated requests;
- read-only monitoring APIs;
- unprivileged LXC containers;
- an IDS rule set updated the same day it was checked;
- SIEM agents reporting as active.

Open uncertainties:

- No service was verified as reachable from the Internet during this phase. NAT and firewall policy were verified for some services, but end-to-end reachability was not tested.
- Backup coverage was not verified.
- Detection coverage was not verified: which logs each agent collects, whether the IDS receives the mirrored traffic, and whether IDS alerts reach the SIEM.

## 2. Scope

In scope: all 33 running workloads on both sites. That includes:

- the edge firewall (configuration evidence only);
- management, monitoring, logging, SIEM and IDS systems;
- reverse proxies, mail, application hosts and datastores;
- game-server nodes;
- a VPN gateway;
- several LXC containers.

Out of scope:

- The Proxmox hypervisors themselves. The auditor never logged into them; hypervisor facts came from the owner.
- Stopped workloads (14, inventoried only).
- Penetration testing, load testing, restore testing, and any change to production.

## 3. Methodology

### Phases

Phase 0 built a canonical inventory by reconciling independent sources: hypervisor listings, the configuration-management inventory, documentation, and live reachability. Documentation counted as claims, not as ground truth.

Phase 1 audited each running workload with versioned procedures (Linux, Docker, LXC, network, datastore) that the owner had approved in advance. The work ran in approved steps, with a checkpoint review after each one.

### Rules of engagement

- Read-only by default: no configuration changes, no package-index refreshes, no restarts, no remediation.
- Nothing was written to production hosts. Audit scripts were streamed over SSH, and all records were kept outside the audited systems.
- No secret values were collected. Records hold metadata, key names and "configured yes/no" only.
- Network probes were limited to one known source, one known destination and one justified port. The default was one attempt, and never more than three. No scanning, enumeration or brute force.
- Evidence came from two sources:
  - commands the auditor ran over approved access paths;
  - commands or screenshots supplied by the owner. For four workloads, all evidence came this way, and the auditor did not re-run those commands independently.
- Any unexpected behaviour, instability, reaction from a security control or possible secret exposure stopped the affected check.
- A procedure command that did not behave as written stopped that check until the procedure was reviewed and corrected.

### Evidence states

| State | Meaning |
|---|---|
| VERIFIED | Observed directly or confirmed independently |
| NOT_VERIFIED | Not established. This is not a negative claim |
| UNKNOWN | No trustworthy evidence either way |
| BLOCKED | Could not be obtained through an approved path |

### Result classes

| Class | Meaning |
|---|---|
| Confirmed finding | Condition verified, all severity inputs resolved |
| Provisional finding | Condition verified, at least one severity input unresolved; reported as a range |
| Candidate | Plausible issue waiting on evidence or an owner policy decision |
| Observation | Informational or hardening note |

### Severity

- Severity = Impact × Likelihood, where Likelihood = min(Exploitability, Exposure weight).
- Exposure weights:

  | Exposure | Weight |
  |---|---|
  | Internet | HIGH |
  | Cross-boundary | MEDIUM |
  | Same-zone | MEDIUM |
  | Local | LOW |

- An unresolved dimension makes the finding provisional. Its range runs from the lowest to the highest plausible value of that dimension.
- Evidence confidence is reported separately and never changes severity.
- A systemic root cause is counted once, with its affected scope, rather than once per host.

### Exposure states

The audit kept these states separate and did not infer one from another:

- local listener;
- same-zone reachability;
- cross-boundary reachability;
- reverse-proxy path;
- NAT path;
- WAN policy allow;
- Internet reachability.

## 4. Architecture

```
                  Internet
                     |
             [ Edge Firewall ]    destination NAT for selected services;
                     |            admin VPN and site-to-site VPN termination
     +---------------+----------------+
     |  Site A      (Proxmox VE)      |
     |  management and automation     |
     |  reverse proxy                 |
     |  application hosts (Docker)    |
     |  datastores                    |
     |  monitoring                    |
     +---------------+----------------+
                     |
           site-to-site VPN tunnel
                     |
     +---------------+----------------+
     |  Site B      (Proxmox VE)      |
     |  reverse proxy                 |
     |  application hosts (Docker)    |
     |  datastores                    |
     |  monitoring                    |
     |  LXC containers (some Docker)  |
     +--------------------------------+

  Security tooling (site placement not shown):
  SIEM with agents on most hosts, passive network IDS
```

- Administration goes through a VPN and a configuration-management server. A few workloads outside the managed inventory are administered directly.
- Monitoring consists of:
  - an active-check monitoring server;
  - time-series metrics servers;
  - a log store;
  - a SIEM with agents across the fleet;
  - a passive network IDS.
- Docker is used widely, in some cases inside LXC containers.

## 5. Results at a glance

| Metric | Value |
|---|---|
| Workloads covered | 33 of 33 running (Site A 16, Site B 17) |
| Confirmed findings | 8 (7 MEDIUM, 1 LOW) |
| Provisional findings | 9 (1 MEDIUM–HIGH, 1 LOW–HIGH, 7 LOW–MEDIUM) |
| Distinct findings | 17 |
| Open candidates | 65 |
| Observations | 125 |
| Findings by site | Site A only 6, Site B only 5, both sites 6 |
| Services verified reachable from the Internet | None (not tested end to end) |

## 6. Systemic findings

Findings that are not tied to one host keep their original IDs. Host-specific findings use neutral public IDs (`PUB-F-...`).

### F-GLOBAL-SSH-001: shared SSH host identities (confirmed, MEDIUM)

- Condition: VM templates were cloned without regenerating SSH host keys. Three shared identity groups cover 27 hosts, and one more host matches a group on one key type only.
- Impact: whoever holds one group's host private key can impersonate every host in that group to any client that trusts the key. That weakens host authentication on administrative SSH paths. Exposure is LOCAL, because key possession is required.
- Remediation principle: generate a unique identity per host, and update the known-hosts stores of every automation client in the same change. Schedule this after the trust-path fixes so automation doesn't break.

### F-GLOBAL-SSH-002: root login with password authentication (provisional, MEDIUM–HIGH)

- Condition: the effective SSH configuration permits root login and password authentication on 30 of 33 workloads. The other three either follow a key-only, non-root baseline or have SSH disabled.
- Impact: HIGH, because a successful login gives root.
- Exploitability: unresolved. Password strength was not assessed. Per-source SSH penalties or fail2ban are active on many hosts but missing on hosts running older OpenSSH releases.
- Mitigation: a few hosts accept SSH only from administrative networks. This lowers exposure, but not the rated range.
- Remediation principle: move to a key-only, non-root administrative baseline in stages, with verified console recovery for each host. The remotely housed site goes last.

### F-GLOBAL-PATCH-001: no kernel update path on one template (confirmed, MEDIUM)

- Condition: hosts built from one VM template lack the distribution's kernel metapackage, so routine upgrades never move their kernel forward. This affects 11 hosts; one of them also has automatic upgrades disabled.
- Contrast: hosts from the other main template carry the metapackage and are current.
- Remediation principle: restore the update path in the template and on each host, then schedule the reboots.

### PUB-F-TRUST-001: monitoring identity with broad administrative SSH trust (confirmed, MEDIUM)

- Condition: a monitoring service's SSH key is authorized for root or an administrative account on 25 hosts, with no source or command restrictions.
  - Key presence, SSH daemon acceptance and network path were assessed as separate dimensions. On one host the daemon's root-login policy blocks the key, and on another a host firewall excludes the monitoring source.
  - Network reachability from the monitoring host was not exercised, so no host is described as reachable.
- Impact: the key reaches a large part of the fleet with administrative privileges, including security-sensitive systems. That creates a broad potential blast radius. This describes configuration, not a demonstrated access path.
- Severity: MEDIUM (Impact HIGH, Exploitability MEDIUM, Exposure LOCAL). The host count did not raise it.
- Remediation principle:
  - map what the monitoring system actually needs SSH for;
  - give it a dedicated non-root account on each target;
  - restrict the key by source and command;
  - protect the private key;
  - remove authorizations it doesn't use.

### F-GLOBAL-DKR-001: containers with direct access to the host Docker daemon (confirmed, MEDIUM)

- Condition: the host Docker socket is mounted into 8 containers on 5 hosts, with no authorization plugin or socket proxy in front of it. The containers are a deployment platform, orchestration helpers in a mail suite, and container-metrics agents.
- Impact: compromising one of those containers gives root-equivalent control of its host. Exposure is LOCAL.
- Contrast: several other container deployments mount no socket at all.
- Remediation principle: put daemon access behind a least-privilege, read-only socket proxy, or remove it where it isn't functionally needed. Some of these uses are required by the vendor, so they need an explicit owner decision.

### F-GLOBAL-CRED-001: credential files readable by any local user (confirmed, MEDIUM per row)

- Condition: six files that hold credentials were readable by any local account. This was determined from the file mode together with the parent-directory chain. The files are application environment files, a service configuration file, a database authentication file and an admin-credential configuration file. Their values were not collected.
- Scope: 6 rows, each rated between LOW and MEDIUM.
- Impact: a local foothold on those hosts would cross the application's trust boundary.
- Remediation principle: set ownership and modes so only the consuming service can read each file.

### F-GLOBAL-TLS-001: TLS private keys readable by local users (provisional, LOW–MEDIUM)

- Condition: on two reverse proxies, private-key files in one certificate directory were readable by any local user. The standard protected key stores on the same hosts were correctly restricted.
- Severity: Exposure LOCAL, Exploitability MEDIUM. Only Impact is unresolved, because the audit did not map keys to the services they protect.
- Remediation principle: restrict the key files to the service that terminates TLS.

## 7. Segmentation and exposure

No service was verified as reachable from the Internet during this phase. NOT_VERIFIED means the state was not established. It does not mean "not exposed".

- **Edge publication:**
  - the edge firewall publishes selected services through destination NAT;
  - matching WAN allow rules were verified for some of them;
  - a NAT rule and an allow rule together were not treated as proof of Internet reachability.
- **Cross-site policy:**
  - the site-to-site firewall policy permits broad cross-site access instead of service-specific segmentation, and the admin VPN policy is similarly broad;
  - traffic across the sites was verified only for the specific paths tested;
  - whether this breadth is intended is a decision for the owner.
- **Hypervisor firewall:**
  - hypervisor-level firewall enforcement was not consistently active across the environment. Where the firewall service ran with enforcement disabled, per-interface firewall flags on guests were set but had no effect;
  - where a policy exists, its effect on individual VMs was not verified;
  - the intended policy is an owner decision.
- **Host firewalls:**
  - most hosts have none;
  - four hosts filter at host level, with policies of varying strictness;
  - one of them, a default-deny setup that admits its web service only from its reverse proxy, was verified effective.
- **IPv6:** IPv6 exposure controls were not fully verified at one site. This is the largest single open question about exposure.

## 8. Identity and trust concentration

| Relationship | What the evidence supports |
|---|---|
| Monitoring key authorized on 25 hosts | Verified configuration; network path not exercised |
| Configuration-management automation with administrative access to managed hosts | Verified, by design |
| Shared SSH host identities across template clones | Verified; exploiting it requires stealing a host key |
| Root and password SSH on 30 workloads | Verified configuration; password strength unknown |
| Static privileged passwords outside the rotation | Verified static; whether they match across hosts was not verified |
| Containers with host Docker daemon access | Verified; root-equivalent on the host |
| VPN CA private key on the VPN gateway | Verified |
| Broad cross-site and admin-VPN firewall policy | Verified policy; traffic verified only on tested paths |

None of these relationships was exercised. The report does not claim a demonstrated compromise path. The concern is architectural: a few identities and templates span most hosts on both sites, and segmentation does not contain them.

## 9. Patch management

- **Template dependence:** hosts from one template have no kernel update path (F-GLOBAL-PATCH-001); hosts from the other main template are current.
- **Standalone workloads without automatic updates:** four provisional findings (PUB-F-PATCH-002 to -005) and further candidates. Symptoms:
  - package metadata many months out of date;
  - one host with 21 security updates pending at its last metadata refresh;
  - one host missing advisory-level kernel and OpenSSL fixes.
- **Edge firewall:** one release behind a version that includes a security fix (candidate). Its integrity health check passed.
- **Container images:** several deployments run images many months old or use floating `latest` tags.
- **Limits of this phase:**
  - per-CVE applicability was checked at advisory level for one host only;
  - the report makes no CVE-specific exploitability claims;
  - there is no documented patch-latency target, so release lag alone did not promote any candidate.

## 10. Credentials and secrets

- **Findings:**
  - F-GLOBAL-CRED-001: six locally readable credential files;
  - F-GLOBAL-TLS-001: TLS keys;
  - PUB-F-PKI-001: the VPN CA private key sits online on the gateway.
- **Names versus values:** several configuration files contain keys with secret-like names whose values were deliberately not collected. They stay candidates unless a literal secret is established.
- **Other gaps:**
  - static privileged passwords outside the rotation;
  - root passwords on unmanaged hosts outside the rotation;
  - SIEM administrator accounts that still carry installer-default names (credential state not verified).
- **Handled well:**
  - weekly automated root rotation on managed hosts;
  - several application secrets readable by root only;
  - standard TLS key stores protected.

## 11. Containers and virtualization

- **Docker socket exposure:** 8 containers on 5 hosts (F-GLOBAL-DKR-001).
- **LXC:**
  - every audited container is unprivileged, confirmed from the hypervisor side;
  - nesting is enabled where Docker runs inside LXC;
  - no explicit host mounts, device passthrough or raw configuration overrides were observed;
  - the hypervisor-side AppArmor label reads "unconfined", so restrictive AppArmor confinement is not established;
  - resource limits are enforced.
- **Weakest case:** one containerized monitoring workload had no effective network filtering at any of the three layers checked: the guest firewall (policy accept), hypervisor firewall enforcement (off), and the per-interface firewall flag (not set).
- **Best case:** one application host, with:
  - 13 containers, none privileged, all under Docker's default AppArmor profile, none with the socket mounted;
  - datastores internal to the Compose network;
  - one proxy published, on the LAN only and reachable only from the reverse proxy;
  - an API that refused unauthenticated requests.

## 12. Monitoring and detection

Having a tool deployed is not the same as having detection verified.

| Capability | State |
|---|---|
| SIEM agents enrolled and active | VERIFIED for every listed agent |
| SIEM events recently received | VERIFIED at manager level; partial per agent |
| Relevant log sources configured per agent | NOT_VERIFIED |
| SIEM retention | NOT_VERIFIED (no index-retention policy observed) |
| SIEM agent enrollment | PUB-F-SIEM-001: no shared enrollment password |
| IDS mode | Passive (AF_PACKET), VERIFIED; inline prevention not evidenced |
| IDS scope | One network segment only |
| IDS rules | Updated the same day, VERIFIED; update mechanism NOT_VERIFIED |
| Mirrored traffic reaching the IDS | NOT_VERIFIED |
| Alerts from the IDS reaching the SIEM | NOT_VERIFIED |

The monitoring identity also reaches security-sensitive systems; the network path was not exercised. If the monitoring host were compromised, detection itself could be affected.

## 13. Backups

Backup coverage is NOT_VERIFIED across the fleet. For most workloads, this phase supports no statement either way, and missing evidence was not treated as missing backups.

Known facts:

- one hypervisor backup job failed;
- several workloads had no hypervisor backup job visible;
- one database had no host-level backup job visible;
- the edge firewall's configuration backup was not evidenced;
- the SIEM had no index-retention policy visible.

Restore testing was out of scope.

## 14. Controls that held up

- **SSH:**
  - the management server enforces key-only, non-root SSH;
  - one more host follows the same baseline;
  - the edge firewall has SSH disabled;
  - per-source SSH penalties or fail2ban are active on many hosts.
- **Host filtering:** four hosts filter at host level, and one default-deny setup was verified effective.
- **Login gates:** several administrative interfaces and APIs answered unauthenticated requests with 401 or a redirect to a login page, and anonymous object-storage listing returned AccessDenied. Each was checked with a single request and no credentials.
- **Minimal exposure:** datastores are kept inside their Compose networks, and one host publishes only a proxy, on the LAN.
- **Monitoring APIs:** on the metrics servers checked, the administrative, lifecycle and write endpoints are disabled.
- **Credential rotation:** weekly automated root rotation on managed hosts, verified.
- **Currency:**
  - hosts from one template generation are fully current;
  - the IDS rules were updated the same day;
  - the edge firewall passed its integrity check.
- **Virtualization:** every LXC container is unprivileged, with no explicit host mounts or device passthrough.
- **SIEM:** agents are active and ingestion is running.

## 15. Remediation roadmap

Nothing in this roadmap has been carried out. Items are ordered by, in turn:

1. risk reduction;
2. blast radius;
3. dependencies;
4. operational safety;
5. reversibility.

Every change is meant to go through an explicit change protocol (proposal, impact, failure conditions, rollback, approval), with extra care at the remotely housed site.

| # | Item | Reason for its position | Main dependency | Reversibility |
|---|---|---|---|---|
| 1 | Restrict the monitoring identity (PUB-F-TRUST-001) | Largest potential blast radius | Inventory of what the monitoring system uses SSH for | Per-host key edits; reversible |
| 2 | Test Internet exposure from an external vantage point, over IPv4 and IPv6 | Settles many provisional ranges | Owner approval; an external vantage point | Read-only |
| 3 | Owner decisions on cross-site segmentation and hypervisor firewall intent | Unblocks the segmentation work | None | No system change |
| 4 | Roll out key-only, non-root SSH in stages (F-GLOBAL-SSH-002), remote site last | Removes the HIGH upper bound across the fleet | Verified key access and console recovery | Console revert |
| 5 | Restrict credential and TLS key files (F-GLOBAL-CRED-001, F-GLOBAL-TLS-001) | Cheap, low-risk reduction | Confirm which service account reads each file | Easy to revert |
| 6 | Narrow edge and VPN policy according to item 3 | Cross-site blast radius | Item 3; confirmed firewall config backup | Reversible; needs a maintenance window |
| 7 | Restore update mechanisms, apply pending updates, update the edge firewall | Closes kernel and library gaps | Backup evidence (item 9); reboot windows | Snapshot-based |
| 8 | Put Docker daemon access behind a proxy; add authentication to the same-zone cache service | Root-equivalent container paths | Application requirements | Config revert |
| 9 | Gather backup coverage evidence, then plan restores | Needed before items 6 to 8 | Owner evidence | Read-only |
| 10 | Verify and fix detection coverage: agent log sources, IDS mirror and alert path, SIEM enrollment password | Incidents need to be visible | SIEM admin access | Reversible |
| 11 | Regenerate unique SSH host identities (F-GLOBAL-SSH-001) | Has to follow items 1 and 4 | Coordinated known-hosts updates | Coordinated; scheduled last |
| 12 | Move the VPN CA key offline, add authentication to monitoring APIs, restore repository signature checks | Remaining contained items | Certificate-issuance workflow | Reversible |

## 16. Notes on method

- **Reconcile inventories first.** Doing this before any assessment surfaced migrated, stopped and undocumented workloads that the documentation had wrong.
- **Keep exposure states separate.** A listener, a NAT rule and a firewall allow rule are three different facts, and none of them proves Internet reachability.
- **Count root causes once.** A template defect or an over-trusted identity is one finding with a scope, not 25 findings.
- **Separate a key's authorization from its usability.** Whether a key is present, whether the daemon accepts it, and whether a network path exists were recorded as three separate dimensions. That kept the trust finding from being overstated.
- **Report ranges, not guesses.** When a severity input was unresolved, the finding got a range and a named blocking uncertainty instead of a single invented severity.
- **Record unverified as unverified.** Unknown backup coverage and unknown IPv6 policy are reported as open questions, not as passes.
- **Label evidence provenance.** Where the auditor could not reach a system directly, the report says the evidence came from the owner.
- **Stop on procedure defects.** A command that did not behave as written stopped its check. The defect was logged and the procedure corrected before the work resumed.

## 17. Conclusion

Phase 1 is complete. All 33 running workloads have sufficient coverage, and none is left blocked or unknown.

The environment has solid controls in specific places:

- the management server baseline;
- host filtering on the project-management host;
- automated password rotation;
- SSH disabled on the edge firewall;
- unprivileged containers.

It also has a small number of trust-concentration patterns that span both sites:

- a broadly authorized monitoring identity;
- shared host identities;
- root password SSH;
- broad segmentation.

Those patterns are the main architectural concern, and the roadmap starts there. Internet reachability, backup coverage, detection effectiveness and vulnerability applicability remain open and are documented as such.

## 18. Appendix: finding register

| # | ID | Status | Severity | Summary |
|---|---|---|---|---|
| 1 | F-GLOBAL-SSH-001 | Confirmed | MEDIUM | Shared SSH host identities from cloned templates (27 hosts in 3 groups) |
| 2 | F-GLOBAL-PATCH-001 | Confirmed | MEDIUM | No kernel update path on one template (11 hosts) |
| 3 | PUB-F-TRUST-001 | Confirmed | MEDIUM | Monitoring identity authorized for administrative SSH on 25 hosts without restrictions |
| 4 | F-GLOBAL-DKR-001 | Confirmed | MEDIUM | Host Docker daemon access from 8 containers on 5 hosts |
| 5 | F-GLOBAL-CRED-001 | Confirmed | MEDIUM (per row) | Credential files readable by any local user (6 rows) |
| 6 | PUB-F-PKI-001 | Confirmed | MEDIUM | VPN gateway stores the CA private key that authorizes client identities |
| 7 | PUB-F-DATA-001 | Confirmed | MEDIUM | In-memory data store accepts unauthenticated connections from the same trust zone |
| 8 | PUB-F-MON-001 | Confirmed | LOW | Metrics API reachable without authentication; write and admin endpoints disabled |
| 9 | F-GLOBAL-SSH-002 | Provisional | MEDIUM–HIGH | Root login with password authentication (30 of 33 workloads) |
| 10 | F-GLOBAL-TLS-001 | Provisional | LOW–MEDIUM | TLS private keys readable by local users on two reverse proxies |
| 11 | PUB-F-MON-002 | Provisional | LOW–MEDIUM | Log-store API without authentication |
| 12 | PUB-F-SUPPLY-001 | Provisional | LOW–MEDIUM | Package manager set to trust a third-party repository even when authentication checks fail |
| 13 | PUB-F-PATCH-002 | Provisional | LOW–MEDIUM | No active update mechanism; advisory-level kernel and OpenSSL fixes missing |
| 14 | PUB-F-SIEM-001 | Provisional | LOW–MEDIUM | SIEM agent enrollment accepts agents without a shared enrollment password and listens on all IPv4 interfaces |
| 15 | PUB-F-PATCH-003 | Provisional | LOW–MEDIUM | Container without automatic security updates; package metadata long out of date |
| 16 | PUB-F-PATCH-004 | Provisional | LOW–MEDIUM | No automatic update mechanism; 21 security updates pending at last refresh |
| 17 | PUB-F-PATCH-005 | Provisional | LOW–HIGH | No working update path on a small utility container |

The 65 open candidates fall into these themes:

- network exposure: services listening on all interfaces without host filtering;
- administrative interfaces that are published or unfiltered;
- segmentation;
- hypervisor firewall enforcement;
- patch currency;
- credential hygiene, where file contents were not verified;
- authentication boundaries that were not tested;
- DNS resolver scoping;
- lifecycle and operational debt;
- privileged or dormant automation tooling.

The 125 observations are hardening and informational notes. Recurring themes:

- local journal retention that is too short;
- gaps in remote logging;
- permissive defaults such as forwarding and X11;
- engines and tools that are installed but unused;
- positive notes, such as the edge firewall's clean integrity check.
