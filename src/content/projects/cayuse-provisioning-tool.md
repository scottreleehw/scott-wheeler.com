---
title: "Cayuse Provisioning Tool"
description: "Automated user provisioning system for Cayuse research compliance platform — replaces legacy manual scripts with a structured CSV-to-API pipeline."
tags: ["automation", "okta", "python", "higher-ed"]
featured: true
date: 2026-02-15
---

## Overview

A rebuild of legacy provisioning scripts for Cayuse, a research compliance and grants management platform used by the university. The old process involved manual CSV manipulation and fragile scripts. The new tool provides a structured pipeline from enrollment data to Cayuse account provisioning via Okta API integration.

## Problem

- Legacy scripts were brittle, undocumented, and required manual intervention at every step
- No validation or error handling — failures were silent
- Program-to-department mapping was hardcoded and frequently wrong
- No audit trail of what was provisioned or when

## Solution

Built a phased replacement:

1. **Phase 1:** CSV input with validation, transformation, and Cayuse upload — direct replacement of old scripts with proper error handling
2. **Phase 2:** Azure Fabric data warehouse integration — pull enrollment data directly instead of relying on manual CSV exports
3. **Phase 3:** Scheduling and automation with an IT team runbook

## Tech Stack

- Python for the core provisioning logic
- Okta API (SSWS) for identity verification and Employee ID lookup
- Azure Fabric for enrollment data (Phase 2)
- Designed for monthly scheduled execution

## What I Learned

- Writing clear requirements up front (even informal ones) made AI-assisted development dramatically faster — the entire Phase 1 build was done in an afternoon with Claude Code
- Provisioning tools need extensive dry-run and validation modes — you don't want to find out about a bug when it's already created 200 accounts wrong
