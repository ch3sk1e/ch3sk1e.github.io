---
title: "TokenGrabber: A Multi-Resource Token Store for Entra ID Attack Paths"
date: 2026-09-30
draft: false
summary: "How losing track of which identity had which token in which terminal during CARTP lab work turned into a PowerShell tool."
tags: ["Entra ID", "Azure", "red team", "tooling", "PowerShell"]
---

TokenGrabber was born out of CARTP lab work, not a live engagement, so I want to be upfront about that: it hasn't been battle-tested on a real client yet. What it *was* tested on is exam day, where it ended up being the primary tool I reached for.

## The problem that started it

Access tokens in Entra ID expire after about an hour, which is fine on its own, but in the middle of lab work I found myself juggling multiple identities, each with their own sets of access tokens, each scoped to different resources (msgraph, ARM, vault, storage, whatever was required at the time), each using different `client-ids`... Needless to say, I kept losing track of which identity I had a token for, in which terminal, and for which scope. Re-authenticating every time I lost the thread got old fast.

What I wanted was somewhere central to stash all the loot and refresh it with one command whenever I needed it. Enter TokenGrabber!

## Two modules, kept separate on purpose

- **`Get-MultiToken.psm1`** - seeds an identity (credential login, device code, client credentials, or importing a refresh token minted elsewhere) and keeps a persistent access-token store so you're not re-authenticating every time.
- **`Get-PRTStore.psm1`** - a separate store just for clear Primary Refresh Tokens and session keys. A PRT is a different class of secret entirely - it can mint tokens for *any* client via `roadtx`, not just one resource - so it gets kept apart from ordinary access tokens rather than living in the same file.

```powershell
# Device code flow - use this when MFA is actually enforced
Get-MultiToken -DeviceCode -ClientId azurecli -Resources msgraph,arm -TenantId 'contoso.onmicrosoft.com'

# Resume later - no fresh credentials needed, just a store lookup
Get-MultiToken -Identity 'jdoe@contoso.onmicrosoft.com' -Resources vault
```

That second command is really the whole point of building this. Once an identity's seeded, getting a token for a different resource later is a one-liner: a silent refresh-token redemption instead of a fresh interactive login, assuming Conditional Access doesn't have other ideas.

## The client-ID rabbit hole

While building this out I actually noticed something that took me a bit to properly understand. So I  mint a token for the same identity and the same resource, but change the client ID, and the scopes you get back change too. Turns out this comes down to which client applications are pre-authorized (pre-consented) for that resource in Entra ID. Take `azurecli` and `azurepowershell` for instance, they are both part of Microsoft's FOCI family and are broadly pre-authorized across nearly all first-party Azure resources, which is why they tend to hand back wider scopes than something narrower like Intune Company Portal. The ceiling is still whatever the identity is actually entitled to as a client can't get you more access than the identity has, but starting from a client with a wider pre-authorized scope means you're more likely to see the full extent of that access without extra consent prompts getting in the way.

TokenGrabber tracks this per-client in `known-clients.json`, and deliberately never silently swaps your `-ClientId` based on the resource you asked for. You get a warning if a pairing looks unusual, but the client you asked for is the client you get. I didn't really want the tool quietly changing what I was authenticating as behind the scenes.

## What I'm actually proud of

I'm not a software developer. I'd describe myself more as a code "extender", a Frankenstein maker... pull from here, push it there, shove it all together into something that does the job, whilst learning as I go. TokenGrabber leans heavily on work from security researchers who've done the real hard yards on this (credited below), and with coding agents becoming genuinely useful, I figured I'd combine what I actually needed operationally with the speed of an LLM to help me build it. In all honesty, I'm still learning to accept this workflow as "operational efficiency" rather than "cheating" or "AI Slop" :D ... all in due time ...

Jokes aside, at the end of the day, I'm proud of having designed something that solved a real problem I had, and it ended up being **THE** tool I actually reached for on exam day, not something that sounded good in theory and then sat unused.

## Credit where it's due

The credential-based login flow is adapted from [`Get-AccessToken.ps1`](https://github.com/Gerenios/AADInternals) in **AADInternals** by Dr. Nestori Syynimaa, and the refresh-token-to-multi-resource redemption pattern was informed by [**TokenTacticsV2**](https://github.com/f-bader/TokenTacticsV2). TokenGrabber is a from-scratch reimplementation built around a persistent local store and a `-Resources`/`-ClientId` multi-audience model, not a copy of either project - but the underlying OAuth mechanics owe credit to both.

## What's next

I'm currently looking for new opportunities as a Red Team operator, and I'm using this stretch to study, dig deeper into things like this, and document whatever turns out interesting on this blog. If any of it ends up helping someone else along their own path, I'd be genuinely glad to have been part of that.

Code's on [GitHub](https://github.com/ch3sk1e/TokenGrabber). For use in authorized security assessments and lab environments only.
