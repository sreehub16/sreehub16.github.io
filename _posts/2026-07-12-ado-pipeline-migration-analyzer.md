---
title: Building a Migration Readiness Analyzer for Azure DevOps to GitHub Actions
date: 2026-07-12 20:00:00 -0400
categories: [Azure DevOps, GitHub Actions]
tags: [azure-devops, github-actions, migrations, python, powershell, github]
description: >-
  A Phase 0 discovery tool for Azure DevOps to GitHub Actions migrations -
  what it finds, why it matters for planning a real migration, and the
  design decisions behind it.
---

## Problem statement

A recent engagement involved an estate of several hundred Azure DevOps pipelines, the large majority already YAML-based, with a handful of centralized template repositories that most of those pipelines depended on in some form. Some source repos had already moved to GitHub while their pipeline definitions stayed in Azure DevOps - a normal mid-migration state, not an edge case. The question that came up immediately wasn't "how do we convert these pipelines" - it was "which ones, in what order, and what's actually shared between them."

That's a discovery problem, not a conversion problem, and it doesn't get solved by opening pipelines one at a time. At enterprise scale, some pipelines can convert directly, some need refactoring, and some depend on shared infrastructure - templates, service connections, variable groups - complex enough to need a design conversation before conversion even starts. Without a way to tell those apart up front, a migration plan is really just a guess.

This tool (**[source on GitHub](https://github.com/sreehub16/ADO_Pipeline_Migration_Analyzer)**) was built to answer that question directly: it inventories an Azure DevOps project, analyzes every pipeline, and produces a report a migration team can plan a wave-by-wave conversion against.

## What it finds, and why it matters for planning

**Which shared template repositories carry the most leverage.** Most large ADO estates don't have hundreds of independent pipelines - they have a handful of centralized template repositories consumed by hundreds of pipelines each. Converting those templates once, before touching any consumer pipeline, is almost always the highest-leverage first move in a migration wave plan. The tool surfaces exactly which repos those are and how many pipelines depend on each one.

**Which pipelines are safe to convert directly vs. which need a design conversation.** Every pipeline gets tiered - Direct, Refactor, or Redesign - based on template depth, self-hosted runner dependencies, deployment approval gates, and task compatibility. This turns "500 pipelines" into a plan, not a wall of unknowns.

**Which ADO tasks have no GitHub Actions equivalent yet.** Unmapped tasks are ranked by how many pipelines they affect, so a customer can see that fixing one task mapping unblocks 40 pipelines at once, rather than discovering it pipeline by pipeline.

**Which variable groups need OIDC/federated identity work, not a simple copy.** Key Vault-linked variable groups authenticate differently in GitHub Actions than in Azure Pipelines. The tool distinguishes these from static groups, since that difference changes migration effort meaningfully.

**Which service connections exist, what type each is, and how many pipelines depend on each.** Every connection type re-authenticates differently in GitHub Actions - OIDC federation for Azure, a GitHub App or PAT for cross-repo access - so this is typically its own workstream, separate from pipeline conversion. The inventory also flags connections referenced in pipeline YAML that couldn't be confirmed against the project's registered list, rather than silently dropping them.

**Which pipelines are actually still in use.** Run history determines Active vs. Dormant status - a real, common finding is that 20-40% of an estate hasn't run in months and may not need migrating at all.

## Two technical notes worth knowing if you're building something similar

**Raw and expanded YAML answer different questions.** ADO's Preview API resolves every template reference into a flat file - useful for seeing the real, complete set of tasks a pipeline runs, but it also erases every `template:` key in the process. Detecting template usage has to run against the raw, as-authored file; everything else runs against the expanded one. Picking the wrong source for either check produces a subtly wrong result, not an obvious failure.

**GitHub-sourced pipelines are common and now fully analyzable.** A large share of any real estate will have repos already on GitHub while the pipeline definition stays in ADO - a normal mid-migration state, not an edge case. ADO's own APIs can't reach a GitHub repo directly, but the Preview API resolves it through the pipeline's service connection regardless, and a GitHub PAT closes the remaining gap by fetching the raw file directly from GitHub's Contents API. Neither pipeline type nor repo location should leave a migration blind spot.

## Design principle: bounded reports, not exhaustive ones

The Excel report is deliberately structured so every summary sheet is bounded by *item variety* (how many distinct templates, connections, or tasks exist) rather than by pipeline count or total consumer relationships - which can explode at scale. Per-pipeline dependency detail lives on the pipeline's own row instead of a separate exploding table. The underlying JSON manifest carries the complete, unbounded detail for anything needing exhaustive relational queries - a report and a database serve different jobs, and conflating them produces something too big to be useful at either.

## Known limitations, stated plainly

- True multi-level template nesting depth isn't measured - the tool confirms a pipeline uses shared templates, not how many layers deep.
- NuGet/npm feed identification per pipeline is deliberately out of scope for now - feed config commonly lives in a repo file, not the pipeline YAML, making detection unreliable without more customer data on which pattern dominates.

A few other analysis dimensions have been scoped out for now, not overlooked, and are worth naming for anyone extending this further:

- Pipeline-to-pipeline dependency mapping, to sequence migration waves rather than just score individual pipelines
- Self-hosted agent pool and deployment environment inventories
- Unresolved variable reference detection, and other ADO-implicit runtime dependencies (e.g. `System.AccessToken`) that need a deliberate replacement decision in GitHub Actions
- A dedicated inventory of third-party tool integrations used across build and deployment - SonarQube, ServiceNow, JFrog Artifactory, and similar. These currently surface only indirectly through task mapping, not as their own view of what external systems a pipeline actually touches.

## What's next

An MCP layer over the JSON manifest is the planned next step - the data is already structured for programmatic, bidirectional dependency queries rather than just static reports.
