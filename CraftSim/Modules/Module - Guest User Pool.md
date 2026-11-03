---
tags:
  - craftsim
  - login
  - guest
  - robust
aliases:
  - Guest pool
  - Guest User login
---

# Module - Guest User Pool

← Back to [[CraftSim]] | Related: [[Runtime Feature Switches]], [[Avatar, NPC & Region Modules]] (Display Names), [[Build, Configuration & Self-Hosting]]

> Read from `OpenSim/Services/LLLoginService/LLLoginService.cs` and commits `3cb45c127e`, `62f852aceb`, `9e3de293c7`, `5d4ded69a3` (2026-09-01/02) and `cc3a401f8f` (2026-11-03). Not run in this pass. Upstream OpenSim has nothing like it: a login of "Guest User" is just a normal account lookup.

## What it does
A viewer logs in as **`Guest User`** and gets its own throwaway identity, **`Guest User1`, `Guest User2`, ...**, with no registration. Several people can use the same shared credentials at once.

1. The login first authenticates the `Guest User` template account normally (so it must exist, with a known password). That account supplies the e-mail and the **base appearance**.
2. The service walks `Guest User1..N` and takes the first slot that is missing or offline (`GridUserInfo.Online` false or no last region).
   - **Missing**: a new `UserAccount` is created with `DisplayName = "Guest User"`, a fresh inventory skeleton (`CreateUserInventory`; commit `9e3de293c7` fixed it being skipped), and a copy of the template's appearance.
   - **Offline**: reused; `DisplayName` is reset to `Guest User` and `DisplayNameSetAt` to 0, and the inventory skeleton is checked again.
3. **Randomised look** (new slots only): each visual-param slider has a 30 % chance of a random shift of -40..+40 (clamped 0..255), and every face-texture tint moves by up to +-0.2 per channel, so guests are visually distinct.
4. The slot's password is set to a random value and the service re-authenticates, then the rest of the login proceeds as that slot (`firstName`/`lastName`/`account` are swapped).

The pool therefore grows to the peak number of simultaneous guests and never has gaps (creation always scans from 1).

## Daily cleanup (`ReapIdleGuestPoolAccounts`)
Nothing reclaimed the pool in the first version: each simultaneous guest ever seen left a permanent account plus auth, grid-user, avatar and inventory rows. Commit `5d4ded69a3` adds a timer.

- First run 5 minutes after start, then every `GuestPoolCleanupIntervalHours` (default 24; **0 disables**).
- Walks the contiguous pool, then trims **from the tail down**, stopping at the first **online** account or at `GuestPoolKeepWarm` (default 1) accounts from the bottom. It stops instead of skipping, so the pool never gets a gap.
- `DeleteGuestPoolAccount` is a best-effort cascade: inventory items, inventory folders, `ResetAvatar`, grid-user record, auth info, then the account. Failures are logged and the sweep continues.
- Logs `Guest pool cleanup removed N idle pooled guest account(s), M remaining`.
- Only scope `UUID.Zero` is swept.
- It needed new `Delete` methods that did not exist in the service/data layers: `IUserAccountService`, `IAuthenticationService`, `IGridUserService`, `IAuthenticationData`, `IGridUserData`, implemented for MySQL, PGSQL, SQLite and Null stores and the local connectors (with cache invalidation). The HG and remote HTTP connectors return false.

## Configuration
All ini files and where each key goes: [[Configuration Reference (ini files)]] section 4.

Keys live in the Robust/standalone `[LoginService]` section; the commented defaults are in `bin/Robust.ini.example`, `bin/Robust.HG.ini.example` (added 2026-11-03) and `bin/config-include/StandaloneHypergrid.ini`.

| Key | Default | Meaning |
| :--- | :--- | :--- |
| `GuestPoolCleanupIntervalHours` | 24 | Sweep interval; 0 = never |
| `GuestPoolKeepWarm` | 1 | Accounts at the bottom of the pool the sweep keeps |
| `[Features] GuestLogin` | true | Master runtime switch, see below |

## Runtime switch
`guest-login` in [[Runtime Feature Switches]] (`features disable guest-login`, MCP `feature_set`). It is checked before any lookup, so a refused guest costs nothing. The viewer sees "Guest login is currently disabled on this grid. Please log in with your own account." (`LLFailedLoginResponse.GuestLoginDisabledProblem`). Guests already in world stay. With a separate Robust the flag is Robust's own, set from Robust's console.

## Caveats
- Guests are matched by the literal names `Guest` / `User` (case-insensitive).
- Duplicate-presence rules still apply to the chosen slot, not to the template.
- The pool keys are commented in the two Robust example files and in `StandaloneHypergrid.ini`. They are still absent from `Standalone.ini`, `StandaloneCommon.ini.example` and `OpenSimDefaults.ini`, so a standalone setup that does not include `StandaloneHypergrid.ini` has to add them by hand.
- A DB backup taken between sweeps contains the pool accounts; they are disposable.
