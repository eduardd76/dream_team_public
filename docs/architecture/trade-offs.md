# Trade-offs we made deliberately

The decisions an architect will ask about within 10 minutes. We answer them up front so the conversation doesn't get stuck on the same five questions.

## 1. Human-in-the-loop on every change (no auto-apply)

**What we chose:** every approved fix is applied by `change-executor` *after* a human approves in the UI.

**What we gave up:** the marketing-sexy "fully autonomous remediation" claim. Pure-autonomy demos look better in a webinar.

**Why we chose it:**

- The regulated buyer (PCI, SOX, FINRA) won't approve auto-apply in production.
- The reputational cost of one wrong auto-apply outweighs the throughput cost of human approval.
- The platform's value prop is *decision-grade evidence* on the card, not *removing the human*.

**When we'd reconsider:** for non-production environments (staging, lab), `auto_apply_on_allow` is a planned policy flag. Production stays human-gated.

## 2. Rule-based gates, not LLM-judged decisions

**What we chose:** CRO uses JSON-defined policies. BRF uses a tenant catalog. Pass 2 uses regex. None of the gates ask an LLM "is this safe?"

**What we gave up:** the ability to handle policies that aren't easily expressible in rules (e.g., "respect the cultural calendar of the country where the device lives").

**Why we chose it:**

- Auditors trust JSON they can read. They don't trust LLM weights.
- Determinism: the same incident, the same hour → the same verdict. Always.
- A rule that fires wrong is *debuggable* in 5 minutes. An LLM that fires wrong takes weeks.

**When we'd reconsider:** if a customer presents a policy class that genuinely needs natural-language reasoning, we'd add an LLM-as-judge step **with rule-based gates around its output** — a recommendation, not a decision.

## 3. Topology-as-code (single fabric.yaml) instead of NetBox-as-source-of-truth

**What we chose:** `fabric.yaml` at the repo root is the single source of truth. NetBox + MCP collectors flow data IN to populate the runtime KG; they don't define the platform's model.

**What we gave up:** the ability to point at an existing NetBox and "just go" — onboarding requires a YAML rendering step.

**Why we chose it:**

- Version-controlled topology = version-controlled tests. The KG seed, the containerlab generator, the NetBox import all derive from one file.
- Customers who don't have NetBox (or who run it as read-only documentation) aren't blocked.
- The YAML is small enough (~100 lines for a 3-device fabric) that a customer can audit it.

**When we'd reconsider:** at scale (50+ devices) the manual YAML approach becomes tedious. Phase B-2+ adds a `netbox-to-fabric-yaml` generator (one-directional, NetBox source → fabric.yaml output, committed to the customer's repo).

## 4. Containerlab (Layer 2 DT) + vrnetlab (Layer 3 DT), not pure simulation

**What we chose:** real vendor OS images running in containers. SR Linux containers ship straight from Nokia. cEOS-lab ships straight from Arista. NX-OS is vrnetlab-packaged from the customer's existing image.

**What we gave up:** speed (10-15s per validate vs. milliseconds for pure simulation) and image-licensing complexity (cEOS license is per-host).

**Why we chose it:**

- Pure simulation misses vendor quirks. NX-OS reorders ACLs in subtle ways that simulation doesn't model. Real images catch this.
- The DT's verdict has to be defensible. "We ran your fix on a real Nokia SR Linux container; here's the BGP-EVPN session state pre/post" beats "we simulated the topology and the math said OK."
- Cost is acceptable (~$0.006/incident at Tier 2).

**When we'd reconsider:** for high-volume incident classes (link flap noise, transient interface errors) we could route to GraphSimulator only. Pass 3 already does this when Pass 2 fails.

## 5. Deterministic regex triage at the Team Leader (no LLM there)

**What we chose:** Team Leader uses a regex rule table to triage syslogs → specialist + tier + fault_type. No LLM call in the triage path.

**What we gave up:** flexibility on novel syslog formats. Adding a new vendor syslog pattern means adding a regex rule.

**Why we chose it:**

- Routing must be *bit-exact reproducible*. Determinism contract — `TEST_PLAN.md` §6.
- Triage is on the latency-critical path. An LLM call adds 2-5s for no decision value.
- Regex rules are auditable in seconds. LLM routing decisions are not.

**When we'd reconsider:** the test catalog already includes ~64 incident patterns. Above ~500 patterns, the regex table becomes unwieldy and we'd add an LLM-assisted *fallback* — never a primary.

## 6. Self-hosted LLMs for domain reasoning (not OpenAI / Anthropic)

**What we chose:** Qwen2.5-7B, CodeLlama-7B, Mistral-7B running on `g5.xlarge` boxes in our own AWS account. OpenAI is a fallback if local endpoints are down.

**What we gave up:** simplicity (we manage 4 vLLM endpoints), latency (16-22s per LLM call vs. 3-5s on OpenAI), and pure-quality scores (gpt-4o-mini wins composite eval).

**Why we chose it:**

- Data residency: customer data (syslogs, device states) doesn't leave their VPC.
- Customer trust: "your data never leaves your AWS account" beats "we send it to OpenAI."
- Cost predictability: known $/hr, not unpredictable token usage.
- Compliance defensibility: easier to defend a self-hosted model to an auditor than a third-party API.

**When we'd reconsider:** for customers who have explicit OpenAI / Azure-OpenAI agreements (GxP-class), we'll happily flip to their preferred endpoint. The LLM is an injectable dependency.

## 7. Phase B (vendor abstraction) is built progressively, not big-bang

**What we chose:** the vendor adapter seam (`VendorAdapter` ABC + registry + factory) shipped in Phase A. Each adapter migrates progressively: NX-OS first, then Arista EOS, then SR Linux. Tests with golden-parity gates between vendors.

**What we gave up:** a clean "vendor-agnostic from day 1" architecture story.

**Why we chose it:**

- Big-bang vendor rewrites are how you get 6-month outages on critical paths.
- Each vendor migration is independently testable + reversible (`MCP_USE_ADAPTER` env flag, default off).
- Customers can pilot with a single vendor while the others remain on the legacy path.

**See:** [vendor-abstraction.md](vendor-abstraction.md) for the design.

---

## What we did NOT trade off

- **No half-finished safety gates.** The 7 passes are either implemented or absent; nothing is "stub for the demo."
- **No undocumented escape hatches.** Every env var that changes behavior is in this docs site or in the source.
- **No silent fallback to "everything is fine."** If a gate fails to evaluate, the card shows `skip_reason` with a named reason. The reviewer always knows what they're approving.
