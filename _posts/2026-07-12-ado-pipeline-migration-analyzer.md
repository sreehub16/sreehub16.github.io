---
title: Building a Migration Readiness Analyzer for Azure DevOps to GitHub Actions
date: 2026-07-13 10:00:00 -0400
categories: [Azure DevOps, GitHub Actions]
tags: [azure-devops, github-actions, migrations, python, powershell, github]
description: >-
  What I learned building a Phase 0 discovery tool for Azure DevOps to GitHub
  Actions migrations - the assumptions that turned out wrong, the bugs that
  taught me the most, and the design decisions I'd make again.
---

## Why I built this

I work as a GitHub migration SME, spending most of my time helping teams move off Azure DevOps Pipelines and onto GitHub Actions. The part nobody tells you going in: the hard problem isn't converting YAML syntax. It's knowing, before you touch a single pipeline, which ones are safe to convert directly, which ones will fight you, and which shared infrastructure - templates, service connections, variable groups - actually matters most to get right first.

At enterprise scale (I've worked environments with 80,000+ repos), you cannot discover that complexity by opening pipelines one at a time. You need a Phase 0 discovery pass before conversion starts.

So I built one: a five-stage tool that inventories an Azure DevOps project, analyzes every pipeline's real complexity, and produces a report a migration team can actually plan against. This post is less "here's what it does" and more "here's what I got wrong along the way and what fixing it taught me" - because the wrong turns were more instructive than the parts that worked on the first try.

## The shape of it

```
1. PowerShell inventory   → discovers every pipeline in the org
2. Selector                → picks a project, splits YAML vs Classic
3. Fetcher                 → pulls raw + expanded YAML per pipeline
4. Rule engine              → tiers each pipeline: Direct / Refactor / Redesign
5. Report generator         → Markdown + Excel, executive summary + detail
```

PowerShell for discovery, Python for everything downstream. Not a stylistic choice - the PowerShell inventory script had years of hardening behind it already (pagination edge cases, throttling, encoding safety for non-ASCII pipeline names) and porting working code to Python for consistency's sake would have been pure busywork.

## The bug that taught me the most: raw vs. expanded YAML

Early on, I added Azure DevOps' Pipeline Preview API to the fetcher - a dry-run endpoint that resolves every template reference into one fully flattened YAML file, without actually triggering a build. It felt like a clean win: instead of writing a recursive template resolver myself, let ADO's own engine do the resolution and hand me the real, final structure.

I made the rule engine analyze that expanded file for everything, including template/`extends` detection. It worked. Then a customer-shaped test case - a genuinely modular pipeline, three levels of nested templates across repos - came back tiered as **Refactor** when it should have been **Redesign**.

The bug: **expansion doesn't just resolve templates, it erases every `template:` key from the file.** That's the entire point of expansion - by the time you're looking at the flattened output, there's nothing left that says "this came from a template." I'd built a check that depended on evidence the very API call I was using had already destroyed.

The fix wasn't complicated once I understood it: route template/`extends` detection to the **raw** file specifically, and route everything else - tasks, variable groups, approval gates, service connections - to the **expanded** file, since that's where the real, fully-resolved content lives. Two different questions, two different sources of truth, and no single file answers both.

The instructive part wasn't the fix. It was realizing "expanded" and "complete" aren't the same thing, and a feature can be technically correct and still quietly wrong about what it's looking at.

## The GitHub-rewired scenario, and a scare that turned out to be good news

A large chunk of any real customer's estate will have repos that have already moved to GitHub while the pipeline definition itself still lives in Azure DevOps - a common mid-migration state. Azure DevOps' own Git Items API can't reach a GitHub-hosted repo, so my first version simply marked these pipelines `unsupported` and moved on. Given how common this state actually is, that meant the tool's best feature - template and shared-repo leverage analysis - went dark for exactly the pipelines that mattered most.

**Step 1** was a bet worth testing empirically rather than assuming: the Preview API operates at the *pipeline* level, not the repo level. Azure DevOps' own execution engine already knows how to reach a GitHub repo through the pipeline's service connection when it actually runs. Would the same hold true for a dry-run Preview call?

It did. Removing the early bail-out and letting Preview run regardless of repository type immediately restored full task, pool, variable-group, and approval-gate analysis for GitHub-sourced pipelines - no new credentials required. The only genuine gap left was the raw file, needed specifically for template detection.

**Step 2** closed that gap: fetch the raw YAML directly from GitHub's Contents API, using the exact repo and filename the pipeline definition itself reports - never guessed.

Testing it produced a real scare. I compared a raw file I'd pulled from GitHub against the tool's expanded output, and they were clearly different pipelines - one had an active deployment task the other had commented out, different trigger blocks entirely. For a moment it looked like the tool was silently fetching the wrong content.

It wasn't. Tracing it down, the mismatch came from a manual research misstep on my end while investigating - not a bug in the fetch logic. But the process of ruling that out was worth more than if it had just worked cleanly the first time: it forced me to verify, concretely, that the tool fetches from the pipeline's *actual configured file and branch* - never assumed, never guessed - which is exactly the property you want from something producing a migration risk assessment. A near-miss that validates your design decisions is a good outcome, even when it doesn't feel like one in the moment.

## Designing the report: bounded beats complete

The Excel report went through more redesigns than any other part of this project, and the lesson generalizes past this one tool.

My first version of the "which pipelines use this shared template" view joined every consuming pipeline's name into a single cell. That works fine with ten pipelines. A template consumed by hundreds of pipelines in a real enterprise estate will overflow Excel's 32,767-character cell limit - and long before you hit that hard limit, a cell listing 200 names is already unreadable.

My fix for that was reasonable: truncate the preview and add a companion "detail" sheet - one row per (item, consuming pipeline) pair - so nothing was lost. Then I did the math on total rows: at 1,000 pipelines averaging even a modest number of dependencies each, that detail sheet balloons into the same unreadable-sprawl problem I'd just fixed, just relocated.

The actual fix was a change in shape, not size. Every summary sheet - shared repositories, templates, service connections, tasks - stays bounded by **item variety** (how many distinct templates, connections, tasks exist), which grows slowly even at huge scale. The reverse lookup - which pipeline uses what - lives entirely on the Pipelines sheet itself, one row per pipeline (unavoidable, but that row count was never the problem), with dependency columns that stay small because they're bounded by what *one* pipeline actually touches, not by how popular a shared item is across the whole estate.

The corollary that fell out of this almost for free: since I'm already tracking dependency data at the pipeline level, that same structure means an eventual MCP layer consuming the underlying JSON can answer "what does this pipeline depend on" and "what depends on this template" with equal ease - both directions of the same relationship, just organized differently depending on which one you're asking.

## What I'd tell someone building something similar

**Verify your APIs' documented behavior against reality before trusting it.** I assumed the Build Definitions API included last-run data for free. It doesn't, reliably - a separate call to the Builds API was needed. I only found this because real test data (a pipeline with confirmed run history) came back as "Unknown," which shouldn't have been possible if my assumption were correct.

**A bounded, honest gap beats an unbounded, silent one.** Several places in this tool - unresolved service connections, variable groups the scanning credential can't see, standalone vs. templated pipelines - could have been designed to hide uncertainty. Instead they're surfaced explicitly: "this exists but we can't confirm it, verify directly" is more useful to a migration team than a false sense of completeness.

**The thing that looks like scope creep sometimes isn't.** Service connections, variable groups, and Key Vault linkage weren't in my original plan. They came from thinking through what a real migration consultant would actually need to present to leadership - and each one followed the exact same fetch-resolve-aggregate pattern I'd already built for templates, so the marginal cost of adding them was small relative to the analysis they unlocked.

## What's still open

- **NuGet/npm feed identification per pipeline** - deliberately parked. The feed a pipeline actually consumes is frequently configured in a `NuGet.config` file in the source repo, not visible in the pipeline YAML at all - and that pattern is *most* common in exactly the template-driven pipelines this tool is best at analyzing. Real detection coverage here needs real customer data first, not a guess.
- **Recursive template depth resolution** - the tool correctly identifies "this pipeline uses shared templates" but can't measure exact nesting depth from a single file without recursively fetching every referenced template. Documented as a known limitation rather than quietly approximated.
- **An MCP layer** over the JSON manifest, so this becomes something an AI agent can query directly rather than something that only produces static reports.

The repo, with a fuller README covering setup, the full rule catalog, and every limitation stated as plainly as I could manage, is here: [github.com/sreehub16/ado-pipeline-analyser](https://github.com/sreehub16/ado-pipeline-analyser).
