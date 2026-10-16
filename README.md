# Raid Aegis

**A live companion dashboard for Raid: Shadow Legends.**

Raid Aegis runs quietly in the background while you play and mirrors your account onto
a real-time web dashboard — champions, artifacts, relics, dungeon teams, boss battles,
Great Hall, and more, always up to date. No screenshots, no spreadsheets, no manual
tracking.

## Features

- **Champions** — your full roster at a glance: level, grade, ascension, masteries, and equipped gear.
- **Artifacts & Relics** — every piece you own, with full stat breakdowns.
- **Account overview** — level, XP progress, resources, shards, and your in-game profile picture and frame.
- **Boss Battles** — current-cycle team power, damage, and boss HP for Clan Boss, Hydra, and Chimera.
- **Dungeon Teams** — the team currently set for each farming dungeon.
- **Great Hall** — every affinity/stat upgrade level at a glance.
- **Inbox** — a browsable view of your in-game mailbox.

All of it updates live — leave the dashboard open in a browser and watch it change as you play.

## Install

1. Download the latest installer from [Releases](https://github.com/TF-Gaming/raid-aegis/releases/latest).
2. Run `RaidAegisSetup-*.exe`. It installs to `C:\Program Files\Raid Aegis` and Windows will ask
   you to approve it (administrator access). If you have an earlier version installed, this
   replaces it and keeps your linked account.
3. Reading live game data needs elevated access, so Raid Aegis also asks Windows for it each time
   it starts — see **Is this safe?** below. Installing under Program Files means no other program
   running as you can swap the app for something else behind that prompt.
4. That's it — Raid Aegis connects itself and your dashboard starts filling in automatically.

Raid Aegis lives in your system tray. Right-click the icon to show the window, pause live
updates, enable auto-start on login, or quit.

## Updating

Raid Aegis checks for new versions automatically and will prompt you when one is available —
just click **Update Now** and it handles the rest. You can also check manually any time from
the tray menu (**Check for Updates**).

## Is this safe?

Yes. Raid Aegis only *reads* data from the game to display it on your dashboard — it never
modifies game memory, automates actions, or interacts with the game in any way. It can't
affect your account, and it isn't a bot, cheat, or hack. Reading data does require Raid Aegis
to run with elevated permissions on Windows (the same requirement PlariumPlay itself has for
launching the game), which is why you'll see that one UAC prompt.

## Requirements

- Windows 10/11
- Raid: Shadow Legends installed via PlariumPlay

## Report an issue

Found a bug, or something look wrong on your dashboard? Please
[open an issue](https://github.com/TF-Gaming/raid-aegis/issues) — include your Raid Aegis
version (shown in the app window title) and a description of what happened.

## Who makes Raid Aegis

Raid Aegis is made by TF Gaming Ltd, a company registered in England and Wales (company number 17519659). It is an independent tool, not affiliated with or endorsed by Plarium Games Ltd.
