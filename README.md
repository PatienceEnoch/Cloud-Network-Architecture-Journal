<p align="center">
  <img src="https://raw.githubusercontent.com/PatienceEnoch/PatienceEnoch/main/assets/profile-header.svg" alt="Patience Enoch — Cloud and Network Engineering" width="100%">
</p>

# Cloud Network Architecture Journal

This is where I keep the ideas that stick after the lab is over.

I use this journal to connect routing, cloud architecture, distributed systems, failure behavior, and observability back to things I can test. Some entries are concept notes. Others come directly from projects where I built a topology, broke a path, watched the control plane react, or changed an architecture after seeing where it failed.

> **A backup path is architecture. Recovery time is behavior.**

## Current threads

### Routing and convergence

- [BGP Failover in a Three-AS Mini Internet](core/bgp-failover-and-timers.md)
- [BGP Path Selection](core/bgp-path-selection.md)
- [IGP vs BGP](core/igp-vs-bgp.md)
- [SPF and Convergence](core/spf-and-convergence.md)
- [Churn and Suppression](core/churn-and-suppression.md)

These notes connect directly to my [Mini Internet](https://github.com/PatienceEnoch/mini-internet) lab, where I use FRRouting and Docker to watch BGP react to controlled link failures.

### Observability and failure

- [Network Flight Recorder: From Local Diagnostic Tool to Cloud Operations Architecture](cloud/network-flight-recorder-architecture.md)
- [Guarded Remediation: Verification Before Trust](cloud/guarded-remediation-and-rollback.md)
- [Failure Domains and Local-First Observability](cloud/failure-domains-and-local-first-observability.md)

These grew out of [Network Flight Recorder](https://github.com/PatienceEnoch/network-flight-recorder), my local-first troubleshooting and incident-evidence project.

## What lives here

| Area | What I am studying |
| --- | --- |
| **Core networking** | topology, loopbacks, control vs data plane, adjacency, clean state |
| **Routing** | BGP, OSPF, IS-IS, path selection, convergence, churn, anycast |
| **Distributed systems** | state, consistency, CAP, failure domains, graceful degradation |
| **Cloud architecture** | VPC design, NAT, segmentation, AWS operations, hybrid connectivity |
| **Observability** | telemetry, evidence, incident lifecycle, network failure analysis |
| **Design principles** | blast radius, least privilege, idempotence, verification, rollback |

## A few ideas I keep coming back to

- Loopbacks are identity, not just interfaces.
- A clean routing table does not prove a clean data path.
- Redundancy and convergence are different problems.
- Metrics tell me that something happened; evidence helps explain why.
- Observability should not disappear with the system it is observing.
- A successful configuration change is not proof of successful recovery.

## Where this is heading

My next major networking work is hybrid cloud connectivity:

~~~text
Local routing lab
      |
   IPsec VPN
      |
     AWS
      |
Transit Gateway
   /       \
Dev VPC   Prod VPC
      |
Flow Logs / CloudWatch
      |
Network Flight Recorder
~~~

The goal is to keep building one connected body of work: routing fundamentals, hybrid connectivity, cloud segmentation, infrastructure as code, observability, and failure testing.

## About the notes

This is a working engineering journal, not a polished textbook. I keep older ideas when they are useful, correct them when my understanding changes, and prefer tested observations over impressive-sounding claims.

I sometimes use AI tools as a thinking partner for organization or explanation. The labs, measurements, configurations, and conclusions I publish here are things I check against my own work and source material.

---

[Main GitHub profile](https://github.com/PatienceEnoch) · [Portfolio](https://github.com/PatienceEnoch/Hopkins_portfolio)
