---
title: "Enterprise App Deployment via MDM"
description: "Mass deployment of a university web app across Windows (Intune) and Mac (Jamf) endpoints using NinjaOne, browser policies, and auto-launch configurations."
tags: ["intune", "jamf", "ninjaone", "mdm", "higher-ed"]
date: 2026-01-20
---

## Overview

Deployed a university web application as a managed experience across all institutional endpoints — Windows via Intune, Mac via Jamf, and cross-platform via NinjaOne. The goal was to make it feel like a native app: desktop shortcut, auto-launch on login, and browser homepage policies pushing users to the portal.

## Approach

### Desktop Shortcut + Auto-Launch (NinjaOne)
- Created a desktop shortcut pointing to the web app URL
- Configured auto-launch on user login
- Tested across Windows 10/11 environments
- Deployed via NinjaOne scripting policies to all managed devices

### Browser Homepage Policies
- **Windows:** Intune configuration profiles pushing Chrome and Edge homepage/startup page policies
- **Mac:** Jamf configuration profiles for Safari and Chrome homepage settings
- Policies ensure the portal loads on every new browser window

## Challenges

- Browser homepage policies behave differently across Chrome, Edge, and Safari — each needs its own policy format
- NinjaOne script execution context (SYSTEM vs user) affects shortcut visibility and auto-launch behavior
- Balancing "always visible" with "not annoying" — the app should be easy to find without being intrusive

## Result

A consistent experience across platforms where the university portal is one click (or zero clicks) away on every managed device.
