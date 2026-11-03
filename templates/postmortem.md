---
title: "<Incident title>"
date: YYYY-MM-DD
severity: SEV?        # SEV1 (outage) .. SEV4 (minor)
services: []
status: Draft         # Draft | Reviewed | Actions complete
---

# Postmortem: <title>

## Summary

<2–3 sentences: what happened, impact, duration.>

## Impact

<Users/orders/requests affected; data loss; SLA breach.>

## Timeline (UTC)

| Time | Event |
|------|-------|

## Root cause

<Use systematic-debugging evidence; ask "why" until it's a system cause.>

## What went well / what didn't

## Action items

| Action | Type (prevent/detect/mitigate) | Owner | Due | Link |
|--------|-------------------------------|-------|-----|------|

## Memory updates

<Standards, ADRs, or lessons updated as a result.>
