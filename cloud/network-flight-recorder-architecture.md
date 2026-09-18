# Network Flight Recorder: From Local Diagnostic Tool to Cloud Operations Architecture

This entry documents the architecture lessons I learned while building my [Network Flight Recorder](https://github.com/PatienceEnoch/network-flight-recorder).

The project began as a local Linux troubleshooting tool: capture a known-good network state, compare it with a degraded state, identify what changed, and produce evidence that supports a likely root cause.

As the project grew, it became a useful architecture exercise in observability, resilience, security, infrastructure as code, cloud operations, and controlled remediation.

---

## The Core Architecture Question

The project is centered on a simple operational question:

> What changed immediately before the network failed?

A useful answer requires more than an alert. The system needs a baseline, current state, evidence, correlation, and a way to preserve what happened.

The architecture evolved into:

```text
Healthy network
      |
Baseline snapshot
      |
      +-------------------+
                          |
Current network           |
      |                   |
Current snapshot ---------+
      |
Change analysis
      |
Evidence + correlation
      |
Incident report
      |
Redaction
      |
AWS evidence and operations layer
      |
CloudWatch visibility
```

The remediation path is deliberately separate:

```text
Diagnosis
   |
Remediation proposal
   |
Safety guardrails
   |
Human approval
   |
Execution
   |
Verification
```

---

## Architectural Decision 1: Keep Diagnosis Local-First

One of the most important design choices was keeping the core diagnostic workflow independent of AWS.

The recorder captures routes, interfaces, addresses, DNS state, reachability, latency, packet loss, MTU information, and failed services directly from the Linux host.

This means the system can still collect evidence and diagnose a network problem even when cloud connectivity is unavailable.

### Why this matters

A monitoring system should not become useless simply because the network it is monitoring has failed.

If every part of the diagnostic workflow depended on reaching an external cloud service, a major network outage could remove both the application and the visibility needed to troubleshoot it.

The cloud layer therefore enhances the system rather than becoming a hard dependency for basic diagnosis.

### Principle

> Observability should not depend entirely on the system being observed.

This is a failure-domain decision as much as a software decision.

---

## Architectural Decision 2: Separate Diagnostic and Operations Layers

The project separates responsibilities.

### Local diagnostic layer

Handles:

- Network-state collection
- Baseline snapshots
- Change detection
- Root-cause correlation
- Incident reports
- Continuous monitoring
- Recovery verification

### AWS operations layer

Handles:

- Evidence storage
- Incident history
- Operational metrics
- Incident summaries
- Dashboard visibility

This reduces coupling.

The Linux diagnostic engine can evolve independently from the cloud operations layer, while AWS provides centralized storage and visibility.

---

## Architectural Decision 3: Infrastructure as Code

The AWS infrastructure is defined with Terraform rather than created manually.

The current AWS layer includes:

- Amazon S3
- Amazon CloudWatch
- IAM-related controls
- CloudWatch Logs
- CloudWatch dashboard resources

Terraform makes the environment reproducible and reviewable.

Instead of relying on memory or console configuration, the desired infrastructure state is stored with the application code.

### Why this matters

Infrastructure as code improves:

- Repeatability
- Change visibility
- Version control
- Reviewability
- Recovery and recreation
- Automation

It also allows infrastructure configuration to be validated in CI before deployment.

---

## Architectural Decision 4: Treat Diagnostic Evidence as Sensitive Data

Network snapshots can reveal information such as:

- Internal IP addresses
- Hostnames
- Probe destinations
- Network topology clues
- Resolver configuration

Because of that, evidence storage should not be treated like ordinary public application data.

The project uses deterministic pseudonymization so sensitive values can be replaced before evidence is shared or uploaded.

The same value maps consistently across snapshots, which allows comparisons without exposing the original value.

### AWS evidence protections

The S3 evidence layer includes:

- Public-access blocking
- S3 versioning
- SSE-S3 encryption at rest
- A bucket policy denying insecure transport
- Redacted evidence uploads

The resulting model is:

```text
Network evidence
      |
Redaction
      |
TLS-protected transfer
      |
Private S3 bucket
      |
Encrypted + versioned evidence
```

### Principle

> Operational evidence can be valuable and sensitive at the same time.

---

## Architectural Decision 5: Observability Needs Both Metrics and Context

A single metric can tell me that something happened.

It usually cannot explain why.

The CloudWatch layer therefore uses more than one type of operational signal.

Examples include:

- `SnapshotSuccess`
- `ReachabilityFailure`
- Incident summaries in CloudWatch Logs
- A dashboard for recent operational state

The local recorder preserves detailed evidence while CloudWatch provides higher-level operational visibility.

This creates two complementary views:

```text
CloudWatch
    -> Is the system healthy?
    -> Are failures occurring?
    -> What incidents are recent?

Network Flight Recorder evidence
    -> What changed?
    -> What observations support the diagnosis?
    -> What happened before and after recovery?
```

### Principle

> Metrics indicate behavior. Evidence explains behavior.

---

## Architectural Decision 6: Test Failures Deliberately

The project includes a Docker-based failure-injection lab.

The lab can:

1. Create a disposable network environment.
2. Capture a healthy baseline.
3. Remove the container's default route.
4. Capture the degraded state.
5. Diagnose the routing failure.
6. Generate an incident report.
7. Restore network configuration.
8. Verify recovery.

This gives the architecture a reproducible failure scenario instead of relying only on happy-path testing.

The failure is isolated to the disposable container, which keeps the test safe.

### Principle

> Resilience cannot be demonstrated only by testing success.

---

## Architectural Decision 7: Automate Validation Before Automating Change

GitHub Actions validates the project on pushes and pull requests.

The CI pipeline includes:

### Application

- Python tests with pytest
- Ruff linting

### Security

- Python dependency auditing

### Infrastructure

- `terraform fmt -check`
- `terraform init -backend=false`
- `terraform validate`

This creates an automated quality gate around both application code and infrastructure configuration.

The important architectural distinction is that validation is highly automated while remediation remains controlled.

---

## Architectural Decision 8: Human-Controlled Remediation

Detecting a problem and changing a network are not equivalent risk levels.

Network Flight Recorder therefore treats remediation conservatively.

The recovery layer uses:

- Explicitly allowed remediation actions
- Approval-required remediation plans
- Guardrails around supported actions
- Post-remediation verification
- Before-and-after reporting

The architecture intentionally follows:

```text
Observe
   |
Diagnose
   |
Propose
   |
Approve
   |
Act
   |
Verify
```

rather than:

```text
Detect -> immediately change production
```

### Principle

> Automation should increase confidence before it increases authority.

---

## Incident Lifecycle Thinking

The project eventually moved beyond one-time snapshot comparison into incident lifecycle tracking.

The recorder can preserve:

- Failure evidence
- Open incident state
- Recovery evidence
- Incident start time
- Recovery time
- Outage duration
- Final lifecycle summary

This changes the mental model from:

```text
Something failed.
```

to:

```text
Healthy
   |
Transition
   |
Failure detected
   |
Incident opened
   |
Evidence preserved
   |
Recovery observed
   |
Incident closed
   |
Lifecycle summarized
```

That is much closer to how operational systems are reasoned about in production environments.

---

## Architecture Lessons I Am Taking Forward

The most important lessons from this project are:

- Keep critical diagnostics available during connectivity failures.
- Separate local system responsibilities from centralized cloud operations.
- Treat evidence as sensitive data.
- Use infrastructure as code so architecture is reproducible.
- Use metrics for visibility and detailed evidence for explanation.
- Test failure states intentionally.
- Automate validation aggressively.
- Give remediation more safeguards than observation.
- Design around the full incident lifecycle, not only detection.
- Reduce dependencies inside the same failure domain whenever possible.

---

## Final Takeaway

Network Flight Recorder started as a networking troubleshooting project, but building it forced me to think beyond individual commands or AWS services.

The larger lesson is that architecture is about relationships between components, dependencies, failure domains, evidence, security boundaries, and operational behavior.

The most important design question was not simply:

> Which cloud service should I use?

It was:

> If part of this system fails, what information and capabilities must still remain available?

That question will influence how I design future cloud and network systems.
