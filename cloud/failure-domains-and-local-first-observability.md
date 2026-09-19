# Failure Domains and Local-First Observability

One of the strongest architectural lessons from [Network Flight Recorder](https://github.com/PatienceEnoch/network-flight-recorder) came from a simple question:

> What happens if the network path used by the monitoring system is the thing that fails?

That question changed how I think about observability.

A cloud dashboard can be useful, but if every diagnostic capability depends on reaching the cloud, then a major connectivity failure can remove both the service and the visibility needed to explain the outage.

Network Flight Recorder was designed to avoid that dependency.

---

## The failure-domain problem

Imagine a monitoring architecture like this:

~~~text
Linux host
   |
Network failure occurs
   |
Cloud monitoring agent
   |
Internet
   |
Cloud API
   |
Dashboard
~~~

If the local route, gateway, DNS path, or upstream connection fails, the system may lose its ability to report the very failure it is supposed to diagnose.

That creates a shared failure domain.

The application, the network path, and the observability path all depend on the same connectivity.

If that path disappears, visibility disappears with it.

---

## The local-first design

Network Flight Recorder separates local diagnosis from centralized operations.

~~~text
                LOCAL
                  |
        Network state collection
                  |
            Snapshot history
                  |
            Change analysis
                  |
         Root-cause correlation
                  |
           Incident evidence
                  |
             Diagnosis
                  |
          -------------------
                  |
               CLOUD
                  |
            Redacted data
                  |
          Amazon S3 storage
                  |
        CloudWatch metrics/logs
                  |
              Dashboard
~~~

The important part is that everything above the boundary can still operate without AWS.

The recorder can inspect:

- routes
- default gateway
- interfaces
- IP addresses
- DNS resolvers
- DNS lookup behavior
- reachability
- latency
- packet loss
- MTU
- failed services

It can compare state, correlate symptoms, and produce a diagnosis locally.

AWS adds durability and centralized visibility when connectivity exists.

It is not required for the first layer of troubleshooting.

---

## Why this is a failure-domain decision

At first, I thought of "local vs cloud" mainly as a software architecture choice.

It is really a resilience choice.

If the diagnostic engine depended on AWS, then these two systems would share a dependency:

~~~text
Production network
        |
Internet path
        |
Diagnostic system
~~~

A failure in the shared path could affect both.

Instead, the architecture looks more like:

~~~text
             +----------------------+
             |  Local diagnostics   |
             |  routes / DNS / NICs |
             |  evidence / analysis |
             +----------+-----------+
                        |
                 connectivity available
                        |
             +----------v-----------+
             |   AWS operations     |
             | S3 / CloudWatch      |
             +----------------------+
~~~

The local diagnostic capability sits outside the cloud-connectivity failure domain.

That means the system can still answer:

> What changed?

even when it cannot yet answer:

> Did the cloud dashboard receive the update?

---

## Observability should degrade gracefully

A resilient observability system does not need every component to work perfectly during an outage.

It needs the most important capabilities to remain available.

For Network Flight Recorder, the priority is:

1. preserve local evidence
2. diagnose the failure
3. track the incident
4. upload or centralize evidence when connectivity becomes available

That is graceful degradation.

If AWS becomes unreachable, the architecture loses centralized visibility temporarily, but it does not lose the underlying diagnostic evidence.

The system becomes less convenient, not blind.

---

## Detailed evidence vs centralized visibility

This design also clarified that local evidence and cloud telemetry serve different purposes.

### Local evidence answers

- What route disappeared?
- Did the gateway change?
- Which interface changed state?
- Did DNS stop resolving?
- Did packet loss increase?
- What was different from the known-good state?

### Cloud telemetry answers

- Is the recorder healthy?
- Are failures being detected?
- How many incidents are occurring?
- What recent incident summaries exist?
- Is evidence being preserved centrally?

The cloud layer provides operational awareness.

The local layer provides forensic detail.

Both matter, but they do not need to have the same failure dependencies.

---

## Store-and-forward thinking

A useful way to think about the architecture is:

~~~text
Observe locally
      |
Preserve locally
      |
Analyze locally
      |
Connectivity available?
   /           \
 yes            no
  |              |
upload         retain
  |              |
centralize     retry later
~~~

This is similar to store-and-forward behavior in other distributed systems.

The system protects the evidence first and treats remote delivery as a secondary operation.

That is especially important during intermittent or partial connectivity failures.

---

## The dependency inversion

A less resilient design would make local diagnosis depend on cloud services.

Network Flight Recorder effectively reverses that relationship:

~~~text
Cloud operations depend on local evidence

not

Local diagnosis depends on cloud operations
~~~

That is a subtle but important architectural distinction.

The cloud layer consumes the outputs of diagnosis.

It does not own the ability to diagnose.

---

## Connection to hybrid networking

This lesson will matter even more when I build hybrid connectivity between a local lab and AWS.

Suppose an IPsec tunnel fails.

If all monitoring lives on the AWS side of that tunnel, I may only know that the on-premises side disappeared.

A local observer could still preserve:

- local routes
- tunnel interface state
- BGP neighbor state
- reachability to the tunnel endpoint
- default route behavior
- DNS state
- timestamps surrounding the failure

Then, once connectivity is restored, those records can be centralized.

The same design principle applies:

> Put critical evidence collection close enough to the failure that the failure cannot erase the evidence.

---

## What this changed in my thinking

Before this project, I thought of monitoring mostly in terms of dashboards, alerts, and cloud services.

Now I think about observability in terms of **failure independence**.

For every monitoring component, I want to ask:

- What does this component depend on?
- Could the monitored failure also disable the monitor?
- What evidence survives if connectivity disappears?
- Where is state stored?
- Can the system continue operating in a degraded mode?
- Can evidence be forwarded later?

That is a much more useful architectural lens than simply asking which monitoring service to use.

---

## Design principle

The lesson I want to carry into future cloud and network systems is:

> **Do not place all of your observability inside the same failure domain as the system you need to observe.**

A dashboard is useful.

Evidence that survives the outage is more important.

---

## Final takeaway

Network Flight Recorder taught me that observability is not only about collecting more telemetry.

It is about designing the telemetry path so that failure does not erase the story of what happened.

The architecture should preserve enough local truth to reconstruct the incident even when centralized systems are temporarily unreachable.
