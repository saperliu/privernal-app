# privernal-app — Privernal Studio Mobile Client (uni-app x)

[简体中文](./README.md) | English

privernal-app is the mobile client of Privernal Studio, built on **uni-app x** (uvue + UTS) — one codebase targeting **Web / Android / iOS / WeChat mini-program** (three-platform compatibility is a hard constraint; see the docs for the platform and packaging matrices). It shares the same backend as the web admin: apps and dynamic forms created on PC **sync to the app automatically, no release needed**. The message hub aggregates system messages and workflow todos (todos jump to the PC approval page; tokens travel via a one-time handshake code, never in the URL). Dynamic forms are **rendered natively** as card lists / details / forms — schemas are fetched live, and unsupported component types fall back to the PC page in a webview. Multi-organization accounts skip the org-picker flow, and passwords travel as SHA-512 digests. Building requires HBuilderX 5.x (uni-app x has no pure-CLI workflow).

## Features

- **Login**: enter a server domain (per-company access by domain, persisted locally); SHA-512 password digest; multi-organization accounts auto-select without the PC org-picker
- **Messages TAB**: system messages (list / unread badge / detail / mark-one-read / mark-all-read) + workflow (todo / done / initiated counts; todos jump to the PC approval page)
- **Workbench TAB**: grid of authorized apps (filtered in real time by the current user's grants, refreshed on onShow + pull-to-refresh); tap an icon to enter the app
- **App detail**: fetches the menu tree in real time; **menu groups become section titles, menu items become icon grids** (4-column colorful icons + names); tap to open the list page
- **Native dynamic-form rendering**: `/builder/{code}` menus render **natively** in the app as mobile card lists / details / forms (schema fetched live; forms built on PC sync without a release); unsupported native component types fall back to the PC page (webview)
- **Dynamic form / feature-page webview track**: dashboards, reports, system admin and the rest are carried by a web-view of the PC page (token hand-off login + full-screen content mode) — every PC capability works out of the box
- **Mine TAB**: organization / department / name display, change password, sign out

## Platform Support & Cross-Platform Constraints

**Hard constraint: all code must run correctly on Web (PC), App and mini-program; the app must stay packagable for iOS and Android.** Coding rules are in [AGENTS.md](AGENTS.md) hard constraint #5, the three-platform difference matrix is in [architecture.md](architecture.md) §8, and the packaging matrix is in [development.md](development.md) §6.

| Target | uni-app x support | Status |
| --- | --- | --- |
| Web (PC browser) | 4.0+ | ✅ full-flow E2E verified (2026-10-23) |
| App Android | 3.9+ | code ready; real-device / cloud packaging pending (TODO P1) |
| App iOS | 4.11+ | code ready; cloud-packaging chain pending (needs a developer account, TODO P1) |
| WeChat mini-program | 4.41+ | pending verification (web-view requires a business-domain whitelist) |
| Alipay mini-program | 5.31+ | on demand |
| HarmonyOS | 4.61+ | on demand |

## Quick Start

Requirement: **HBuilderX 5.x** (uni-app x cannot be created or run with a pure npm CLI).

```bash
CLI=~/Applications/HBuilderX.app/Contents/MacOS/cli

$CLI open                                            # start HBuilderX (if not running)
$CLI project open --path <absolute path of this dir>  # import the project (first time)
$CLI launch web --project <absolute path of this dir> # run in the browser, dev server at http://localhost:5173
```

Login example (local development): domain `http://127.0.0.1:8600`, account `admin / admin123`; in Advanced settings fill the PC frontend dev address (e.g. `http://localhost:9527`); leave it empty for production (same-domain deployment).

> ⚠️ `admin / admin123` is only the first-login seed password for local development. **Change it immediately after first login**;
> before any public deployment, verify it has been changed (password policy and mandatory security items are in the backend repo's `SECURITY.md`).

## Documentation

| Document | Contents |
| --- | --- |
| [AGENTS.md](AGENTS.md) | **Required reading for AI agents**: technical hard constraints, UTS pitfalls, API contracts, commands, red lines |
| [project-overview.md](project-overview.md) | Overall description: background, users, scope boundaries, deployment shapes |
| [architecture.md](architecture.md) | Architecture & data flow: three-platform cooperation, module layout, core sequences, API inventory |
| [DESIGN.md](DESIGN.md) | Visual rules: palette, type scale, spacing, component styles, uni-app x style constraints |
| [user-guide.md](user-guide.md) | User manual: feature walkthroughs and FAQs |
| [development.md](development.md) | Development: environment setup, daily commands, regression checklist, packaging & release, troubleshooting |
| [component-api.md](component-api.md) | Components: utility module APIs, reusable UI fragments, built-in component usage, web-view contract |
| [TODO.md](TODO.md) | Current tasks, priorities and progress |

## Directory Structure

```
privernal-app/
├── pages/
│   ├── login/login.uvue                      Login (domain / account / password / advanced)
│   ├── message/message.uvue                  Messages TAB (system messages | workflow)
│   ├── workbench/workbench.uvue              Workbench TAB (app grid)
│   ├── mine/mine.uvue                        Mine TAB (profile / change password / sign out)
│   ├── app-detail/app-detail.uvue            App feature list (flattened menu tree)
│   ├── webview/webview.uvue                  web-view container (hosts PC pages)
│   └── change-password/change-password.uvue  Change password
├── utils/
│   ├── http.uts                              Request wrapper + platform API paths (unified 401 handling)
│   ├── store.uts                             Local storage (domain / PC address / token / user info)
│   └── sha512.uts                            Pure UTS SHA-512 (login / password-change digests)
├── static/                                   tabBar icons (81px PNG)
├── pages.json                                Routes + tabBar (messages / workbench / mine)
├── manifest.json                             App manifest (uni-app-x)
└── App.uvue / main.uts                       App entry
```

## Current Status

The web target has passed full-flow E2E verification (login → workbench apps → dynamic forms → messages → mine → sign out); verification records and screenshots are in the [development.md](development.md) regression checklist and `../gui-test-screenshots/app-e2e/`. Android/iOS real-device verification is pending (see [TODO.md](TODO.md)).
