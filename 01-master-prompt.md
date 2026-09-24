# Master Prompt — HyperProbe Source Code Security Audit

> Copy everything in the fenced block below and paste it into Claude Code, running from **inside the `prodprobe-master` directory**, with `../hyperprobe-audit-pack/` alongside it. Turn on deep mode first with `/effort ultracode`.

---

```
ultracode

You are the lead reviewer on an independent, third-party audit-grade secure source code audit of this
repository (product: HyperProbe; monorepo: prodprobe). Treat this as a rigorous third-party
assessment whose written output must withstand external scrutiny. Depth, correctness and evidence
matter far more than speed or token cost. Do not flatter the code and do not soften real issues.

================================================================================
AUTHORISATION & SAFETY (read first, non-negotiable)
================================================================================
- Scope of ACTION is this source tree only. You review code by READING it and by building small
  LOCAL proof-of-concepts in a throwaway lab. You do NOT attack, scan, or connect to any live or
  production HyperProbe deployment. Reading source is not authorisation to touch a running system.
- Secrets are radioactive. If you find a live credential, private key, API token or password in the
  code, report THAT IT EXISTS at file:line, redacted (e.g. show the first 4 chars). Never use it,
  never test it against any service, never print it in full, and recommend rotation. Note that any
  key already committed to a public repo must be treated as compromised regardless of its power.
- Do not run the target's own build/test/postinstall scripts to "see what happens" without reading
  them first — a repo under review can contain hostile install hooks. Read build.sh, package.json
  scripts, .husky/*, and any postinstall before executing anything. Prefer read-only analysis; when
  you must run something, run it in the disposable Docker lab, never against real infrastructure.
- The demo/test repo hp-test-payment-demo (github.com/hypertestco/hp-test-payment-demo) is a LOCAL
  TARGET ONLY, used to exercise the SDK/MCP end to end. Do NOT report findings about the demo app's
  own code (it deliberately contains a bug for teaching). Its purpose here is to stand up a lab.

================================================================================
INPUTS PROVIDED TO YOU
================================================================================
The folder ../hyperprobe-audit-pack/ contains the plan and standards you will follow:
  02-audit-plan.md      — scope, architecture, threat model, per-component review targets
  03-checklist.md       — the control checklist you must work through and mark
  04-references.md      — the standards + tools to map findings to (all links verified)
  05-finding-template.md— the EXACT format every finding file must use
  06-seed-leads.md      — UNVERIFIED first-pass hypotheses (see "On seed leads" below)
Read all of these before starting. If a real project skill named `hyperprobe-source-audit` or
`source-code-review` is available, load it — it encodes this methodology in more detail.

================================================================================
ON SEED LEADS  (important — do not anchor on them)
================================================================================
06-seed-leads.md lists things a quick first read found suspicious. They are LEADS, NOT FINDINGS.
For each one: independently trace the real code path and either PROVE it (attacker-reachable source
-> dangerous sink, guard absent/wrong, plus a local PoC where feasible) or DISPROVE it and say why.
Then look far beyond the list. The seed leads are a floor, not a ceiling. The highest-value bugs in
a codebase like this — broken access control / IDOR, multi-tenancy leaks, auth/session logic, race
conditions, business-logic flaws — are usually NOT the ones a first skim catches. A scanner matches
patterns; you trace trust. Budget most of your effort on the surfaces the leads did NOT mention.
Do not treat a seed lead as confirmed just because it sounds plausible.

================================================================================
METHOD — run this as multi-agent workflows (ultracode)
================================================================================
There is a ready workflow script at ../hyperprobe-audit-pack/workflow/hyperprobe-audit.js. You may
run it, adapt it, or author your own — but the SHAPE must be:

  PHASE 1  MAP (fan-in): agents read the architecture and produce a shared inventory — entry points
           (tRPC routers, the gRPC broker/logger, NATS consumer, MCP server tools, VS Code webview
           messages, licensing lambda, cron jobs), dangerous sinks (expression eval in every SDK,
           JSON/deserialization, child_process, Redis Lua, Prisma queries, URL fetches), the trust
           boundaries, the framework/lib versions, and the dependency manifests. Write it to
           audit/00-attack-surface-map.md.

  PHASE 2  REVIEW (fan-out, no agent-count limit): spawn independent reviewers along BOTH axes and
           let them run in parallel —
             • per COMPONENT: backend, dashboard, logger(gRPC), logger-consumer, database/prisma,
               node-sdk, python-sdk, java-sdk, ruby-sdk, mcp-server, vsc-extension, licensing-server,
               common/license-manager, infra (docker/consul/traefik/nats), CI (.github/workflows).
             • per CLASS (cross-cutting): broken access control / IDOR / multi-tenancy; authn &
               JWT/session; injection (SQL/NoSQL/command/SSTI via the probe expression evaluators/
               path traversal); the SDK expression-evaluation sandboxes (Python ast, Node
               v8-inspector evaluateOnCallFrame, Java MVEL, Ruby binding.eval) and their "unsafe"
               toggles; SSRF (Sentry/observability outbound); crypto & secret management (license
               signing, credential keyring, token secret); serialization/redaction of captured
               production data; reliability/DoS on the HOST application the agent runs inside;
               supply chain & Docker/Consul/Traefik exposure; and AI/MCP-specific risks.
           Each reviewer returns STRUCTURED candidates (see schema in the workflow), never prose.

  PHASE 3  VERIFY (adversarial, per candidate): for every candidate, spawn independent skeptics
           prompted to REFUTE it. A candidate survives only if it can't be refuted — i.e. the
           source->sink path is real and reachable and the guard truly is missing/wrong for THAT
           sink. Where feasible, confirm with a minimal local repro (a unit test or a docker-lab
           request), never against production. Kill plausible-but-unreachable candidates. Default to
           "refuted" when uncertain. Record the refutation reasoning even for survivors.

  PHASE 4  CHAIN & RANK: for each survivor, state the capability it grants and what it unlocks when
           chained with others (e.g. exposed Consul -> license key rotation -> agent shutdown;
           unauth broker -> probe injection -> arbitrary expression eval inside customer prod app).
           Assign CVSS v3.1 (vector + base score) and a Critical/High/Medium/Low/Info rating.

  PHASE 5  SWEEP FOR VARIANTS: after any confirmed finding, immediately re-check sibling code for the
           same defect shape (the same mistake recurs — same missing ownership check on sibling
           endpoints, same eval toggle in the other 3 SDKs, same string-built query elsewhere).
           Loop phases 2–5 until two consecutive rounds find nothing new.

  PHASE 6  WRITE UP: one Markdown file PER confirmed finding using 05-finding-template.md exactly,
           filed under the right category folder. Build the register and stats. Then a completeness
           critic agent asks "what did we NOT cover?" and that becomes another round.

Use the disposable lab in ../hyperprobe-audit-pack/ guidance (docker compose in infra/) only if you
need dynamic confirmation; keep it isolated and local.

================================================================================
WHAT TO REPORT (7 categories — one main folder, a subfolder each)
================================================================================
Write findings under audit/findings/ :
  01-security/        broken access control, authn/session, injection, SSRF, crypto, secrets,
                      infra exposure that is a direct security issue
  02-ai-mcp/          MCP + AI-agent risks: prompt/tool-output injection via captured production
                      data flowing to the coding agent, tool poisoning, MCP token handling/scope,
                      over-sharing of context. Map to OWASP LLM Top 10 + OWASP MCP Top 10.
  03-functional-bugs/ logic errors, wrong results, unhandled promise rejections, crashes, incorrect
                      state transitions (security-relevant or not)
  04-reliability-dos/ resource exhaustion, unbounded queues/recursion/serialization, missing
                      timeouts, and ANYTHING that could degrade or crash the CUSTOMER'S host app
                      (this is an agent that runs inside production — host-safety is paramount)
  05-privacy-data/    PII / secrets captured into snapshots, redaction gaps, sensitive values in
                      logs, data retention, cross-tenant data visibility
  06-supplychain-infra/ vulnerable/abandoned deps (map to CVE/GHSA), lockfile issues, dependency
                      confusion, Dockerfile, Consul/Traefik/NATS config, GitHub Actions security
  07-code-quality/    weak typing/`any`, dead code, insecure defaults, missing/weak tests, footguns

Each finding: file:line, traced path, CVSS v3.1 + rating, and a remediation with a short corrected
code snippet. Map every finding to CWE + the OWASP category + (where it fits) ASVS 5.0 ID + CERT-In.

================================================================================
DELIVERABLES YOU PRODUCE
================================================================================
  audit/00-attack-surface-map.md        — the Phase 1 map
  audit/findings/<category>/<ID>-*.md   — one file per finding (IDs: SEC-, AIMCP-, BUG-, DOS-,
                                          PRIV-, SUP-, CQ-)
  audit/findings-register.md            — master table of every finding: ID, title, category,
                                          severity, CVSS, file:line, status
  audit/findings-register.csv           — same table as CSV (for the Excel tracker)
  audit/coverage.md                     — the checklist marked done/partial/not-covered, honestly,
                                          with what was skipped and why
  audit/executive-summary.md            — risk posture, counts by severity, top risks, themes

================================================================================
QUALITY BAR (how you'll be judged)
================================================================================
- No finding without a traced, reachable path. Label anything hypothetical as hypothetical.
- No impact inflation. A missing check is only "critical" if you show what it actually lets an
  attacker do. Downgrade honestly.
- Never stop at the first finding or at the seed leads. Enumerate siblings. An empty scanner run is
  not a stop condition — it raises the priority of the logic/authz bugs only source review can find.
- Prefer the strongest, simplest fix, and note when a whole class needs a systemic fix (e.g. a
  single authz helper) rather than a per-site patch.

Begin with Phase 1. Post the attack-surface map, then proceed. Keep a running ledger of candidates
with confidence scores. Ask me only if scope is genuinely blocked; otherwise proceed and note
assumptions.
```

---

## Notes for the human running this

- If you did **not** enable ultracode, delete the first line `ultracode` and instead tell Claude Code: *"Use a workflow with as many agents as needed; do not cap the agent count."*
- The prompt is deliberately explicit about **host-app safety** and **not touching production** because HyperProbe agents run *inside* customers' live applications — that is the single biggest real-world risk surface and the thing an external reviewer will be asked about first.
- If Claude Code proposes a workflow, review the planned phases on the approval card before allowing it. Then let it run and monitor with `/workflows`.
- Re-run the loop (Phases 2–5) at least twice; the variant sweep is where the second tier of findings comes from.
