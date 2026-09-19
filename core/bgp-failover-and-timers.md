# BGP Failover in a Three-AS Mini Internet

## Why I built this

I wanted to move past reading BGP tables and actually watch routing change when a path disappeared.

The lab is intentionally small: three routers, three private autonomous systems, and three links forming a triangle. That gives every router a direct path to the other two while still leaving an alternate route available when one connection fails.

The goal was not just to prove that BGP could reroute traffic. I wanted to see the difference between **having a backup path** and **recovering quickly enough for that backup path to matter**.

Project: [Mini Internet](https://github.com/PatienceEnoch/mini-internet)

---

## Topology

Each router runs FRRouting inside Docker.

```text
                 ISP-A
               AS 65001
              /        \
     10.200.12.0/29   10.200.13.0/29
            /            \
       ISP-B ------------ ISP-C
      AS 65002          AS 65003
          10.200.23.0/29
```

| Router | ASN | Loopback |
|---|---:|---|
| ISP-A | 65001 | 10.200.1.1/32 |
| ISP-B | 65002 | 10.200.2.1/32 |
| ISP-C | 65003 | 10.200.3.1/32 |

The loopbacks give each router a stable identity independent of any single physical or simulated link.

Each router advertises its own loopback through BGP. Prefix lists limit the lab to the three expected /32 routes rather than allowing arbitrary prefixes.

---

## Normal routing state

With every link available, ISP-A learns two possible BGP paths to ISP-C's loopback:

```text
Direct:
A -> C
AS path: 65003

Alternate:
A -> B -> C
AS path: 65002 65003
```

With the other relevant BGP attributes equal, the direct route has the shorter AS path and becomes the selected route.

A ping sourced from A's loopback to C's loopback succeeded with:

```text
4/4 replies
0% loss
reply TTL 64
```

The important point is that the alternative route already existed before the failure. BGP did not have to invent a new path after the link went down; it had to recognize that the current path was no longer usable and select the alternative.

---

## Breaking the direct link

I disabled A's interface toward C while leaving the A-B and B-C links operational.

After BGP converged, A selected:

```text
A -> B -> C
AS path: 65002 65003
next hop: 10.200.12.3
```

Connectivity returned successfully:

```text
4/4 replies
0% loss
reply TTL 63
```

The lower TTL was consistent with the additional router in the path.

After restoring the A-C link, A returned to the direct path:

```text
AS path: 65003
```

This confirmed the basic failover behavior I expected.

But the first test hid the most interesting part.

---

## A working backup route does not mean fast failover

My first checks were performed after the routing change had already completed. That proved reachability, but it said nothing about what happened **during convergence**.

So I repeated the failure while running a continuous ping from A's loopback to C's loopback.

The results were very different from the simple before-and-after test.

| A-C BGP keepalive / hold time | Sent / received | Missing sequences | Lost replies |
|---|---:|---|---:|
| 60 / 180 seconds | 298 / 122 | 32-207 | 176 |
| 3 / 9 seconds | 48 / 41 | 23-29 | 7 |

With the original timers, replies disappeared after sequence 31 and did not return until sequence 208.

With 3-second keepalives and a 9-second hold timer, replies stopped after sequence 22 and returned at sequence 30.

Because the probes were roughly one second apart, the observed gaps suggest an interruption of nearly three minutes in the first test versus roughly 7-8 seconds in the second.

These were manual observations, not synchronized convergence benchmarks, so I would not claim those values as exact BGP convergence times. The experiment was still enough to expose the practical difference between **route redundancy** and **failure detection speed**.

---

## What was actually happening

The asymmetric failure behavior was the part that made the experiment useful.

When I brought A's interface toward C down, A knew immediately that its own interface was unavailable.

C's interface, however, remained attached to the Docker bridge. From C's point of view, the local interface itself had not necessarily failed.

That meant C could continue believing the direct BGP relationship was valid until the protocol detected that its neighbor was no longer responding.

During that period, C could still send return traffic toward the failed A-C path.

Eventually C selected:

```text
C -> B -> A
AS path: 65002 65001
```

and the ping replies resumed.

This helped me understand why looking only at one router's routing table can give an incomplete picture of a failure.

A network path is bidirectional from the application's point of view even when the routing decisions on each side are independent.

---

## Timer experiment

I shortened the BGP timers only on the A-C session.

On ISP-A:

```text
neighbor 10.200.13.3 timers 3 9
```

On ISP-C:

```text
neighbor 10.200.13.2 timers 3 9
```

After restarting the affected routers, the session established with a 3-second keepalive and 9-second hold time.

The shorter timers dramatically reduced the observed interruption in this lab.

That does **not** mean "shorter is always better."

Aggressive timers increase control-plane sensitivity. A brief CPU stall, transient packet loss, or temporary congestion can cause a session to be declared dead unnecessarily. Faster detection has a stability cost.

The engineering question is therefore not:

> What is the fastest timer I can configure?

It is:

> How quickly do I need to detect failure, and how much instability am I willing to introduce to achieve that?

---

## What this changed in my understanding

Before this lab, "BGP failover" was easy to summarize as:

```text
primary path fails -> BGP chooses backup path
```

That description is correct, but incomplete.

The experiment made several things concrete:

**Redundancy is not convergence.**  
A backup route can already exist and traffic can still be unavailable while the control plane determines that the preferred path has failed.

**Failure detection is part of availability.**  
A topology may look highly available on paper while still producing an unacceptable outage if failure detection is slow.

**Both directions matter.**  
A may know a link is down before C does. Forward and return traffic can therefore behave differently during a transition.

**Control-plane state can temporarily disagree with physical reality.**  
A failed forwarding path does not guarantee that every participating router immediately has the same understanding of the failure.

**A clean post-failure ping is not enough evidence.**  
Testing only after convergence can hide the actual user-visible interruption.

---

## How I would test this more rigorously

The next version of the experiment should record timestamps for both the failure event and every probe.

I want to measure:

- time of interface failure
- time the BGP session changes state
- time the alternative route becomes best
- first successful packet after failure
- packet loss during the transition
- behavior when B-C or A-B fails instead
- differences between timer-based detection and faster mechanisms such as BFD

That would turn the lab from a manual observation into a repeatable convergence experiment.

---

## Connection to cloud networking

The topology is local, but the design problem is the same one that appears in cloud and hybrid networks.

When I eventually connect a simulated on-premises network to AWS through an IPsec VPN and BGP, the important questions will still be:

- What path is preferred?
- What alternate path exists?
- How is failure detected?
- How quickly do both sides converge?
- What traffic is lost during the transition?
- What telemetry proves what happened?

That is why this lab matters to my cloud work. It gives me a controlled place to understand routing behavior before hiding it behind managed cloud services.

---

## Engineering takeaway

The most useful lesson from this experiment was simple:

> **A backup path is architecture. Recovery time is behavior.**

Both have to be tested.

A diagram can prove that redundancy exists. It cannot prove how the network behaves when something actually fails.
