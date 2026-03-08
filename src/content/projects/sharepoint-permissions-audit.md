---
title: "SharePoint Permissions Audit & Cleanup"
description: "PnP PowerShell scripts for auditing and resetting broken granular permissions across SharePoint Online sites in a university environment."
tags: ["powershell", "sharepoint", "security", "microsoft-365"]
date: 2026-02-01
---

## Overview

SharePoint Online permissions in large organizations tend to drift over time — broken inheritance, one-off sharing links, granular permissions granted ad hoc by non-admins. This project involved building PowerShell scripts using PnP PowerShell to audit and remediate permissions across multiple site collections.

## Problem

- Sites had accumulated years of broken permission inheritance
- No visibility into who had access to what at a granular level
- Sharing links created by end users bypassed intended access controls
- Compliance requirements (FERPA, GLBA) demanded documented access reviews

## Solution

Built a set of PnP PowerShell scripts to:

1. **Audit** — Crawl site collections and generate reports of all unique permissions, broken inheritance, and sharing links
2. **Report** — Export findings to CSV for review with stakeholders
3. **Remediate** — Reset permissions to inherit from parent where appropriate, remove stale sharing links

## Key Takeaways

- SharePoint's permission model is deceptively complex — what looks like "shared with 5 people" can involve dozens of underlying permission entries
- PnP PowerShell is significantly better than the SharePoint Online Management Shell for this kind of work
- Always run audits before remediation — you need a baseline to know what you're changing
- Document everything — when a VP asks why they lost access to a folder, you need receipts
