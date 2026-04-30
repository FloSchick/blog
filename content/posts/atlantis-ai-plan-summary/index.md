---
title: "Adding an AI Plan Summary to Terraform PRs"
date: "2026-04-30"
draft: true
tags: ["Terraform", "Atlantis", "AI", "Mistral", "Claude", "Grafana", "Observability", "Platform Engineering"]
ShowToc: true
TocOpen: true
---

The thing that finally broke me was a Renovate PR that bumped a single Helm chart version and produced **eleven** Terraform plan comments on a GitHub PR. Each one a wall of `+`, `-`, and `~` lines wider than a 14" laptop. Eleven, because we have 37 Terraform projects and 11 of them transitively depend on that chart. Technically the plan was correct. Practically, nobody on the team was going to read that.

We use [Atlantis](https://www.runatlantis.io/) as our Terraform CI — it's an open-source server that runs `terraform plan` on every PR, posts the output as comments, and applies once a reviewer drops an `atlantis apply` comment. It's the right tool for the job. But the format it produces — raw `terraform plan` output, mechanically pasted into PR comments, split across GitHub's 60 KB issue-comment limit — is built for completeness, not for a human's morning review. Once you're at three comments per project on a multi-project plan, you stop reading and start hoping your CI is right.

That's the moment I decided I wanted a layer on top.

This post is about what I built, and the bits I think are worth telling you about — including the parts where I went down rabbit holes and the security gotchas I had to learn the hard way.

---

## What I Built

A small Python agent that runs as an Atlantis **post-workflow hook** after every `atlantis plan`. It walks Atlantis's data dir for the per-project planfiles, extracts the changed resources, asks an LLM (Mistral by default, Claude as the alternative) for a one- or two-sentence summary per project, and posts a single combined comment on the PR alongside Atlantis's standard output.

The comment doesn't repeat what Atlantis already shows. Atlantis already lists every changed resource, with full diffs, in its own comments. The agent's job is the *insight* layer: what does this change actually accomplish, and is anything in it unexpected.

---

## What a Comment Looks Like

Here's a representative comment on a multi-project PR (names genericised). Three projects, mixed change shapes — exactly the kind of plan where the raw Atlantis output is hardest to scan:

> **🤖 Plan summary**
>
> **3 projects with changes** — `7c3a91e`
>
> ---
>
> **`env/prod-eu-1/infrastructure`**
> 1 to change
>
> Bumps the Kubernetes version from v1.32.8 to v1.32.9 on the prod cluster — routine security patch in the existing 1.32 line.
>
> ---
>
> **`env/prod-eu-1/customer-instances`**
> ⚠️ **1 to destroy** · ♻️ **1 to replace** · 5 to add
>
> Onboards new customer `acme` — adds a dedicated namespace, MySQL database, scheduled backup CronJob, ArgoCD application, and DNS record. **Heads up:** the same plan also replaces the legacy shared MySQL instance because its `performance_class` attribute changed (forces replacement), and destroys the deprecated `customer-trial-2024` namespace. Both look unrelated to the onboarding — worth confirming this is intentional in one PR.
>
> ---
>
> **`env/prod-eu-2/infrastructure`**
> 2 to change
>
> Updates the Traefik ingress IP allowlist (adds two GitHub IP ranges) and the cert-manager ClusterIssuer's ACME email — routine config tweaks.
>
> > _Changes to attributes Terraform marks as sensitive are excluded from this summary. Refer to Atlantis's plan output for full details._

The thing I want to point out: the second project. The customer onboarding is the obvious story, but the same plan also replaces a shared MySQL instance and destroys a namespace — both changes that are easy to miss in 60 KB of `+` and `-` lines. The "Heads up" clause is the moment the agent earns its keep. The first and third projects, by contrast, get one-sentence summaries because they don't need more.

---

## A Side Trip Through AI Observability

While I was building this, Grafana shipped [AI observability for Grafana Cloud](https://grafana.com/docs/grafana-cloud/monitor-applications/ai-observability/). I wired the agent into it as a side experiment because I had a specific question: I picked **Mistral-small** to keep cost and latency down, and Mistral-small has a 32K context window. Atlantis plans vary wildly — a Renovate provider bump might be 200 tokens of reduced JSON, a multi-project chart upgrade can blow past 10K. Was I quietly hitting context limits, or running at 5% utilization?

Token-usage histograms answered that. Running the agent against a handful of PRs over a few days — small Renovate bumps, a couple of multi-project chart upgrades, one customer onboarding — gave me an actual distribution: well under the 32K ceiling, with a long tail I now know to watch. The same data made the cost story concrete — fractions of a cent per summary — which turned the cheap-model choice from a vibe into a number on a graph.

If you're picking a small model for a workload with high size variance, having token-usage telemetry from day one turns "I think it'll fit" into a chart you can point at.

---

## Keeping Secrets Out of the Prompt

The riskier feature is what I call deep-diff mode: optionally include attribute-level diffs in the prompt so the LLM can write "bumps the Kubernetes version from v1.32.8 to v1.32.9" instead of vague "updates the cluster". By default it's off — we send only the resource address and action.

When you turn it on, you have to confront an unpleasant fact: `terraform show -json` does **not** redact sensitive values. It instead provides parallel `before_sensitive` / `after_sensitive` mirrors flagging which paths are sensitive, and leaves the actual values in `before` / `after`. The text-mode `terraform show` redacts. The JSON form doesn't.

*Worth a detour: Terraform's [JSON output format reference](https://developer.hashicorp.com/terraform/internals/json-format) lays out the full schema — `before`, `after`, `before_sensitive`, `after_sensitive`, `after_unknown`, `replace_paths`, `importing`, and a few more I hadn't bumped into. I'd been calling `terraform show -json` for years without reading the schema; this project finally pushed me to, and there's noticeably more in the `change` object than you'd guess from a casual look.*

So a naive walker that just diffs `before` against `after` would happily ship `admin_password: "hunter2" → "hunter3"` straight to the LLM and from there to Grafana's generation ingest. Where it would sit in the conversation log forever.

The fix is a recursive walker that takes the sensitive mirror as a parallel argument and **drops the entire subtree** when either side flags it sensitive — no value, no path, no `(sensitive)` placeholder. Just gone.

```python
def walk_diff(before, after, before_sens, after_sens, path=""):
    if before_sens is True or after_sens is True:
        return []        # nothing emitted; no leak vector
    # …recurse into dicts/lists by key/index pairwise…
```

The leak-detection tests are the strict ones — every test searches the entire serialized output for any trace of the secret value. If the redaction logic ever regresses, the tests fail loudly. There's also a static disclaimer at the bottom of every PR comment so reviewers know that *changes to attributes Terraform marks as sensitive are excluded from this summary; refer to Atlantis's plan output for full details.*

What this still doesn't catch: secrets embedded in non-sensitive string fields (a token in a cloud-init `user_data`, a kubeconfig blob in helm values), and modules that forgot to mark a `variable` as `sensitive = true` in the first place. Both are documented; neither is something the agent can fix on its behalf.

---

## The Opt-In That Stays Off

Even with sensitive redaction, deep-diff mode means non-sensitive attribute values travel off-cluster — to the LLM provider and to Grafana. For most resources that's fine (regions, sizes, image tags) but it's a real change in your data-exfil surface, so deep-diff defaults to **off**. In lean mode the LLM sees only the resource address, type, and action — summaries are vaguer but the surface is essentially zero.

---

## What the Comment Should Not Replace

The most realistic failure mode of this whole project isn't a 500 from Mistral or a bug in the walker. It's *people*. Colleagues see "summary looks fine" and skip the actual Atlantis plan comments. Once that habit forms, the agent has made review *worse*, not better — LLMs do hallucinate, miss context the reduced JSON didn't carry, and occasionally write a confidently-wrong sentence. And by design, sensitive attribute changes don't appear in the summary at all.

A few things in the current implementation push against this:

- Counts and the destroy/replace emphasis line are mechanical, extracted from the planfile JSON — not from the model. Even if the prose softens or omits a destroy, the `⚠️ **N to destroy**` line above it won't.
- The sensitive-fields disclaimer is in every comment, every time, reminding the reader to refer to the actual plan for that class of change.
- Atlantis still requires a human to type `atlantis apply` after seeing the actual plan output. The AI summary isn't part of the apply path.

But those are passive. The active guardrails have to be cultural:

- The summary is a *hint*; the plan is the source of truth. If the summary contradicts your gut on a prod PR, the plan wins, every time.
- Don't approve faster just because there's a summary. A missing or unhelpful summary should feel exactly the same as no summary.
- For high-blast-radius changes (prod data, IAM, networking, anything stateful), read the full plan regardless of what the summary says.

The agent earns its keep on the routine 80% of plans where it correctly says "version bump, nothing weird." The remaining 20% — the surprising plans — are exactly when you can't fully trust the agent to spot the surprise.

---

## What's Still Rough

A few things I'd do differently or haven't done yet:

- **Path-only context for nested changes.** When `node_pool[0].max_nodes` changes, the LLM sees the path and the before/after value, but not the unchanged `name` sibling field that says *which* pool. So summaries can be vague about identity. The fix is a "context anchor" pass that includes `name` / `id` / `key` siblings of changed paths even when those siblings don't differ. Defer until I see this matter on real PRs.
- **Single LLM call, no provider fallback.** If Mistral is down, the comment doesn't post. The agent fails open (Atlantis plan still succeeds), but you lose the summary. Trivial to add a Claude fallback; haven't bothered yet.
- **Deep diff is a global toggle, not per-project.** Today it's a single env var on the Atlantis pod — all projects opt in or all stay in lean mode. Per-project opt-in (deep for `env/dev/*`, lean for prod) would be a clean addition.

---

## Closing

The agent has been running on our Atlantis instance for a couple of days. Reviewing PRs already feels noticeably faster — the summary catches the obvious "yep, this is the version bump I expected" and flags the not-so-obvious "this Renovate PR also forces a node pool replacement because performance class changed" cases. Rough-edge gotchas above notwithstanding, the part that still surprises me is how often the comment is shorter than the PR title.

The agent is around 750 lines of Python with 64 unit tests. Worth it.
