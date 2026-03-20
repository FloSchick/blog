---
title: "Using AI to Survive KubeCon Schedule Planning"
date: "2026-03-20"
tags: ["KubeCon", "AI", "Claude", "Kubernetes", "Productivity", "Python"]
ShowToc: true
TocOpen: true
---

KubeCon EU 2026 in Amsterdam is my fourth KubeCon. You'd think by now I'd have the schedule planning thing figured out. I don't. Every time it's the same story: hours lost on sched.com, tab overload, a half-finished spreadsheet, and the nagging feeling that I've missed something good.

This time I decided to let AI do the heavy lifting — and the result was a searchable web app, a curated recommendation schedule, and an on-the-go assistant I can query from my phone during the conference itself.

---

## The Problem with KubeCon Schedule Planning

KubeCon is enormous. EU 2026 has over **540 sessions** across 4 days: a full Monday of co-located events (ArgoCon, CiliumCon, OpenTofu Day, Agentics Day, and more), followed by three days of the main conference. Add keynotes, lightning talks, panels, workshops, and contribfests and you're staring down a genuinely overwhelming programme.

The official sched.com interface isn't bad, but it's designed for browsing one session at a time. There's no way to search across descriptions, filter by topic tag, or get a cross-day view of everything that touches, say, ArgoCD or platform engineering. Every year I end up with 30 open tabs and a growing list of "I'll decide later" sessions that I never actually decide on until 8am the morning of.

Beyond that, understanding whether a talk is actually relevant requires reading the full abstract — and doing that for hundreds of sessions by hand is genuinely time-consuming.

---

## The Approach: Build a Database First

The first thing I did was build a proper database of every session. I used Claude to fetch and parse individual event pages from sched.com at scale — around 500+ pages — and extract structured data into a CSV: title, speakers, organizations, room, track, content level, full description, and direct event URL.

```
day, date, time_start, time_end, session_type, title, speakers,
organizations, room, track, content_level, description, event_url
```

542 rows. Every co-located event, every main conference session, all four days.

With that in place, I could actually query the data. Want all talks mentioning Crossplane? One Python query. Want everything tagged as a tutorial on Wednesday afternoon? Done. It turned a needle-in-a-haystack problem into something tractable.

---

## The Recommendation Schedule

Once the database existed, I gave Claude context about our infrastructure: we run ArgoCD for GitOps, we're evaluating Crossplane as a potential Terraform replacement, we use Cilium for networking (but don't connect clusters, so ClusterMesh isn't relevant), we care about platform engineering, multi-tenancy, observability, and identity.

The output was a curated Markdown schedule with priority ratings (★★★ = must-see, ★★ = strong interest, ★ = worth attending if no conflicts), verified abstracts pulled from the actual event pages, and a cross-week highlights table. Critically, the recommendations were based on *actual descriptions*, not just titles — so no more picking talks based on a vague heading and ending up in the wrong room.

A few highlights that came out of this process that I wouldn't have found otherwise:

- **Crossplane — The Cloud Native Framework for Platform Engineering** (Tue 15:15) — rated ★★ as a primary Terraform-replacement candidate
- **Backend-First IDP: ArgoCD + Crossplane + OPA** (Mon 15:20, ArgoCon) — exactly the stack we're moving toward
- **Kargo: Progressive GitOps Delivery** (Mon 11:40, ArgoCon) — bridges the gap between GitOps and deployment orchestration

---

## The Web App

A Markdown file in a folder is fine, but not great for a conference where you're standing in a corridor with 60 seconds to decide which room to run to. So I built a small searchable web app and published it on GitHub Pages.

**[floschick.github.io/kubecon-eu-2026](https://floschick.github.io/kubecon-eu-2026/)**

Features:
- Full-text search across titles, descriptions, speakers, and organizations — with live highlighting
- Day filter chips (Mon / Tue / Wed / Thu, color-coded)
- 22 topic tag filters: `#argocd`, `#crossplane`, `#terraform`, `#otel`, `#mcp`, `#cilium`, `#gitops`, `#platform-eng`, and more — auto-generated from keyword matching on the CSV data
- Expandable cards with room, speakers, session type, and a direct link to the sched.com event page
- A Schedule tab that renders the full curated Markdown recommendation file

The whole thing is a single `index.html` + a `data.js` file (~500KB, the full dataset as a JS array) + the `schedule.md`. No build step, no framework, no server — just static files on GitHub Pages.

The tags were generated automatically with a small Python script that matched keywords across title + description + track fields. The top tags by volume: `#ai` (92 talks), `#observability` (91), `#platform-eng` (84), `#ai-agents` (67), `#otel` (65).

---

## The Claude Projects Context

The web app covers the "I need to decide right now" use case. But there's a related one: "I'm waiting in a keynote queue and want to think through my afternoon options."

For that I added both the CSV database and the recommendation Markdown file to a **Claude Project** context. Now from the Claude mobile app I can ask questions like:

> *"What are the best Crossplane talks on Tuesday and what time do they conflict with each other?"*

> *"Is there anything on Wednesday morning about OPA that isn't a keynote?"*

> *"Give me a 30-minute slot filler at 16:00 on Thursday that's about observability."*

Claude has the full dataset in context and can answer these immediately, without me having to open a browser, remember which filters to set, or scroll through cards on a small screen. It's a genuinely different interaction pattern — more like asking a well-briefed colleague than searching a website.

---

## What Worked Well

**Starting with the data.** Building the CSV first meant everything downstream — the recommendations, the web app, the mobile context — was working from the same ground truth. No inconsistencies, no "I think this talk is about X based on the title."

**Parallelism.** Fetching 500+ event pages one at a time would have been painfully slow. Running multiple agents in parallel (8 concurrent scraping agents) brought it down to a manageable time window.

**Separation of concerns.** The database, the recommendations, and the UI are three separate artifacts that serve different needs. I could update the recommendations without touching the app, or query the CSV directly without loading the web page.

---

## Final Thoughts

This is my fourth KubeCon and the first time I've arrived feeling genuinely prepared rather than overwhelmed. The combination of a structured database, AI-assisted curation, a fast mobile-friendly search interface, and an on-demand assistant covers all the situations I actually run into at the conference.

The tooling to do this — capable LLMs, GitHub Pages for free static hosting, sched.com's public event pages — has existed for a while. What changed is that the AI layer is now good enough to do the tedious extraction and synthesis work that previously made this feel not worth the effort.

If you're going to KubeCon and want to try a similar approach for your own interests, the [source repo](https://github.com/FloSchick/kubecon-eu-2026) is public. The CSV schema and tag taxonomy are easy to adapt.

Oh, and the whole thing — database, recommendations, web app, and this blog post — was done in **under an hour**. That still feels a bit absurd to type out. Previous years I'd spend an entire evening on schedule planning and still feel underprepared. This time it was done before dinner.

See you in Amsterdam.

---

*P.S. — Yes, this blog post was also written by Claude. Based on the KubeCon sessions context, naturally. At some point I stopped being the author and started being the editor. I'm at peace with this.*
