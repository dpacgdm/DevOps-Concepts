# PHASE 9: FAANG INTERVIEW PREP — FINAL PROPOSAL

---

## OBJECTIVE

Simulate a realistic FAANG/Staff-level DevOps/SRE interview loop. 7 rounds + hire committee debrief. Target: **Strong Hire (≥85) in all rounds.**

---

## LEVEL CALIBRATION RUBRIC

Every answer you give gets tagged against this. No ambiguity about where you stand.

```
┌──────────┬─────────────────────────────────────────────────────────┐
│ Level    │ What the answer looks like                              │
├──────────┼─────────────────────────────────────────────────────────┤
│ Mid (L4) │ Correct textbook answer. Knows the "what."              │
│          │ Doesn't mention failure modes unprompted.                │
│          │ Single-solution thinking. No tradeoffs.                  │
│          │ Executes tasks. Doesn't question them.                   │
├──────────┼─────────────────────────────────────────────────────────┤
│ Sr (L5)  │ Correct + tradeoffs + failure modes.                    │
│          │ Mentions 2-3 alternatives with reasons for choice.       │
│          │ Thinks about operations (day 2).                         │
│          │ Asks clarifying questions before answering.              │
│          │ Owns the problem end-to-end.                             │
├──────────┼─────────────────────────────────────────────────────────┤
│ Staff(L6)│ Everything above + organizational impact.                │
│          │ Frames answers in business terms unprompted.              │
│          │ Identifies second-order effects.                         │
│          │ Proposes migration path, not just end state.             │
│          │ Addresses cross-team concerns.                           │
│          │ Knows when NOT to build something.                       │
├──────────┼─────────────────────────────────────────────────────────┤
│ Princ(L7)│ Everything above + industry-wide perspective.            │
│          │ Challenges the premise of the question itself.           │
│          │ Designs for 3-year evolution, not just today.            │
│          │ Can teach the concept while answering.                   │
│          │ Shapes engineering culture, not just systems.            │
└──────────┴─────────────────────────────────────────────────────────┘
```

After every answer: **"That was L5. Here's what L6 sounds like."**

---

## ANSWER FRAMEWORKS

Delivered before Round 0. These are not training wheels — every serious candidate at FAANG level uses structured communication. Not having them is a disadvantage.

### Framework 1: System Design
```
STEP 1 — Requirements (3-5 min)
  Functional: What does the system DO?
  Non-functional: Scale, latency, availability, consistency, cost
  Constraints: Existing tech, team size, compliance, timeline
  Anti-pattern: Jumping straight to "I'd use Kafka"

STEP 2 — Back-of-Envelope Math (2-3 min)
  Users → requests/sec → bandwidth → storage → compute
  This drives EVERY downstream decision
  Anti-pattern: Designing without knowing if you need 100 RPS or 100K RPS

STEP 3 — High-Level Design (10 min)
  Boxes and arrows. Data flow. Major components.
  State where data lives. State what's synchronous vs async.
  Anti-pattern: 15 boxes with no data flow arrows

STEP 4 — Deep Dive (15-20 min)
  Pick 2-3 critical components. Go deep.
  Schema, algorithms, replication, caching, failure handling.
  The interviewer may redirect you — follow their lead.
  Anti-pattern: Spreading thin across all components

STEP 5 — Failure Modes & Operations (5-10 min)
  What breaks? How do you detect it? How do you recover?
  Blast radius. Graceful degradation. Circuit breakers.
  Day 2 operations: deploys, rollbacks, scaling, debugging.
  Anti-pattern: "It's highly available" with no specifics

STEP 6 — Tradeoffs & Alternatives (3-5 min)
  What did you NOT choose and why?
  What would change at 10x scale? 100x?
  Cost vs complexity vs reliability tradeoffs.
  Anti-pattern: Presenting your design as the only option
```

### Framework 2: Troubleshooting
```
STEP 1 — Impact Assessment
  Who is affected? How many users? Which services?
  Is this a SEV1 (revenue loss) or SEV3 (degraded non-critical)?
  Anti-pattern: Running commands before understanding scope

STEP 2 — Hypothesis Formation
  Top 3 most likely causes based on symptoms.
  Rank by probability. Start with most likely.
  Anti-pattern: "Let me check the logs" with no hypothesis

STEP 3 — Systematic Investigation
  One hypothesis at a time. Specific commands. Specific outputs expected.
  Each step either confirms or eliminates a hypothesis.
  Anti-pattern: Shotgunning 10 commands hoping something looks wrong

STEP 4 — Confirm Root Cause
  Correlation ≠ causation. Can you reproduce? Can you explain the mechanism?
  Anti-pattern: "I restarted the pod and it's fixed" (that's mitigation, not root cause)

STEP 5 — Fix + Verify
  Apply fix. Verify with metrics/logs/user reports.
  Communicate to stakeholders.
  Anti-pattern: Fix applied, walk away without verification

STEP 6 — Prevention
  What stops this from happening again?
  Alert? Automation? Architecture change? Runbook?
  Anti-pattern: Closing the incident without follow-up items
```

### Framework 3: Deep Dive (STAR+)
```
S — Situation
  Context, constraints, scale, team, business environment.
  Keep it tight. 2-3 sentences max.

T — Task
  YOUR specific responsibility. Not the team's. Yours.
  What was expected of YOU?

A — Action
  What YOU did. Technical depth here.
  Decisions you made. Tradeoffs you navigated.
  Tools, architecture, code, process — specifics.

R — Result
  Quantified impact. Metrics. Business outcomes.
  "Reduced deploy time from 45 min to 8 min"
  NOT "it went well" or "the team was happy"

+ — Retrospective
  What would you do differently?
  What did you learn?
  What follow-up work remains?
  This is where L6 separates from L5.
```

### Framework 4: Coding
```
STEP 1 — Clarify (1-2 min)
  Inputs, outputs, edge cases, constraints.
  "Should this handle concurrent executions?"
  "What's the expected error behavior — retry or fail fast?"

STEP 2 — State Approach (1 min)
  "I'll structure this as [X]. Main components: [Y, Z]."
  Don't code in silence.

STEP 3 — Code Happy Path (15 min)
  Clean structure. Functions. Types.
  Talk while coding — explain choices.

STEP 4 — Error Handling (5 min)
  Every external call: what if it fails?
  Timeouts, retries, graceful degradation.

STEP 5 — Test / Verify (3 min)
  Walk through with example input.
  Edge cases: empty input, huge input, malformed input.

STEP 6 — Production Hardening (5 min)
  Logging, metrics, graceful shutdown, configuration.
  "In production, I'd also add [X]."
```

---

## INTERVIEW STRUCTURE

```
┌──────┬───────────────────────────────┬──────────┬───────────────┬─────────────┐
│Round │ Type                          │ Duration │ Interviewer   │ Exchange    │
│      │                               │          │ Persona       │ Limit       │
├──────┼───────────────────────────────┼──────────┼───────────────┼─────────────┤
│  0   │ Phone Screen (Breadth)        │ 30 min   │ Senior SRE    │ 15-20 Qs   │
│  1   │ System Design                 │ 45 min   │ Staff SRE     │ 1+6 exch.  │
│  2   │ Troubleshooting (Cross-domain)│ 45 min   │ Incident Lead │ 8 steps    │
│  3   │ Coding (Go + Python)          │ 45 min   │ Platform Eng  │ 1+3 exch.  │
│  4   │ Deep Dive (NovaMart)          │ 45 min   │ Eng Manager   │ 1+6 exch.  │
│  5   │ K8s & Infrastructure          │ 45 min   │ Principal Eng │ 5-6 Qs     │
│  6   │ Behavioral + Leadership       │ 45 min   │ Director      │ 6+ Qs      │
├──────┼───────────────────────────────┼──────────┼───────────────┼─────────────┤
│FINAL │ Hire Committee Debrief        │    —     │ All personas  │      —      │
└──────┴───────────────────────────────┴──────────┴───────────────┴─────────────┘

Total: ~5.5 hours of interview simulation
```

---

## ROUND-BY-ROUND SPECIFICATION

### ROUND 0: Phone Screen — Breadth Check
```
Interviewer Persona: Senior SRE (screening)
Duration: 30 min
Format: Rapid-fire. 15-20 questions across ALL domains.
  No deep dives. Breadth only.
  Short, precise answers expected. Not essays.

Domains covered:
  Linux, Networking, Git, Docker, Kubernetes, CI/CD,
  Terraform, Ansible, Prometheus, Loki, Tracing, SLOs,
  AWS, Security, Incident Response, General SRE

Purpose: Filter blind spots. Verify breadth before depth rounds.

Pass criteria: ≥80% correct → proceed to on-site loop
Fail criteria: <80% → gap list generated, remediation before continuing

Examples (style, not actual questions):
  "What's the difference between a Deployment and a StatefulSet?"
  "How does mTLS work in a service mesh?"
  "What happens when a Terraform apply fails halfway?"
  "Explain SLO burn rate in one sentence."
  "What's conntrack and why does it matter?"
  "What's the difference between SNAT and DNAT?"
```

### ROUND 1: System Design
```
Interviewer Persona: Staff SRE
Duration: 45 min
Format:
  - I give an open-ended design problem
  - You get ONE response for initial design
    (requirements + math + high-level + deep dive + failure modes + tradeoffs)
  - Then 4-6 follow-up exchanges (pushbacks, constraint changes, deep probes)
  - If initial design is incomplete, you don't get to "finish later"

Question domains:
  Deployment systems, observability platforms, multi-region architecture,
  DR/failover, secrets management, service mesh, platform self-service,
  CI/CD at scale, data pipeline reliability

What I grade:
  - Do you clarify requirements BEFORE drawing boxes?
  - Do you address scale, failure modes, cost?
  - Are tradeoffs explicit or hand-waved?
  - Can you zoom into any component when pressed?
  - Do you address day 2 operations?
  - Do you know what NOT to build?

Follow-up style:
  "What happens when [component X] goes down?"
  "Your boss says this costs too much. Cut 40%."
  "A new team joins with 50 services. Does your design hold?"
  "Show me the failure domain boundaries."
  "Oh, I forgot to mention — you also need to support GPU workloads."
```

### ROUND 2: Troubleshooting — Cross-Domain Live Debug
```
Interviewer Persona: Incident Commander
Duration: 45 min
Format:
  - I describe symptoms
  - Maximum 8 investigation steps
    (each step: you ask/run something → I give output)
  - If root cause not found in 8 steps, you failed triage
  - After root cause: fix + prevention in ONE response
  - At least ONE scenario requires knowledge from 3+ domains
    (e.g., networking + K8s + observability with CI/CD root cause)

Question domains:
  Network failures, K8s pod/node/cluster issues, database problems,
  CI/CD pipeline failures, observability pipeline breakage,
  cascading failures, split-brain, data corruption, latency spikes,
  intermittent failures (the hardest kind)

What I grade:
  - Do you triage IMPACT first or dive into logs?
  - Is your investigation systematic or random?
  - Do you ask the right questions before running commands?
  - Do you know WHICH tool to reach for and WHY?
  - Can you distinguish symptom from root cause?
  - Do you communicate clearly under pressure?
  - When I say "exec wants an ETA" — do you give a real answer?

Follow-up style:
  "Here's the output. What now?"
  "That command returned nothing. Next move?"
  "The developer says they didn't change anything. Do you believe them?"
  "It's been 20 minutes. The VP is asking for an update. What do you say?"
```

### ROUND 3: Coding — Production DevOps Tooling
```
Interviewer Persona: Platform Engineer
Duration: 45 min
Format:
  - 1-2 coding problems (NOT leetcode — real DevOps tools)
  - Clarify requirements: 1 exchange
  - Write code: 1 response (complete, working, real code — not pseudocode)
  - Follow-ups / refactoring: 2-3 exchanges
  - You MUST use Go for at least one problem and Python for at least one
    (if 2 problems given)

Code evaluation criteria:
  - Syntactically correct (I mentally compile it)
  - Error handling on every external call
  - Structured (functions, types, clear flow)
  - Edge cases considered
  - Production-ready touches (logging, config, graceful shutdown)
  - Readable by a junior engineer

Question domains:
  CLI tools (SLO checker, drift detector, resource auditor),
  Automation scripts (canary promoter, secret rotator, cleanup),
  K8s controllers/operators (conceptual), Terraform helpers,
  Pipeline glue (webhook handlers, notification formatters)

Follow-up style:
  "What happens if the API returns a 429?"
  "This runs as a CronJob. What if the previous run hasn't finished?"
  "Add structured logging. Now."
  "A junior engineer needs to maintain this. Refactor for clarity."
  "Now make it work for 500 namespaces, not 5."
```

### ROUND 4: Deep Dive — NovaMart Architecture Review
```
Interviewer Persona: Engineering Manager
Duration: 45 min
Format:
  - "Walk me through something complex you built recently."
  - You present NovaMart (your choice of which aspect to lead with)
  - I interrogate — 1 initial presentation + 4-6 follow-up exchanges

What I grade:
  - Can you explain architecture decisions to a non-expert?
  - Do you own the tradeoffs or blame constraints?
  - What would you do differently?
  - Do you understand the business impact of your technical choices?
  - Can you handle "why didn't you just use [simpler thing]?"
  - Do you know the limitations of what you built?

Follow-up style:
  "Why Linkerd instead of Istio?"
  "What's your blast radius if Karpenter bugs out?"
  "How do you know your SLOs are correct?"
  "What's the single biggest risk in your architecture right now?"
  "If you had to onboard 50 teams tomorrow, what breaks first?"
  "Walk me through a deploy from git push to production traffic."
  "What's your cost per request? How did you optimize?"
  "Your CTO says move to GCP. What's your migration plan?"
```

### ROUND 5: Kubernetes & Infrastructure Deep Dive
```
Interviewer Persona: Principal Engineer
Duration: 45 min
Format:
  - 5-6 questions, each with 2-3 depth levels
  - You answer each completely in one response
  - No "let me come back to that"
  - Every answer spawns follow-ups. Depth is relentless.

Question domains:
  K8s internals (API server, etcd, scheduler, controller manager),
  Networking (CNI, kube-proxy, service mesh, DNS, NetworkPolicy),
  Storage (CSI, PV lifecycle, etcd storage),
  Scheduling (topology spread, affinity, preemption, Karpenter),
  Security (RBAC, PSA, admission control, IRSA, mTLS),
  AWS infrastructure (VPC, IAM, EKS specifics, cross-account),
  Linux fundamentals that underpin all of it (namespaces, cgroups, iptables)

What I grade:
  - How deep can you go before you hit "I don't know"?
  - Do you say "I don't know" honestly or bullshit?
  - Can you reason from first principles past memorization?
  - Do you understand WHY behind every layer?
  - Can you connect kernel-level to K8s-level to application-level?

Follow-up style:
  "Go deeper."
  "What's actually happening at the kernel level?"
  "You said X. But that contradicts Y. Reconcile."
  "What if I told you that's wrong?" (even when it's right — do you fold?)
  "Explain this to me like I'm a junior. Now explain it like I'm a kernel developer."
```

### ROUND 6: Behavioral + Leadership
```
Interviewer Persona: Director of Engineering
Duration: 45 min
Format:
  - Minimum 6 questions, each with 2-3 follow-ups
  - STAR+ format expected
  - I push past rehearsed answers
  - I probe for SPECIFIC details, not generalities

Leadership Principles Probed:
  1. Ownership — Do you own outcomes end-to-end?
  2. Bias for Action — Do you move fast with calculated risk?
  3. Disagree and Commit — Can you fight then execute the opposite?
  4. Earn Trust — How do you handle being wrong publicly?
  5. Dive Deep — Do you know the details or delegate blindly?
  6. Think Big — Do you solve today's problem or next year's?
  7. Deliver Results — Impact. Not activity. Not effort. Impact.
  8. Hire and Develop — Can you mentor? Can you raise the bar?

I will specifically probe for:
  - A failure YOU caused (not the team, not the system — you)
  - A conflict you LOST
  - A time you were WRONG and how you handled it
  - A time you said NO to senior leadership
  - A time you had to influence without authority
  - A time you inherited a mess and how you dealt with it

What I grade:
  - Self-awareness vs ego
  - Ownership vs blame distribution
  - Specificity vs vague generalities
  - Growth evidence vs static personality
  - Can you be vulnerable without being weak?

Follow-up style:
  "What would you do differently now?"
  "How did you know that was the right call?"
  "What did your skip-level think about that?"
  "That sounds like it worked out. Tell me about one that didn't."
  "You said 'we.' What did YOU specifically do?"
  "Why should I believe that wasn't just luck?"
```

---

## GRADING SYSTEM

### Per-Round Grading
```
┌────────────────────────────┬────────┐
│ Criteria                   │ Weight │
├────────────────────────────┼────────┤
│ Technical Accuracy         │  25%   │
│ Depth of Understanding     │  20%   │
│ Structure / Clarity        │  15%   │
│ Production Awareness       │  15%   │
│ Tradeoff Articulation      │  10%   │
│ Communication              │  10%   │
│ Pressure Handling          │   5%   │
└────────────────────────────┴────────┘
```

### Verdict Scale
```
┌─────────────────┬─────────┐
│ Verdict         │ Score   │
├─────────────────┼─────────┤
│ Strong Hire     │  ≥ 85   │
│ Hire            │  70-84  │
│ Lean Hire       │  60-69  │
│ Lean No Hire    │  50-59  │
│ No Hire         │  < 50   │
└─────────────────┴─────────┘

TARGET: Strong Hire (≥85) in ALL 7 rounds.
```

### Post-Round Feedback Format
```
After every round:
  1. Score per criteria (weighted)
  2. Overall score + verdict
  3. Level tag per answer (L4 / L5 / L6 / L7)
  4. For any answer below L5:
     → Full model answer showing what L6 sounds like
  5. Specific improvement actions (concrete, not "study more")
  6. Pattern observations (recurring strengths/weaknesses across answers)
```

---

## HIRE COMMITTEE DEBRIEF (After All 7 Rounds)

```
Format:
  - Consolidated scorecard across all rounds
  - Overall verdict with justification
  - Strengths (what consistently impressed)
  - Concerns (what consistently worried)
  - Level assessment (L5 / L6 / borderline)
  - "Would I fight for this candidate in the hiring meeting?" — honest answer
  - Top 5 things to sharpen for a real interview
  - Comparison: where you'd land at Google / Amazon / Meta / Netflix
```

---

## RULES OF ENGAGEMENT

```
1. NO HINTS.
   I don't help you mid-answer. This is an interview.

2. "I DON'T KNOW" IS RESPECTED. BULLSHITTING IS NOT.
   "I'm not sure, but I'd reason through it like this..." = good.
   Confidently stating something wrong = red flag, score penalty.

3. I WILL PUSH BACK EVEN ON CORRECT ANSWERS.
   "Are you sure?" doesn't mean you're wrong.
   Hold your ground if you're right. Fold if you realize you're wrong.
   Both are signals. The wrong move is freezing.

4. I WILL CHANGE CONSTRAINTS MID-QUESTION.
   "Oh, I forgot to mention — the cluster also runs GPU workloads."
   "Actually, the team is 2 people, not 20."
   Real interviews do this. Adapt or die.

5. TIME PRESSURE VIA EXCHANGE LIMITS.
   Each round has explicit exchange limits.
   If your initial response is incomplete, you don't get to finish later.
   Manage your answer scope — be complete AND concise.

6. FOLLOW-UPS ARE NOT OPTIONAL.
   If I ask a follow-up, you answer it. Dodging = point deduction.
   "I covered that earlier" only works if you actually did.

7. ONE ROUND PER SESSION.
   Each round gets its own conversation turn. Full focus.
   After each round: full grade + detailed feedback + gap list.

8. ONE REDO — CHOOSE WISELY.
   If you bomb a round, you get ONE redo across all 7 rounds.
   Redo replaces the original score entirely.
   Use it strategically or don't use it at all.

9. BEHAVIORAL ANSWERS MUST BE SPECIFIC.
   "We improved the pipeline" = no credit.
   "I rewrote the shared library to add parallel stages, 
    reducing build time from 45 to 12 minutes" = credit.
   Vague = L4. Specific = L5+.

10. I WILL TELL YOU WHEN A ROUND STARTS AND ENDS.
    No ambiguity. Clean boundaries.
```

---

## EXECUTION ORDER

```
Step 1: You confirm this proposal (or request final changes)
Step 2: Round 0 — Phone Screen fires immediately
Step 3: Rounds 1-6 sequentially, one per session
Step 4: Hire Committee Debrief
Step 5: Gap remediation plan (if needed)
```

---

**This is the final proposal. Review it. Confirm or modify. Once locked, Round 0 starts immediately.**
