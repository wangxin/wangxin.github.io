---
layout: post
title:  "Notes on scaling sonic-mgmt: seven design debts in the testbed"
date:   2026-06-11 10:00:00 +0800
categories: sonic, sonic-mgmt, test-infrastructure, testbed, architecture
---

I have spent the better part of the last five years working inside `sonic-mgmt`, the test repository for the SONiC network operating system. As of 2026 I am its top contributor by commit count, which mostly means I have spent a lot of time staring at the same set of corners and wondering why they hurt.

This post is not a complaint and not a bug list. It is a senior-engineer reflection on the *testbed* design — the layer that provisions VMs, wires up topology, and gives tests something to run against. The framework above it and the test scripts on top of it have their own stories; I will save those for another day.

I want to write this down for two reasons. First, several of the people most likely to act on it no longer share a hallway with me, and these observations are too easy to lose. Second, I suspect the patterns here generalize. If you work on test infrastructure for any large, fast-growing, multi-vendor OSS networking project, some of this will look familiar.

## The root cause, stated once

If you read the seven items below as independent issues, you will reach for seven independent fixes. That is the wrong frame. **Six of the seven trace to the same root cause:** the testbed was designed when *"a few people running a few testbeds"* was the realistic use case, and it has been scaled by accretion ever since. Specifically:

* Configuration is treated as **code** — files written by hand, edited in place, copied between testbeds — instead of **data**, declared once and rendered into whatever artifact a tool needs.
* Tooling was chosen for the original workload (Ansible playbooks, `pytest-ansible`) and never re-evaluated when the workload grew an order of magnitude.
* Multiple representations of the same information exist side by side because each new use case added a new file rather than extending the existing one.

The unifying fix is a **schema-first redesign** of testbed inventory, with a Python-native runtime sitting on top. Most of the seven concerns below then fall out as natural sub-problems of that one initiative, instead of seven separate fights.

Now, the seven items.

## 1. Inventory as code, not data

Today, neighbor VMs and their IP addresses are hand-written into a huge Ansible vars file (`ansible/veos` in the public repo; the internal variant is north of 25,000 lines). Adding a testbed means hand-editing this file, picking IPs by eye, and hoping nobody else picks the same ones at the same time.

Why it hurts: every new testbed is a manual operation; merges between branches collide on the same lines; mass edits (renumbering a subnet, retiring a VM pool) require ad-hoc scripts; IP collisions show up as weeks of intermittent test failures.

What I would do: replace the file with a declarative spec — "this testbed runs a T0 topology on this server, with these DUTs" — and let an allocator pick neighbor VM names and IPs from a pool, recording the leases back to disk. The topology definition itself already says how many neighbors are needed and what role each plays; the spec should not repeat that. The right primitive is a typed inventory schema (Pydantic, JSON Schema) so violations fail at load time, not at deploy time.

A spec might look like this:

```yaml
testbed: tb-east-7
server: vmhost-east-7
topology: t0            # implies neighbor count, roles, and wiring
duts:                   # a testbed may host one or more DUTs
  - str-msn2700-01
neighbors:
  type: veos            # allocator picks names and IPs from the pool
```

The constraint is backwards compatibility. Realistically the old format and the new one have to coexist for at least one release cycle, with a one-way converter so existing testbeds keep working while new ones move over.

## 2. Bridge sprawl: consolidate OVS via OpenFlow + VLAN

Each neighbor today is wired up through roughly four OVS bridges. The bridge count grows linearly with topology size; understanding any given testbed means walking a small forest of `ovs-vsctl show` output.

Why it hurts: bridge sprawl makes the topology hard to reason about, hard to inspect, and creates a lot of host-side kernel state to keep healthy.

What I would do: collapse to one bridge per testbed (or per fabric layer), and use OpenFlow flow tables keyed on VLAN to steer traffic between DUT, neighbor VMs, and PTF. This is essentially the Neutron ML2/OVS pattern, well-trodden in production for a decade.

The honest tradeoff is debuggability. `brctl show` plus `ovs-vsctl show` is very legible to a test engineer chasing "why isn't this packet getting through." `ovs-ofctl dump-flows` followed by reasoning across flow tables is not. **Only do this consolidation if you also commit to building the observability around it** — at minimum a "which flow matched this packet" tool that mortals can use. Otherwise you have traded one kind of complexity for a harder one.

## 3. SSH ProxyJump via the test server

The current path from the `sonic-mgmt` container to the neighbor VMs and PTF container is genuinely tricky. The container has to be IP-space-aware, the bridging has to be set up just so, and "I cannot reach PTF" is a perennial new-user failure mode.

What I would do: stop trying to put the `sonic-mgmt` container *on* the test fabric. Give the test server a host-only private subnet to the neighbors and PTF, and have the container reach them via SSH `ProxyJump` (or `ProxyCommand`) through the test server. This is just the bastion-host pattern, used by every cloud SRE team in the world. Every SSH client supports it, no new code required.

The constraint: per-call SSH negotiation latency. For high-frequency operations (parallel pings, polling) the fix is `ControlMaster` plus persistent control sockets, or batch endpoints implemented server-side. Both are well-known.

## 4. Stop using Ansible playbooks as an orchestrator

Ansible is excellent at one thing — *"apply this state to a fleet of managed hosts, idempotently"* — and the rest of the testbed deployment uses it for everything else. The result is data processing in Jinja2 inside YAML inside playbooks, which is a programming model nobody actually enjoys.

Why it hurts: the playbooks are tedious to write, hard to debug, slow to evolve, and the parts that need real logic (dict filtering, conditional transforms, error handling) become unreadable.

What I would do: write the orchestration in Python. Keep Ansible underneath for the part it is good at — convergent state on managed hosts. Drive it from a Python program that owns the control flow, the data shapes, and the error handling.

This is also where the industry has been moving for years. Pulumi, Argo, Tekton, and most modern in-house orchestrators are Python- or Go-native and use Ansible (or Terraform, or kubectl) as a lower layer when they need it. For testbed orchestration specifically, Python wins hands-down.

## 5. The `pytest-ansible` ceiling

`pytest-ansible` is a thin, convenient layer that exposes maybe twenty percent of what Ansible's runtime can do. Forking and worker control, async callbacks, custom callback plugins, structured result streaming, retries and timeouts in a uniform shape — none of it is reachable through the plugin in any clean way.

The result is that the test framework keeps inventing workarounds at the layer above (homegrown parallelism, ad-hoc retries, string-parsed results) for things the orchestration engine underneath could do natively, if only it were reachable.

## 6. A Python library that wraps Ansible directly

This is the natural pairing with item 5, and the move I would actually invest in.

The wrong choice — the one I almost recommended in the first draft of this post — is **`ansible-runner`**. It is a subprocess wrapper around `ansible-playbook` designed for AWX/Tower's needs: launch a playbook as a managed job, capture its event stream, hand it to a UI. It is not designed for the workload a test framework actually has — thousands of small ad-hoc calls in tight loops, where per-call overhead matters and you want results as Python objects, not events parsed from stdout.

The right primitive is **Ansible's internal Python API**: `TaskQueueManager`, `InventoryManager`, `VariableManager`, `Play`, `DataLoader`. You build a thin wrapper that exposes typed `AnsibleHost` and `AnsibleHosts` classes (and an `AnsibleLocalhost` for completeness). On top of those, every device abstraction in `sonic-mgmt` — DUT, neighbor, PTF, fanout — becomes a regular Python object.

What that wrapper looks like in practice:

* `AnsibleHost(inventory, "vlab-01").shell("uptime")` — single-host call, returns a dict of results, raises on failure unless `task_directives={"ignore_errors": True}`.
* `AnsibleHosts(inventory, "vms_1").ping()` — group call, returns `{hostname: result, ...}` keyed by host. Forking is `TaskQueueManager`'s native concern, not the framework's.
* Dynamic dispatch via `__getattr__` so any Ansible module is callable as a method — same ergonomics as `pytest-ansible`, no plugin in between.
* A context manager for batch mode: `with host: host.shell(...); host.copy(...)` loads tasks and runs them as a single play on exit, amortizing setup cost.
* Module name validation at task-build time so typos fail fast instead of deep in the executor.
* Caller-frame logging so every log line shows the test `file:line` that originated it.

I built a working version of this on a side branch — [`ansible_hosts.py`](https://github.com/wangxin/sonic-mgmt/blob/tbng/ansible/testbed/base/ansible_hosts.py) on the `tbng` branch of my `sonic-mgmt` fork. It is roughly 800 lines, which is itself evidence the abstraction is right-sized. The code is rough in places (mutable default arguments, a context-manager bug I have already spotted, per-host inventory reparsing that should be cached), but the shape is correct and it runs.

This buys three things. First, you exit the `pytest-ansible` ceiling and inherit everything Ansible's executor can do — real forking, real timeouts, real structured results. Second, every device interaction becomes a typed Python object, which is dramatically easier to test, mock, refactor, and reason about. Third — and this matters more than people realize today — **you make the test framework AI-agent-friendly.** A typed Python API with explicit operations is something a coding agent can read, call, and combine. A plugin-mediated DSL inside YAML inside `pytest` is not. Anyone planning to use AI agents for triage, generation, or repair of network tests should put this on the critical path.

The honest cost: Ansible's internal API is not a stable public API. Red Hat steers people to `ansible-runner` precisely so they can change internals freely. If you take this path, you accept the maintenance burden: pin to a known ansible-core version range, run a CI matrix across the range you support, treat ansible-core upgrades as their own work item with their own validation. For an in-house test framework that already pins its dependencies tightly, this is an acceptable trade.

The migration is the other hard part. Every test that uses `ansible_adhoc()` eventually needs updating. But the old plugin and the new library can coexist for as long as needed; you migrate test directories one at a time, behind a feature flag, with the old path staying green throughout.

## 7. One source of truth for testbed inventory

Today, the same device shows up in both the Ansible inventory files and in `ansible/files/*.csv`. Two sources of truth, no enforcement, guaranteed drift. The downstream failure mode — "this testbed works for most people but fails intermittently for one team" — is exactly the shape you would predict.

The fix is not "pick one file format for everything." Different kinds of inventory data want different shapes, and the right answer is to use each format for what it is good at:

* **CSV for tabular, relational data** — the device list, and the physical links between devices (which port on which device connects to which port on which other device). This is fundamentally a table. CSV diffs cleanly in git, is editable in Excel by lab operators, and matches how the data already lives in people's heads.
* **YAML for structured per-device facts** — credentials, ASIC type, role, feature flags, anything that belongs in Ansible `host_vars` / `group_vars`. YAML handles nested and typed data far better than CSV ever will, and it is what Ansible natively consumes.

Both formats serve as source of truth within their own domain. What feeds Ansible is a **dynamic inventory script** — an executable that reads the CSV and YAML on every invocation and returns the inventory JSON Ansible expects. Ansible has supported this natively for years; pointing `ansible-playbook -i inventory.py` (or any executable) at it is all the integration needed. No "regenerate after edit" step, no stale committed inventory file, no drift window. The same script (or a small set of renderers around the same loader) also produces `sonic-mgmt` fixtures, the lab dashboard, IP-allocation reports — none of those are written by hand, ever.

The non-negotiable rule, regardless of format: **the source-of-truth files must be the only writable inputs.** Everything Ansible and the test framework consume is computed on demand from them. Without that enforcement, the dual-source mess comes back in a new shape within a release or two.

A useful side benefit of this split: lab operators (who care about cabling) and test engineers (who care about facts) get to work in the format that fits their job, instead of fighting a single oversized schema that tries to serve both.

## What a clean redesign would look like

If you collapse the seven items into a single picture, you get something like this:

* CSV holds devices and physical links; YAML holds per-device facts (item 7). An Ansible dynamic inventory script reads both on every invocation; no static inventory file is committed.
* An allocator service hands out VM names, IPs, and other shared resources from configured pools (item 1).
* A Python orchestrator (item 4), driving `ansible-runner` underneath via the `AnsibleHost(s)` library (item 6), provisions the testbed.
* On the host, a small number of OVS bridges with OpenFlow rules (item 2) handle traffic distribution, with first-class observability tooling.
* The `sonic-mgmt` container reaches everything via SSH `ProxyJump` through the test server (item 3) — no fabric-level network setup required for the container itself.
* Tests interact with devices through the `AnsibleHost(s)` classes (items 5 and 6), not through a `pytest` plugin.

None of this is technically hard. Every piece exists in production somewhere in the industry today.

## The part that actually is hard

What is hard is **permission and runway.** A redesign of this scope is one to two engineers, full-time, for two to three quarters. Backwards compatibility with existing testbeds — across multiple SONiC branches, multiple ASIC vendors, multiple deployment styles — is the real cost driver, not the new design itself.

There is also a governance reality. `sonic-mgmt` lives inside an OCP project with many vendor contributors; nobody clearly *owns* a cross-cutting redesign of this kind. Individual contributors can fix individual bugs. Vendors will fund features that unblock their hardware. A multi-quarter refactor of shared infrastructure, with no immediate test count to show for it, is exactly the kind of work that is structurally hard to authorize in a multi-vendor OSS context.

That is why these debts accumulate — not because the engineers inside the project cannot see them. Anyone who has spent serious time in `sonic-mgmt` can recite a version of this list from memory. The constraint is organizational, not technical.

## Closing

If you are working on test infrastructure for SONiC, or for any fast-growing networking OSS project, I would genuinely like to hear how you think about these tradeoffs. Where do you agree, where am I wrong, what has your team tried? You can reach me on [LinkedIn](https://www.linkedin.com/in/wangxinwang/).
