---
title: "Optimizing Cloud Costs"
date: 2026-02-14
description: "Practical cloud cost engineering — the levers that actually move the bill, and a workflow for keeping spend honest as you scale."
---

Cloud bills surprise people. Not because the provider is opaque (the invoices are painfully
detailed), but because cost is a *design output*, and most teams only look at it after the design
is locked. This post is the approach I use to keep cloud spend honest: find the real levers,
automate the boring savings, and make cost a first-class part of the architecture review.

## First, see the bill

You can't optimize what you can't attribute. Before touching anything:

- **Turn on tagging** and enforce it. Every resource needs `owner`, `env`, `service`, `cost-center`.
  No tag, no deploy.
- **Use cost allocation reports / billing buckets** per team or service so each team sees *its* number.
- **Set a budget + anomaly alert.** A weekly "this service cost $X" email beats a monthly shock.

Visibility is 40% of the win. Most teams find 10–20% of spend on idle or orphaned resources within
the first week of looking.

## The levers that actually move the number

Not all optimizations are equal. Rank by impact:

1. **Compute shape & utilization.** Oversized instances are the #1 leak. Right-size from real
   CPU/memory metrics, not guesses. A service averaging 12% CPU on a 4-vCPU box is paying for
   potential it doesn't use.
2. **Stop paying for idle time.** Dev/test environments don't need to run at 2am or on weekends.
   Scheduled shutdown + start saves ~65% on non-prod.
3. **Commitment discounts (reserved/instances/savings plans).** If a baseline workload runs 24/7,
   a 1–3 year commitment typically cuts 30–60% vs on-demand. Buy commitments for the *stable*
   baseline only; keep the spiky part on-demand.
4. **Storage lifecycle.** Move cold data down tiers (standard → infrequent → archive). Delete
   orphaned volumes and snapshots. Egress is the sneaky one — watch cross-region and NATgw traffic.
5. **Networking egress.** Data out is expensive and easy to ignore. Cache at the edge, keep
   chatter in-region, and scrutinize any architecture that moves bytes between regions for no
   reason.

## A workflow, not a one-off

Cost optimization decays. The team that did a "cost cleanup sprint" last quarter is over budget
again next quarter unless the discipline sticks. Make it continuous:

- **Architecture review includes a cost estimate.** New service? Estimate monthly run cost before
  approval. "Cheap to build, $4k/mo to run" should be a conscious decision.
- **Right-size on a schedule.** Quarterly utilization review per service.
- **Automate the easy wins.** Auto-stop non-prod on a schedule; auto-delete orphaned volumes after
   N days; alert on untagged resources.
- **Track cost per transaction/per user**, not just total. Total can hide a service that got 3x
  more expensive per request.

## Common mistakes

- **Over-buying commitments** to hit a discount percentage, then not using them. Commit the stable
  baseline; let the rest float.
- **Chasing pennies, missing dollars.** Tuning a Lambda to save $5/mo while a forgotten
  oversized database costs $900/mo. Fix the biggest line item first.
- **No ownership.** When "cost" is nobody's job, it only improves during crises. Give each team its
  number.
- **Ignoring data transfer.** Compute gets the attention; egress and inter-AZ traffic quietly
  dominate.

## Closing thought

Cloud cost engineering isn't about being cheap — it's about *spending deliberately*. The goal is
that every dollar on the bill corresponds to a decision someone made, not a default someone forgot.
Build the visibility, pull the big levers, and bake a cost estimate into the review gate. The
savings compound; the discipline is what makes them stick.
