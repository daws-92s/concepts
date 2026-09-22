# DevOps & SDLC Overview

## What this is / why it matters
Software gets built in stages — plan it, design it, build it, test it, ship it, maintain it. That's the SDLC (Software Development Life Cycle). Over time, teams found that doing these stages once, in strict order (Waterfall), was too slow and too risky. Agile broke the work into small chunks so problems get caught early. DevOps then added automation and operations practices on top of Agile, so releases go out faster and more reliably.

The short version: DevOps exists because slow, risky releases lose to fast, reliable ones.

## How it works (concept-level, plain language)

**Stakeholders** — anyone affected by the system, not just "the client." In a company that could mean: your team, management, developers, testers, operations, security, and end users. Good planning accounts for all of them, not just the people writing code.

**The SDLC phases**:
1. Requirements gathering — what does the client actually need?
2. Planning and analysis
3. Design
4. Development — building it
5. Testing
6. Deployment — releasing it
7. Maintenance — fixing and improving it after release

**Waterfall** does these phases once, in order, without going back. Everything gets tested only at the end, in one big batch. If something's wrong, it's expensive to fix because so much has already been built on top of it.

**Agile** breaks the same work into small cycles called sprints (usually 1–2 weeks). Each sprint builds one small piece, tests it, and gets feedback — instead of waiting months to find out something's wrong. Smaller batches mean cheaper, earlier fixes.

**Why multiple environments** (DEV, SIT, UAT, PRE-PROD, PERF, SEC, PROD): you don't test on real customers first. Each environment is a checkpoint before the next:
- **DEV** — developers build and test their own code
- **SIT** (System Integration Testing) — check that all the pieces work together
- **UAT** (User Acceptance Testing) — real users or business people confirm it meets their needs
- **PRE-PROD** — a near-exact copy of production, final check before go-live
- **PERF** — test how it holds up under heavy load
- **SEC** — test for security holes
- **PROD** — the real, live system

Each stage catches different problems before they reach actual customers.

**DevOps on top of Agile** adds:
- Daily testing, not just testing at the end of a sprint
- Testing both valid inputs (should succeed) and invalid inputs (should fail safely)
- Operational concerns: high availability, auto-scaling, security, and cost control
- Automated build-and-release, instead of manually pushing code out

## Problems I hit + how I solved them
Waterfall's main problem: all testing happens at the very end, against the whole system, so defects show up late and are costly to fix.

The fix isn't a new tool — it's a process change:
- Break work into smaller pieces (sprints) so defects show up early, against a small piece of the system
- Test more often (daily, not just at the end of a sprint)
- Add environments between "my laptop" and "real customers" (SIT, UAT, PERF, SEC) so problems get caught in a safe place first

## Key takeaways
- DevOps is a response to a business problem: slow releases lose market share, even if they're well-tested.
- Waterfall's weakness is that it delays all risk to the end. Agile's strength is catching problems early, in small pieces.
- Multiple environments exist so failures happen somewhere cheap (DEV) before they can happen somewhere expensive (PROD).
- DevOps builds on Agile — Agile is about how software gets planned and built; DevOps is about how it gets tested, released, and operated, with automation.
- Good process change is incremental: understand what's already working, prove improvements small, then roll them out through DEV → SIT → UAT → PROD.
