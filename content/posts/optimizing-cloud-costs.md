---
title: "Optimizing Cloud Costs"
date: 2026-02-14
description: "Practical cloud cost engineering — the levers that actually move the bill, and a workflow for keeping spend honest as you scale."
---

Cloud bills surprise people. Not because the provider is opaque, the invoices are painfully detailed, but
because cost is an output of design, and most teams only look at it after the design is locked. This is the
approach I use to keep cloud spend honest: find the real levers, automate the boring savings, and make cost
a normal part of the architecture review.

## First, see the bill

You can't optimize what you can't attribute. Before touching anything:

- Turn on tagging and enforce it. Every resource needs owner, env, service, cost-center. No tag, no deploy.
- Use cost allocation reports per team or service so each team sees its own number.
- Set a budget and an anomaly alert. A weekly "this service cost $X" email beats a monthly shock.

Visibility is most of the win. Most teams find 10 to 20 percent of spend on idle or orphaned resources in the
first week of actually looking.

## The levers that actually move the number

Not all optimizations are equal, so rank by impact.

Compute shape and utilization is the biggest leak. Oversized instances are the number one culprit. Right-size
from real CPU and memory metrics, not guesses. A service averaging 12 percent CPU on a 4-vCPU box is paying
for headroom it never uses.

Stop paying for idle time. Dev and test environments don't need to run at 2am or on weekends. Scheduled
shutdown and start saves around 65 percent on non-prod.

Commitment discounts, reserved instances, savings plans, matter if a baseline workload runs 24/7. A one to
three year commitment typically cuts 30 to 60 percent versus on-demand. Buy commitments for the stable
baseline only and keep the spiky part on-demand.

Storage lifecycle. Move cold data down tiers, standard to infrequent to archive. Delete orphaned volumes and
snapshots. Egress is the sneaky one, so watch cross-region and NAT gateway traffic.

Networking egress. Data out is expensive and easy to ignore. Cache at the edge, keep chatter in-region, and
scrutinize any architecture that moves bytes between regions for no reason.

## A workflow, not a one-off

Cost optimization decays. The team that ran a "cost cleanup sprint" last quarter is over budget again next
quarter unless the discipline sticks. Make it continuous.

Architecture review includes a cost estimate. New service? Estimate the monthly run cost before approval.
"Cheap to build, four grand a month to run" should be a conscious decision.

Right-size on a schedule, a quarterly utilization review per service.

Automate the easy wins. Auto-stop non-prod on a schedule, auto-delete orphaned volumes after N days, alert on
untagged resources.

Track cost per transaction or per user, not just total. A total can hide a service that got three times more
expensive per request.

## Common mistakes

Over-buying commitments to hit a discount percentage, then not using them. Commit the stable baseline and let
the rest float.

Chasing pennies, missing dollars. Tuning a Lambda to save five bucks a month while a forgotten oversized
database costs nine hundred. Fix the biggest line item first.

No ownership. When cost is nobody's job, it only improves during crises. Give each team its number.

Ignoring data transfer. Compute gets the attention, while egress and inter-AZ traffic quietly dominate.

## The point

Cloud cost engineering isn't about being cheap, it's about spending deliberately. The goal is that every
dollar on the bill corresponds to a decision someone made, not a default someone forgot. Build the
visibility, pull the big levers, and bake a cost estimate into the review gate. The savings compound, and the
discipline is what makes them stick.
