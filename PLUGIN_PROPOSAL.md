# Claude Code Plugin Proposal for App Store Connect CLI

> **Status:** Proposal — seeking feedback before implementation
>
> **Author:** Generated via analysis of the full CLI surface, existing ASC Skills, Claude Code plugin ecosystem, and user persona modeling.

## Executive Summary

This proposal describes a **Claude Code plugin** for the App Store Connect CLI (`asc`) that wraps the existing CLI with guided workflows, safety hooks, specialized agents, and domain skills. The plugin does **not** introduce an MCP server — the CLI already outputs minified JSON by default and handles auth, retries, and pagination natively. Instead, the plugin adds what the CLI structurally cannot: conversational setup, automatic safety guards, multi-step orchestration, and team-wide distribution.

The plugin is delivered in **5 phases**, progressing from core daily workflows (Phase 1) to complete CLI surface coverage (Phase 5). At full build-out: **17 slash commands, 26 skills, 4 agents, 3 hooks** — covering 100% of the CLI's 65+ command groups.

---

## Table of Contents

- [Why a Plugin, Not an MCP Server](#why-a-plugin-not-an-mcp-server)
- [Why a Plugin Over Standalone Skills](#why-a-plugin-over-standalone-skills)
- [Plugin Location and Installation](#plugin-location-and-installation)
- [User Personas](#user-personas)
- [Phased Rollout](#phased-rollout)
  - [Phase 1: Foundation](#phase-1-foundation)
  - [Phase 2: App Lifecycle and Content](#phase-2-app-lifecycle-and-content)
  - [Phase 3: Monetization and Pricing](#phase-3-monetization-and-pricing)
  - [Phase 4: Platform-Specific Features](#phase-4-platform-specific-features)
  - [Phase 5: Complete Coverage](#phase-5-complete-coverage)
- [Coverage Progression](#coverage-progression)
- [Delta Matrix: CLI vs Plugin](#delta-matrix-cli-vs-plugin)
- [What the Plugin Adds That the CLI Cannot Do](#what-the-plugin-adds-that-the-cli-cannot-do)
- [Structural Limitations](#structural-limitations)
- [Directory Structure](#directory-structure)
- [README Extension](#readme-extension)
- [Open Questions](#open-questions)

---

## Why a Plugin, Not an MCP Server

| MCP Server Argument | Why It Doesn't Apply Here |
|---|---|
| Structured tool responses | CLI already outputs **minified JSON by default** — designed for AI agents |
| Stateful token caching | JWT generation is sub-millisecond from `.p8` key; ASC API has no session state |
| Tool discovery | `asc --help` at every level is self-documenting |
| Streaming build logs | `--wait` flag already polls and reports processing status |
| Richer integration | The CLI covers **1,210 API paths** across **65+ command groups**. Reimplementing even 10% as MCP tools creates a parallel codebase to maintain |

An MCP server would translate JSON-to-JSON through a server process with high engineering and maintenance cost for zero additional capability.

---

## Why a Plugin Over Standalone Skills

The existing [ASC Skills](https://github.com/rudrankriyam/app-store-connect-cli-skills) (11 skills, installed via `npx add-skill`) provide AI instruction sets. A plugin adds capabilities that standalone skills structurally cannot provide:

| Capability | Standalone Skills | Plugin |
|---|---|---|
| First-time setup guidance | User figures it out alone | `/asc:setup` walks through install, auth, config step-by-step |
| Automatic auth validation | Nothing | `SessionStart` hook catches auth failures before cryptic 401s |
| Destructive operation warnings | Nothing | `PreToolUse` hook warns on unconfirmed `submit`, `expire`, `revoke` |
| One-shot status dashboard | Must chain 4-5 commands | `/asc:status` gives builds + reviews + versions in one view |
| Team-wide distribution | Each person installs skills manually | `.claude/settings.json` auto-enables for entire team |
| Marketplace discoverability | Must know the GitHub URL | Users find it in the Claude Code plugin marketplace |
| Slash commands | Not possible | 17 guided workflow entry points |
| Specialized agents | Not possible | 4 agents for multi-step orchestration with error recovery |

---

## Plugin Location and Installation

### Location

The plugin lives **in the main repo** alongside the CLI source:

```
App-Store-Connect-CLI/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace catalog
├── plugins/
│   └── asc/                      # The plugin
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── skills/
│       ├── commands/
│       ├── agents/
│       └── hooks/
├── internal/                     # Existing CLI source (unchanged)
└── ...
```

### Marketplace manifest (`.claude-plugin/marketplace.json`)

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "app-store-connect-cli",
  "version": "1.0.0",
  "description": "Claude Code plugin for the App Store Connect CLI — guided workflows, safety hooks, and domain skills for iOS/macOS app management.",
  "owner": {
    "name": "Rudrank Riyam",
    "email": "..."
  },
  "plugins": [
    {
      "name": "asc",
      "description": "Guided workflows, safety hooks, and domain skills for the App Store Connect CLI",
      "version": "1.0.0",
      "source": "./plugins/asc",
      "category": "productivity",
      "keywords": ["ios", "macos", "app-store-connect", "testflight", "xcode", "apple"]
    }
  ]
}
```

### User installation

```bash
# Add the marketplace (one-time)
/plugin marketplace add rudrankriyam/App-Store-Connect-CLI

# Install the plugin
/plugin install asc@app-store-connect-cli
```

### Team auto-install (`.claude/settings.json`)

```json
{
  "extraKnownMarketplaces": {
    "app-store-connect-cli": {
      "source": { "source": "github", "repo": "rudrankriyam/App-Store-Connect-CLI" }
    }
  },
  "enabledPlugins": {
    "asc@app-store-connect-cli": true
  }
}
```

---

## User Personas

| Persona | Description | Primary CLI Commands | Plugin Value |
|---|---|---|---|
| **Solo indie dev** | 1-2 apps, does everything themselves | auth, builds, testflight, submit, reviews, metadata | **Highest** — setup, release workflows, review responses |
| **Release manager** | 5+ apps, manages team releases | builds, publish, submit, versions, testflight, profiles | **High** — multi-app status, profile switching, orchestration |
| **CI/CD engineer** | Writes pipelines, not interactive use | builds upload, testflight, submit, auth (env vars) | **Medium** — CI config generation, setup guidance |
| **Monetization manager** | Subscriptions, IAP, pricing, reports | subscriptions, iap, pricing, analytics, finance | **Medium-high** — guided monetization setup (Phase 3) |
| **macOS developer** | Notarization, Developer ID signing | notarization, signing, certificates, profiles | **Medium** — notarization + signing commands |
| **Game developer** | Game Center achievements, leaderboards | gamecenter + everything an indie dev uses | **Medium** — release workflows + Game Center skill (Phase 4) |
| **Enterprise / compliance** | Encryption declarations, EULA, alternative distribution | encryption, eula, alternativedistribution | **Low-medium** — enterprise skill (Phase 5) |

---

## Phased Rollout

### Coverage Strategy

Not every command group needs a slash command. The approach:

- **Slash commands** for guided multi-step workflows (high-value, common operations)
- **Skills** for domain knowledge (teaches Claude a command group — wide coverage, low cost)
- **Agents** for autonomous orchestration (complex multi-step flows with error recovery)
- **Hooks** for automatic safety and validation (cross-cutting)

---

### Phase 1: Foundation

> **Target:** Everyone. The 85% of daily usage.
>
> **Effort:** Medium
>
> **Delivers:** Setup, builds, releases, status, safety hooks

#### Commands (7)

| Command | Wraps CLI Groups | Description |
|---|---|---|
| `/asc:setup` | auth | Guided first-time setup: check if `asc` installed (offer Homebrew if not), walk through API key generation (open browser), run `asc auth login`, verify with `asc auth status --validate`, set default app ID |
| `/asc:profiles` | auth | List all profiles (`asc auth status --verbose`), switch default, add new profile, remove old |
| `/asc:doctor` | auth | Run `asc auth doctor`, interpret results conversationally, offer fixes (`--fix --confirm`) |
| `/asc:status` | apps, builds, versions, submit, reviews, crashes | Multi-app health dashboard: auth status, latest builds with processing state, current App Store version + review status, active TestFlight groups, recent crash count, recent review summary |
| `/asc:release` | publish, builds, versions | Guided release: TestFlight or App Store? Which IPA? Which groups/version? Phased release? Runs `asc publish testflight` or `asc publish appstore` with all flags. Waits for processing. |
| `/asc:find` | apps, builds, testflight, bundleids | Conversational ID resolver: "find my app ID", "what's the group ID for External Testers?", "which build is version 2.1.0?" Chains list commands with filters. |
| `/asc:help` | (all) | Context-aware help: not just `--help` — explains concepts, suggests workflows, shows examples for the user's situation |

#### Skills (6)

| Skill | Covers CLI Groups | Description |
|---|---|---|
| `asc-cli-usage` | (all) | Flags, pagination, output formats, auth patterns, `--help` discovery, env vars, config |
| `asc-release-flow` | publish, builds, versions, submit | End-to-end release workflows, phased release, version management |
| `asc-testflight` | testflight, beta-groups, beta-testers, beta-feedback, beta-crash-logs, beta-notifications, beta-license-agreements, **crashes** (standalone), **feedback** (standalone) | Complete TestFlight domain including standalone crash/feedback commands |
| `asc-build-lifecycle` | builds, buildbundles, buildlocalizations, prerelease | Build processing, uploads, latest build resolution, expiration, cleanup |
| `asc-id-resolver` | apps, builds, versions, testflight groups/testers | Teaches Claude how to chain list commands to resolve human-readable names to API IDs |
| `asc-xcode-build` | (xcodebuild, not the CLI) | Building, archiving, and exporting iOS/macOS apps before upload |

#### Agents (2)

| Agent | Triggers | Description |
|---|---|---|
| **release-orchestrator** | "release my app", "deploy to TestFlight", "submit to App Store" | Multi-step release with error recovery: resolve app ID, find/upload build, check processing, resolve group IDs, distribute, set notes, submit |
| **build-monitor** | "is my build done", "check build status", "what's happening with my builds" | Checks build processing status, waits if needed, reports issues across apps |

#### Hooks (3)

| Hook | Event | Description |
|---|---|---|
| Auth validation | `SessionStart` | Silently runs `asc auth status`. If auth fails, warns user before they encounter cryptic errors. |
| Destructive guard | `PreToolUse` (Bash) | Matches `asc submit create`, `asc builds expire`, `asc certificates revoke`, `asc builds expire-all` without `--confirm`. Warns before execution. |
| App ID reminder | `PreToolUse` (Bash) | Matches `asc` commands that typically need `--app` but don't have it and `ASC_APP_ID` is not set. Reminds user to set a default. |

#### Phase 1 Persona Coverage

| Persona | Coverage |
|---|---|
| Solo indie dev | **85%** of daily workflows |
| Release manager | **80%** |
| CI/CD engineer | **40%** (skills help write scripts) |
| Monetization manager | **15%** |
| macOS developer | **50%** |
| Game developer | **55%** |

---

### Phase 2: App Lifecycle and Content

> **Target:** Metadata managers, macOS devs, review responders
>
> **Effort:** Medium
>
> **Delivers:** Metadata sync, signing, notarization, reviews, submission pre-flight

#### Commands (+5 = 12 total)

| Command | Wraps CLI Groups | Description |
|---|---|---|
| `/asc:metadata` | localizations, migrate, betaapplocalizations, betabuildlocalizations, app_events | Pull/edit/push app metadata, validate character limits, Fastlane format migration |
| `/asc:reviews` | reviews | Ratings summary, recent reviews (filterable by stars/territory), draft + post responses via `asc reviews respond` |
| `/asc:submit` | submit, versions, assets, encryption | Guided submission with pre-flight checks: metadata completeness, screenshot coverage, encryption declarations, build validity. Then submit with `--confirm`. |
| `/asc:signing` | signing, certificates, profiles, bundleids | Guided cert creation, profile setup, capability management, `asc signing fetch` |
| `/asc:notarize` | notarization | Submit for notarization, wait for completion, check status, fetch developer logs on failure |

#### Skills (+6 = 12 total)

| Skill | Covers CLI Groups |
|---|---|
| `asc-metadata-sync` | localizations, betaapplocalizations, betabuildlocalizations, migrate, app_events, productpages |
| `asc-signing-setup` | signing, certificates, profiles, bundleids, devices |
| `asc-notarization` | notarization |
| `asc-submission-health` | submit, versions, assets, encryption, agerating, eula, routingcoverage |
| `asc-assets` | assets, routingcoverage, backgroundassets |
| `asc-performance` | performance (metrics, diagnostics, download) |

#### Agents (+1 = 3 total)

| Agent | Triggers | Description |
|---|---|---|
| **review-responder** | "respond to reviews", "handle negative reviews", "draft review responses" | Fetches reviews, drafts responses, lets user approve before posting |

#### Phase 2 Persona Coverage

| Persona | Coverage |
|---|---|
| Solo indie dev | **93%** |
| Release manager | **90%** |
| CI/CD engineer | **50%** |
| Monetization manager | **15%** |
| macOS developer | **90%** |
| Game developer | **60%** |

---

### Phase 3: Monetization and Pricing

> **Target:** Paid app devs, monetization managers, growth teams
>
> **Effort:** Medium-high (subscription/IAP flows are complex)
>
> **Delivers:** Subscription management, IAP, pricing, analytics, finance reports

#### Commands (+3 = 15 total)

| Command | Wraps CLI Groups | Description |
|---|---|---|
| `/asc:subscriptions` | subscriptions, offercodes, winbackoffers, promotedpurchases | Guided subscription group setup, subscription creation, pricing, offer codes, promotional offers, introductory offers, win-back offers, promoted purchases |
| `/asc:iap` | iap | Guided IAP creation, pricing, localizations, availability, submit for review |
| `/asc:reports` | analytics, finance | Download sales/finance reports with guided date/vendor/region selection |

#### Skills (+4 = 16 total)

| Skill | Covers CLI Groups |
|---|---|
| `asc-pricing` | pricing, subscriptions (pricing subset), iap (pricing subset) — territory-specific pricing, PPP strategies, equalization |
| `asc-subscriptions` | subscriptions, offercodes, winbackoffers, promotedpurchases — groups, offers, availability, grace periods |
| `asc-iap` | iap — consumables, non-consumables, localizations, images, price schedules |
| `asc-analytics-finance` | analytics, finance — sales reports, financial reports, analytics requests, instances, segments |

#### Phase 3 Persona Coverage

| Persona | Coverage |
|---|---|
| Solo indie dev | **97%** |
| Release manager | **95%** |
| CI/CD engineer | **55%** |
| Monetization manager | **90%** |
| macOS developer | **90%** |
| Game developer | **65%** |

---

### Phase 4: Platform-Specific Features

> **Target:** Game devs, Xcode Cloud users, App Clip devs, CI engineers
>
> **Effort:** Medium
>
> **Delivers:** Game Center, Xcode Cloud, App Clips, webhooks, sandbox, CI config

#### Commands (+1 = 16 total)

| Command | Wraps CLI Groups | Description |
|---|---|---|
| `/asc:ci` | xcodecloud, webhooks | Xcode Cloud workflow trigger/status, webhook setup. Also generates CI pipeline env var configs for GitHub Actions / Fastlane. |

#### Skills (+5 = 21 total)

| Skill | Covers CLI Groups |
|---|---|
| `asc-gamecenter` | gamecenter — achievements, leaderboards, leaderboard-sets, challenges, activities, groups, matchmaking, details, app-versions |
| `asc-xcode-cloud` | xcodecloud — run, status, workflows, build-runs, actions, artifacts, test-results, issues, scm, products |
| `asc-appclips` | appclips — default-experiences, advanced-experiences, header-images, invocations, domain-status, review-details |
| `asc-webhooks` | webhooks — list, get, create, update, delete, deliveries, ping |
| `asc-sandbox` | sandbox — list, get, update, clear-history |

#### Agents (+1 = 4 total)

| Agent | Triggers | Description |
|---|---|---|
| **ci-config-generator** | "set up CI for my app", "configure GitHub Actions for ASC", "write a Fastlane lane" | Generates CI pipeline YAML with correct `asc` commands, env vars, auth setup |

#### Phase 4 Persona Coverage

| Persona | Coverage |
|---|---|
| Solo indie dev | **97%** |
| Release manager | **97%** |
| CI/CD engineer | **80%** |
| Monetization manager | **93%** |
| macOS developer | **90%** |
| Game developer | **93%** |

---

### Phase 5: Complete Coverage

> **Target:** Enterprise, compliance, edge cases, completeness
>
> **Effort:** Low (mostly skills for rarely-used features)
>
> **Delivers:** 100% command group coverage

#### Commands (+1 = 17 total)

| Command | Wraps CLI Groups | Description |
|---|---|---|
| `/asc:config` | (config.json, env vars) | Configure all defaults: app ID, vendor number, output format, timeouts, retry settings. Writes to config.json or generates shell exports for `.zshrc`/`.bashrc`. |

#### Skills (+5 = 26 total)

| Skill | Covers CLI Groups |
|---|---|
| `asc-devices` | devices — list, get, register, update, local-udid |
| `asc-users` | users, actors — list, get, update, delete, invite, invitations, visible-apps |
| `asc-enterprise` | encryption, eula, alternativedistribution, marketplace, agreements, preorders, nominations — EU compliance, alternative distribution, enterprise features |
| `asc-app-setup` | categories, agerating, accessibility, productpages, app_events — initial app configuration, custom product pages, A/B test experiments |
| `asc-identifiers` | merchantids, passtypeids, androidiosmapping — Wallet pass IDs, merchant IDs, cross-platform mapping |

#### Phase 5 Persona Coverage

| Persona | Coverage |
|---|---|
| Solo indie dev | **99%** |
| Release manager | **99%** |
| CI/CD engineer | **85%** |
| Monetization manager | **97%** |
| macOS developer | **95%** |
| Game developer | **97%** |
| Enterprise / compliance | **95%** |

---

## Coverage Progression

```
Phase 1: Foundation          ████████████████████░░░░░  55% groups, 85% daily use
  7 commands, 6 skills, 2 agents, 3 hooks
  Personas: Everyone — setup, builds, releases, status

Phase 2: App Lifecycle       ███████████████████████░░  75% groups, 93% daily use
  +5 commands, +6 skills, +1 agent
  Personas: +macOS devs, reviewers, metadata managers

Phase 3: Monetization        ████████████████████████░  88% groups, 97% daily use
  +3 commands, +4 skills
  Personas: +Paid app devs, growth teams

Phase 4: Platform Features   █████████████████████████  96% groups, 99% daily use
  +1 command, +5 skills, +1 agent
  Personas: +Game devs, CI engineers, App Clip devs

Phase 5: Long Tail           █████████████████████████  100% groups
  +1 command, +5 skills
  Personas: +Enterprise, compliance, edge cases
```

### Final Totals

| Component | Count |
|---|---|
| Slash commands | 17 |
| Skills | 26 |
| Agents | 4 |
| Hooks | 3 |
| Total markdown files | ~50 |
| Shell scripts (hooks) | ~3-4 |
| Config files | plugin.json, marketplace.json, hooks.json |

### Phase Investment vs Return

| Phase | New Files | Personas Unlocked | Marginal Value |
|---|---|---|---|
| Phase 1 | ~18 | Everyone (core workflows) | **Highest** — setup, safety, release flows |
| Phase 2 | ~13 | macOS devs, content managers | **High** — app lifecycle completeness |
| Phase 3 | ~10 | Monetization / growth | **Medium-high** — unlocks paid-app persona |
| Phase 4 | ~8 | Game devs, CI engineers | **Medium** — niche but important personas |
| Phase 5 | ~7 | Enterprise, edge cases | **Low** — completeness, not critical |

---

## Delta Matrix: CLI vs Plugin

### What the Plugin Covers per CLI Command Group

| CLI Command Group | Phase | Coverage Type | Notes |
|---|---|---|---|
| auth | 1 | Command + Hook | `/asc:setup`, `/asc:profiles`, `/asc:doctor`, SessionStart hook |
| apps | 1 | Command | `/asc:status`, `/asc:find` |
| builds | 1 | Command + Agent | `/asc:status`, `/asc:release`, build-monitor agent |
| publish | 1 | Command + Agent | `/asc:release`, release-orchestrator agent |
| testflight | 1 | Skill + Command | `asc-testflight` skill, `/asc:release` |
| crashes | 1 | Skill | `asc-testflight` skill (standalone command coverage) |
| feedback | 1 | Skill | `asc-testflight` skill (standalone command coverage) |
| versions | 1 | Command (implicit) | Covered by `/asc:release`, `/asc:status` |
| prerelease | 1 | Skill | `asc-build-lifecycle` skill |
| buildbundles | 1 | Skill | `asc-build-lifecycle` skill |
| buildlocalizations | 1 | Skill | `asc-build-lifecycle` skill |
| bundleids | 2 | Command + Skill | `/asc:signing`, `asc-signing-setup` skill |
| certificates | 2 | Command + Skill | `/asc:signing`, `asc-signing-setup` skill |
| profiles | 2 | Command + Skill | `/asc:signing`, `asc-signing-setup` skill |
| signing | 2 | Command + Skill | `/asc:signing`, `asc-signing-setup` skill |
| notarization | 2 | Command + Skill | `/asc:notarize`, `asc-notarization` skill |
| localizations | 2 | Command + Skill | `/asc:metadata`, `asc-metadata-sync` skill |
| migrate | 2 | Command | `/asc:metadata` |
| betaapplocalizations | 2 | Skill | `asc-metadata-sync` skill |
| betabuildlocalizations | 2 | Skill | `asc-metadata-sync` skill |
| reviews | 2 | Command + Agent | `/asc:reviews`, review-responder agent |
| submit | 2 | Command + Hook | `/asc:submit`, destructive guard hook |
| assets | 2 | Skill | `asc-assets` skill |
| backgroundassets | 2 | Skill | `asc-assets` skill |
| routingcoverage | 2 | Skill | `asc-assets` skill |
| performance | 2 | Skill | `asc-performance` skill |
| app_events | 2 | Skill | `asc-metadata-sync` skill |
| encryption | 2 | Skill | `asc-submission-health` skill |
| agerating | 2 | Skill | `asc-submission-health` skill |
| eula | 2 | Skill | `asc-submission-health` skill |
| subscriptions | 3 | Command + Skill | `/asc:subscriptions`, `asc-subscriptions` skill |
| offercodes | 3 | Command + Skill | `/asc:subscriptions`, `asc-subscriptions` skill |
| winbackoffers | 3 | Command + Skill | `/asc:subscriptions`, `asc-subscriptions` skill |
| promotedpurchases | 3 | Command + Skill | `/asc:subscriptions`, `asc-subscriptions` skill |
| iap | 3 | Command + Skill | `/asc:iap`, `asc-iap` skill |
| pricing | 3 | Skill | `asc-pricing` skill |
| analytics | 3 | Command + Skill | `/asc:reports`, `asc-analytics-finance` skill |
| finance | 3 | Command + Skill | `/asc:reports`, `asc-analytics-finance` skill |
| gamecenter | 4 | Skill | `asc-gamecenter` skill |
| xcodecloud | 4 | Command + Skill | `/asc:ci`, `asc-xcode-cloud` skill |
| appclips | 4 | Skill | `asc-appclips` skill |
| webhooks | 4 | Command + Skill | `/asc:ci`, `asc-webhooks` skill |
| sandbox | 4 | Skill | `asc-sandbox` skill |
| devices | 5 | Skill | `asc-devices` skill |
| users | 5 | Skill | `asc-users` skill |
| actors | 5 | Skill | `asc-users` skill |
| categories | 5 | Skill | `asc-app-setup` skill |
| accessibility | 5 | Skill | `asc-app-setup` skill |
| productpages | 5 | Skill | `asc-app-setup` skill |
| alternativedistribution | 5 | Skill | `asc-enterprise` skill |
| marketplace | 5 | Skill | `asc-enterprise` skill |
| agreements | 5 | Skill | `asc-enterprise` skill |
| preorders | 5 | Skill | `asc-enterprise` skill |
| nominations | 5 | Skill | `asc-enterprise` skill |
| merchantids | 5 | Skill | `asc-identifiers` skill |
| passtypeids | 5 | Skill | `asc-identifiers` skill |
| androidiosmapping | 5 | Skill | `asc-identifiers` skill |
| notify | 5 | Skill | Referenced in `asc-cli-usage` skill |
| version | — | Not covered | Trivial — Claude runs `asc version` directly |
| completion | — | Not covered | Shell completion setup — not relevant in Claude Code |
| install | — | Not covered | Meta command for ASC Skills — plugin replaces this |

---

## What the Plugin Adds That the CLI Cannot Do

These are capabilities that exist **only in the plugin** with no CLI equivalent:

| Plugin Capability | Description |
|---|---|
| **Guided first-time setup** | `/asc:setup` walks through install, API key generation (opens browser), auth login, validation, default config — conversationally. The CLI can run `auth login` but cannot guide a new user through the prerequisite steps. |
| **Automatic auth validation** | `SessionStart` hook silently verifies auth before work begins. The CLI has no concept of "session start." |
| **Destructive operation guard** | `PreToolUse` hook catches `asc submit create`, `asc builds expire`, `asc certificates revoke` without `--confirm` and warns. The CLI accepts the missing flag silently (it just errors). |
| **App ID reminder** | `PreToolUse` hook detects missing `--app` when `ASC_APP_ID` is unset. The CLI errors after the fact. |
| **Multi-command dashboard** | `/asc:status` combines auth, builds, versions, reviews, crashes into one view. The CLI requires 4-5 separate commands. |
| **Conversational ID resolution** | `/asc:find` asks "what are you looking for?" and chains the right list commands. The CLI requires knowing which command and which flags to use. |
| **Review response drafting** | review-responder agent reads reviews, drafts responses, lets user approve, then posts. The CLI can post (`asc reviews respond`) but cannot draft. |
| **Error recovery in workflows** | release-orchestrator agent diagnoses failures and tries alternatives. The CLI's `publish` commands either succeed or fail. |
| **Team auto-install** | `.claude/settings.json` ensures everyone gets the same tooling. No CLI equivalent. |
| **CI config generation** | ci-config-generator agent writes GitHub Actions YAML / Fastlane lanes with correct `asc` commands. The CLI runs in CI but cannot write CI configs. |
| **Context-aware help** | `/asc:help` understands what the user is trying to accomplish. `--help` shows flags. |

---

## Structural Limitations

Even at 100% command group coverage, the plugin has inherent limitations:

1. **Flag-level depth.** The CLI has ~200+ subcommands with ~500+ unique flags. Skills teach patterns and domains, not every flag. Claude falls back to `asc <command> --help` for specific flags. This is by design — skills that enumerated every flag would be massive and brittle (breaking every time the CLI adds a flag).

2. **The plugin requires the CLI.** Every command and agent calls `asc` under the hood. The plugin adds no direct API access. Users must have `asc` installed (though `/asc:setup` handles this).

3. **CI pipelines don't use the plugin.** CI/CD pipelines call `asc` directly. The plugin helps _write_ CI scripts (via the ci-config-generator agent), but doesn't _run_ in pipelines.

4. **CLI utility commands are not covered.** `asc version`, `asc completion`, and `asc install skills` have no plugin equivalent and don't need one.

---

## Directory Structure

```
plugins/asc/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── setup.md
│   ├── profiles.md
│   ├── doctor.md
│   ├── status.md
│   ├── release.md
│   ├── find.md
│   ├── help.md
│   ├── metadata.md              # Phase 2
│   ├── reviews.md               # Phase 2
│   ├── submit.md                # Phase 2
│   ├── signing.md               # Phase 2
│   ├── notarize.md              # Phase 2
│   ├── subscriptions.md         # Phase 3
│   ├── iap.md                   # Phase 3
│   ├── reports.md               # Phase 3
│   ├── ci.md                    # Phase 4
│   └── config.md                # Phase 5
├── skills/
│   ├── asc-cli-usage/
│   │   └── SKILL.md
│   ├── asc-release-flow/
│   │   └── SKILL.md
│   ├── asc-testflight/
│   │   └── SKILL.md
│   ├── asc-build-lifecycle/
│   │   └── SKILL.md
│   ├── asc-id-resolver/
│   │   └── SKILL.md
│   ├── asc-xcode-build/
│   │   └── SKILL.md
│   ├── asc-metadata-sync/       # Phase 2
│   │   └── SKILL.md
│   ├── asc-signing-setup/       # Phase 2
│   │   └── SKILL.md
│   ├── asc-notarization/        # Phase 2
│   │   └── SKILL.md
│   ├── asc-submission-health/   # Phase 2
│   │   └── SKILL.md
│   ├── asc-assets/              # Phase 2
│   │   └── SKILL.md
│   ├── asc-performance/         # Phase 2
│   │   └── SKILL.md
│   ├── asc-pricing/             # Phase 3
│   │   └── SKILL.md
│   ├── asc-subscriptions/       # Phase 3
│   │   └── SKILL.md
│   ├── asc-iap/                 # Phase 3
│   │   └── SKILL.md
│   ├── asc-analytics-finance/   # Phase 3
│   │   └── SKILL.md
│   ├── asc-gamecenter/          # Phase 4
│   │   └── SKILL.md
│   ├── asc-xcode-cloud/        # Phase 4
│   │   └── SKILL.md
│   ├── asc-appclips/            # Phase 4
│   │   └── SKILL.md
│   ├── asc-webhooks/            # Phase 4
│   │   └── SKILL.md
│   ├── asc-sandbox/             # Phase 4
│   │   └── SKILL.md
│   ├── asc-devices/             # Phase 5
│   │   └── SKILL.md
│   ├── asc-users/               # Phase 5
│   │   └── SKILL.md
│   ├── asc-enterprise/          # Phase 5
│   │   └── SKILL.md
│   ├── asc-app-setup/           # Phase 5
│   │   └── SKILL.md
│   └── asc-identifiers/         # Phase 5
│       └── SKILL.md
├── agents/
│   ├── release-orchestrator.md
│   ├── build-monitor.md
│   ├── review-responder.md      # Phase 2
│   └── ci-config-generator.md   # Phase 4
├── hooks/
│   └── hooks.json
├── scripts/
│   ├── check-auth.sh
│   ├── check-confirm-flag.sh
│   └── check-app-id.sh
└── README.md
```

---

## README Extension

Add the following section to the main `README.md` after the existing "ASC Skills" section:

```markdown
## Claude Code Plugin

Use `asc` directly from [Claude Code](https://code.claude.com) with guided workflows, safety hooks, and intelligent agents.

### Install

```bash
# Add the marketplace (one-time)
/plugin marketplace add rudrankriyam/App-Store-Connect-CLI

# Install the plugin
/plugin install asc@app-store-connect-cli
```

### Commands

| Command | Description |
|---------|-------------|
| `/asc:setup` | Guided first-time setup (install, auth, configure) |
| `/asc:status` | App health dashboard (builds, reviews, TestFlight, versions) |
| `/asc:release` | Guided release to TestFlight or App Store |
| `/asc:submit` | App Store submission with pre-flight checks |
| `/asc:reviews` | Review summary with response drafting |
| `/asc:metadata` | Sync app metadata and localizations |
| `/asc:signing` | Certificate and profile setup |
| `/asc:notarize` | macOS notarization workflow |
| `/asc:subscriptions` | Subscription and offer management |
| `/asc:iap` | In-app purchase setup |
| `/asc:reports` | Sales, finance, and analytics reports |
| `/asc:ci` | Xcode Cloud and CI/CD configuration |
| `/asc:profiles` | Manage auth profiles |
| `/asc:doctor` | Diagnose and fix auth issues |
| `/asc:config` | Configure defaults (app ID, output format, timeouts) |
| `/asc:find` | Resolve app/build/group IDs by name |
| `/asc:help` | Context-aware help and examples |

### Team Setup

Auto-enable for your team via `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "app-store-connect-cli": {
      "source": { "source": "github", "repo": "rudrankriyam/App-Store-Connect-CLI" }
    }
  },
  "enabledPlugins": { "asc@app-store-connect-cli": true }
}
```

Includes 26 domain skills covering the full CLI surface. See [plugins/asc/README.md](plugins/asc/README.md) for details.
```

---

## Open Questions

1. **Skill subsumption.** Should this plugin replace the existing [app-store-connect-cli-skills](https://github.com/rudrankriyam/app-store-connect-cli-skills) repo, or coexist? Recommendation: subsume — one source of truth, one install.

2. **Plugin name.** `asc` is concise but may conflict. Alternatives: `asc-cli`, `app-store-connect`. The slash command prefix (`/asc:`) works well with all three.

3. **Phase 1 scope.** Is Phase 1 the right MVP, or should Phase 2 components (especially `/asc:submit` and `/asc:reviews`) be pulled into Phase 1?

4. **Existing skill content.** Should the 11 existing SKILL.md files be used as-is, adapted, or rewritten? They're MIT-licensed so can be incorporated.

5. **Author attribution.** The plugin would live in `rudrankriyam/App-Store-Connect-CLI`. Should the plugin author match the repo author, or list contributors separately?
