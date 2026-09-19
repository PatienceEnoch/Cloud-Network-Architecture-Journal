Cloud Architect Journal of Patience Enoch

A living technical atlas of distributed systems, routing, cloud design, and the structural thinking of a modern cloud/network engineer.

1. Core Concepts

Fundamentals every architect must internalize:

What a topology truly is (structure, not diagram)

Loopbacks & identity

Control plane vs data plane

Hop-by-hop state

Flooding, LSAs, LSDB

What makes a “clean state”

Paths vs relationships

Convergence, churn, suppression

**Folder: /core/**

2. Routing & Distributed Systems

Deep dives into:

IS-IS & why hyperscalers prefer it

OSPF

Tier 1 vs Tier 2 ISPs

Anycast vs Unicast

Why Anycast failover is unpredictable

Stateful TCP fragility at the edge

QUIC + stateless APIs

CAP theorem applied to networks

Eventual consistency at the edge

Segment Routing (SR-MPLS, SRv6)

[BGP Failover in a Three-AS Mini Internet](core/bgp-failover-and-timers.md)

**Folder: /routing/**

3. Network Analytics

Includes:

Flaps, churn, suppression

How to read traceroutes like an SRE

Interpreting convergence events

Latency vs jitter vs loss

LSDB change interpretation

Telemetry as truth

Patterns of network failure

**Folder: /analytics/**

4. Cloud Architecture

VPC design patterns

Subnetting strategies

NAT patterns

Multi-tier microservice blueprints

Region, AZ, and failure-domain reasoning

Edge vs core architectures

Hyperscaler design philosophies

[Network Flight Recorder: From Local Diagnostic Tool to Cloud Operations Architecture](cloud/network-flight-recorder-architecture.md)

**Folder: /cloud/**

5. Design Principles

Reusable architectural heuristics:

Simplicity vs flexibility

Minimize state

Blast radius isolation

Designing for churn

Graceful degradation

Idempotent systems

**Folder: /design/**

6. Philosophies

“gold nuggets":

SPF doesn’t find paths — it finds relationships

Loopbacks are identity, not interfaces

Control planes hold truth, not routes

“Networks are living systems”

“Consistency is a choice; availability is a promise”

“Topology is story, not geometry”

**Folder: /philosophy/**

7. Diagrams

Visual maps for:

Cloud architectures

Routing flows

TOR relay design

VPC blueprints

Distributed system diagrams

**Folder: /diagrams/**

8. Notes & References

Reference materials like:

PDFs

External reading summaries

Quotes

Long-form notes

**Folder: /notes/**

9. Labs

My hands-on architect portfolio:

Tor Relay Engineering

AWS VPC Blueprint

Terraform VPC rebuild

Splunk SOC investigations

GNS3 routing labs

Anycast experiments

CAP theorem experiments

**Folder: /labs/**

**Commitment**

This repo represents my transformation into a cloud architect. Every file is an intentional neural pathway I am building. Everything here has a purpose.

**Discalimer** I use ChatGPT as a thinking partner to help me clarify concepts, structure my notes, and accelerate my learning process.
All insights, interpretations, and architectural framing are my own.
AI supports my productivity — it does not replace my original thought.
