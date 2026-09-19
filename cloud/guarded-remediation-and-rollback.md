# Guarded Remediation: Verification Before Trust

This entry documents one of the most important lessons I learned while expanding [Network Flight Recorder](https://github.com/PatienceEnoch/network-flight-recorder): detecting a problem is much safer than changing a network.

The project originally focused on observation:

~~~text
Capture state
   |
Compare against baseline
   |
Find meaningful changes
   |
Correlate symptoms
   |
Produce evidence-backed diagnosis
~~~

Once I added remediation, the architecture had to change.

A diagnostic tool can be wrong without immediately changing the network. A remediation system can turn a wrong conclusion into a new outage.

That means automation needs stronger controls as its authority increases.

---

## A successful command is not proof of recovery

A simple remediation workflow might be:

~~~text
Problem detected
      |
Run repair command
      |
Command exits successfully
      |
Declare success
~~~

That is not enough.

A route can be added but point to the wrong gateway. An interface can come up while the path is still unusable. A service can restart without restoring end-to-end connectivity.

Network Flight Recorder therefore validates the result rather than trusting command execution alone.

> A successful command is not proof of successful recovery.

---

## The guarded recovery model

The recovery flow is intentionally conservative:

~~~text
Observe
   |
Diagnose
   |
Propose
   |
Approve
   |
Execute
   |
Verify
   |
Success? ---- yes ----> Preserve evidence
   |
   no
   |
Rollback
   |
Verify restored state
~~~

Each stage answers a different question.

**Observe:** What is the network doing?

**Diagnose:** What evidence supports a likely root cause?

**Propose:** What narrowly scoped change could address it?

**Approve:** Is the action allowed and explicitly authorized?

**Execute:** Apply only the approved action.

**Verify:** Did the network actually return to the expected state?

**Rollback:** If verification fails, restore the pre-remediation state instead of leaving the system in an uncertain configuration.

---

## Why an allowlist matters

The remediation layer does not accept arbitrary shell commands as repairs.

Supported actions are represented by explicit action IDs and checked against an allowlist.

An unrestricted model would effectively be:

~~~text
Diagnosis -> generate any command -> execute it
~~~

The guarded model is:

~~~text
Diagnosis
   |
Choose from known supported actions
   |
Validate action
   |
Require approval
   |
Execute defined behavior
~~~

That reduces the blast radius of both software bugs and incorrect diagnoses.

> The safest automation is not the automation that can do everything. It is the automation that can do exactly what it is trusted to do.

---

## Before-and-after evidence

The recovery workflow preserves evidence from both sides of the change.

In the isolated routing-failure lab, the record can show:

~~~text
Before remediation
Default route: missing

Known-good state
Default route: present

Remediation
Restore network configuration

After remediation
Default route: restored

Verification
PASSED
~~~

This is more useful than a log line saying "fixed."

It creates an auditable chain:

~~~text
Observed failure
      |
Approved action
      |
State change
      |
Observed recovery
~~~

---

## Rollback changes the safety model

The most important addition was not remediation itself. It was rollback after failed verification.

Without rollback:

~~~text
Execute change
      |
Verification fails
      |
Unknown or degraded state
~~~

With rollback:

~~~text
Execute change
      |
Verification fails
      |
Restore pre-change state
      |
Verify rollback
~~~

Rollback does not make every change safe, but it gives the system a defined response when the expected outcome does not occur.

---

## Why this is not a general self-healing network

Network Flight Recorder's current remediation capability is intentionally narrow and restricted to the isolated lab.

A production self-healing system would need to account for concurrent failures, stale state, dependency chains, repeated remediation loops, authorization boundaries, partial recovery, production rollback safety, and escalation when automation cannot recover the system.

The lab demonstrates the control pattern without pretending that one recovery script equals production-grade autonomous operations.

---

## The broader engineering lesson

As systems gain automation, they also gain authority.

I now think of that progression as:

~~~text
Observe
   |
Explain
   |
Recommend
   |
Change
   |
Autonomously recover
~~~

Every step increases potential impact.

Observation may only need read permissions. Recommendation needs explainability. Changes need authorization and scope limits. Autonomous recovery needs verification, rollback, rate limits, persistent state, and stronger safety guarantees.

> Automation should gain safeguards at least as quickly as it gains authority.

---

## Connection to network engineering

This applies directly to Terraform, Ansible, Python automation, SDN controllers, and orchestration platforms.

A deployment should not stop at:

~~~text
configuration applied successfully
~~~

The more useful question is:

~~~text
Did the intended network behavior actually occur?
~~~

That may require checking route presence, next-hop selection, BGP adjacency state, end-to-end reachability, DNS resolution, latency, packet loss, application health, or security-policy behavior.

Configuration state and operational state are related, but they are not identical.

---

## What I am taking forward

For future cloud and network automation, I want the workflow to look like:

~~~text
Pre-change state
      |
Controlled change
      |
Post-change validation
      |
Expected behavior?
   /        \
 yes        no
  |          |
record     rollback or escalate
~~~

Verification should be part of the deployment itself rather than an optional troubleshooting step afterward.

---

## Final takeaway

The most useful lesson from guarded remediation was:

> **Do not trust that a change worked. Measure the network after the change.**

If the evidence says recovery failed, the system should have a defined path back instead of continuing from an unknown state.
