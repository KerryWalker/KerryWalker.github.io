---
layout: post
title: Changing Visual Studio Dev Tunnel Access from Private to Public
excerpt: How to update an existing Dev Tunnel's access level via the CLI without losing your tunnel URL.
tags:
  - visual-studio
  - dev-tunnels
---

I was setting up webhook testing for a project and had created a Dev Tunnel in Visual Studio. After configuring the webhook provider with my tunnel URL, I realised the tunnel was set to Private - but I needed it Public for the external service to reach it.

The problem: Visual Studio's UI doesn't show an option to change access level on existing tunnels. Deleting and recreating would change the URL, meaning I'd have to reconfigure all the webhooks again.

## The Solution: Dev Tunnel CLI

The Dev Tunnel CLI can modify existing tunnels without changing their URLs.

### Install the CLI

```powershell
winget install Microsoft.devtunnel
```

### Sign In

The CLI needs authentication with the same Microsoft account used in Visual Studio:

```powershell
devtunnel user login
```

This opens a browser window to complete sign-in.

### Find the Tunnel ID

List all tunnels to get the full ID:

```powershell
devtunnel list
```

Output looks like:

```
Tunnel ID                           Host Connections     Labels                    Ports    Expiration    Description
sunny-river-abc1234.eun1            0                    VisualStudioCreatedT...   1        30 days       My Tunnel 1
cool-breeze-xyz5678.eun1            0                    VisualStudioCreatedT...   1        30 days       My Tunnel 2
```

### Grant Public Access

Use the full tunnel ID with the access command:

```powershell
devtunnel access create cool-breeze-xyz5678.eun1 --anonymous
```

The tunnel is now public, and the URL stays the same.

## Useful CLI Commands

| Command | Description |
|---------|-------------|
| `devtunnel list` | List all tunnels |
| `devtunnel access list <tunnel-id>` | View current access settings |
| `devtunnel access create <tunnel-id> --anonymous` | Make tunnel public |
| `devtunnel access delete <tunnel-id> --anonymous` | Revert to private |

## Security Note

Public tunnels expose your local machine to the internet with no authentication. I switch back to private when I'm done testing - there's no need to leave it open.

## Result

The webhook provider could now reach my local machine, and I didn't have to update any URLs. The whole thing took about two minutes once I knew the CLI existed.
