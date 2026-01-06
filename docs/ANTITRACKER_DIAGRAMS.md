# AntiTracker Visual Architecture

This document provides visual diagrams to help understand how IVPN's AntiTracker (ad and tracker blocking) feature works.

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          IVPN Desktop Application                        │
│                                                                           │
│  ┌──────────────────────────┐         ┌───────────────────────────┐    │
│  │      UI (Electron)        │         │     Daemon (Go)           │    │
│  │                           │   IPC   │                           │    │
│  │  ┌─────────────────────┐ │◄────────┤  ┌─────────────────────┐ │    │
│  │  │ settings-           │ │         │  │ service.go          │ │    │
│  │  │ antitracker.vue     │ │         │  │ - GetAntiTracker    │ │    │
│  │  │                     │ │         │  │   Status()          │ │    │
│  │  │ • Enable/Disable    │ │         │  │ - SetManualDNS()    │ │    │
│  │  │ • Select Block List │ │         │  │ - getAntiTracker    │ │    │
│  │  │ • Hardcore Mode     │ │         │  │   Dns()             │ │    │
│  │  └─────────────────────┘ │         │  └─────────────────────┘ │    │
│  │                           │         │                           │    │
│  │  ┌─────────────────────┐ │         │  ┌─────────────────────┐ │    │
│  │  │ Store               │ │         │  │ DNS Manager         │ │    │
│  │  │ (Vuex)              │ │         │  │ (dns.go)            │ │    │
│  │  │                     │ │         │  │                     │ │    │
│  │  │ antiTracker: {      │ │         │  │ • Apply DNS         │ │    │
│  │  │   Enabled: bool     │ │         │  │   Settings          │ │    │
│  │  │   Hardcore: bool    │ │         │  │ • Platform-Specific │ │    │
│  │  │   BlockListName     │ │         │  │   Implementation    │ │    │
│  │  │ }                   │ │         │  └─────────────────────┘ │    │
│  │  └─────────────────────┘ │         │                           │    │
│  └──────────────────────────┘         └───────────────────────────┘    │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ DNS Queries
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           VPN Tunnel (Encrypted)                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    IVPN Cloud Infrastructure                             │
│                                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              AntiTracker DNS Servers (10.0.254.x)                 │  │
│  │                                                                    │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │  │
│  │  │   Basic      │  │ Comprehensive│  │ Restrictive  │  ...      │  │
│  │  │  10.0.254.4  │  │  10.0.254.6  │  │  10.0.254.18 │           │  │
│  │  │  (Normal)    │  │  (Normal)    │  │  (Normal)    │           │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘           │  │
│  │                                                                    │  │
│  │  ┌─────────────────────────────────────────────────┐             │  │
│  │  │           DNS Filtering Engine                  │             │  │
│  │  │                                                  │             │  │
│  │  │  • Check domain against block lists             │             │  │
│  │  │  • Apply Hardcore mode rules                    │             │  │
│  │  │  • Return blocked/allowed response              │             │  │
│  │  └─────────────────────────────────────────────────┘             │  │
│  │                                                                    │  │
│  │  ┌─────────────────────────────────────────────────┐             │  │
│  │  │              Block Lists Database               │             │  │
│  │  │                                                  │             │  │
│  │  │  • OISD Big         • Hagezi Pro               │             │  │
│  │  │  • EasyList         • Steven Black             │             │  │
│  │  │  • Hagezi Light     • Custom IVPN lists        │             │  │
│  │  │  • Plus Hardcore variants (Google/FB blocking) │             │  │
│  │  └─────────────────────────────────────────────────┘             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                          Legitimate DNS Response
                        (or NXDOMAIN if blocked)
```

## DNS Query Flow

```
┌──────────────┐
│   User App   │  (Browser, Email, etc.)
│  (Chrome,    │
│   Firefox)   │
└──────┬───────┘
       │
       │ DNS Query: "tracker.example.com"
       │
       ▼
┌──────────────────────────────────────────────────┐
│         Operating System DNS Resolver            │
│  Configured to: 10.0.254.4 (AntiTracker Basic)  │
└──────┬───────────────────────────────────────────┘
       │
       │ DNS Query (through VPN tunnel)
       │
       ▼
┌─────────────────────────────────────────────────┐
│         IVPN VPN Server                         │
│  Routes to AntiTracker DNS: 10.0.254.4          │
└──────┬──────────────────────────────────────────┘
       │
       │
       ▼
┌─────────────────────────────────────────────────┐
│    IVPN AntiTracker DNS Server (10.0.254.4)    │
│                                                  │
│  1. Receive: "tracker.example.com"              │
│  2. Check Block List: ✓ Found in tracker list  │
│  3. Decision: BLOCK                             │
│  4. Return: NXDOMAIN (domain not found)         │
└──────┬──────────────────────────────────────────┘
       │
       │ NXDOMAIN Response
       │
       ▼
┌──────────────────────────────────────────────────┐
│         Operating System DNS Resolver            │
└──────┬───────────────────────────────────────────┘
       │
       │ NXDOMAIN (not found)
       │
       ▼
┌──────────────┐
│   User App   │  Connection fails → tracker blocked ✓
└──────────────┘
```

## Component Interaction Diagram

```
   User Action                 Frontend                 Backend                 Cloud
       │                          │                        │                      │
       │ Enable AntiTracker       │                        │                      │
       ├─────────────────────────►│                        │                      │
       │                          │                        │                      │
       │                          │ Update Vuex State      │                      │
       │                          │ antiTracker.Enabled=true                     │
       │                          │                        │                      │
       │                          │ IPC: SetAlternateDns   │                      │
       │                          ├───────────────────────►│                      │
       │                          │                        │                      │
       │                          │                        │ SetManualDNS()       │
       │                          │                        │ getAntiTrackerDns()  │
       │                          │                        │ → 10.0.254.4         │
       │                          │                        │                      │
       │                          │                        │ Apply System DNS     │
       │                          │                        │ settings             │
       │                          │                        │                      │
       │                          │                        │ Update Firewall      │
       │                          │                        │ (allow 10.0.254.4)   │
       │                          │                        │                      │
       │                          │   IPC: Status Update   │                      │
       │                          │◄───────────────────────┤                      │
       │                          │                        │                      │
       │ UI shows: Enabled ✓      │                        │                      │
       │◄─────────────────────────┤                        │                      │
       │                          │                        │                      │
       │ Browse website           │                        │                      │
       ├──────────────────────────┴────────────────────────┴──────────────────────┤
       │                                                                            │
       │                      DNS Query: "ad.example.com"                          │
       ├───────────────────────────────────────────────────────────────────────────►
       │                                                                            │
       │                                                   Check block list         │
       │                                                   ad.example.com → BLOCK   │
       │                                                                            │
       │                      Response: NXDOMAIN (blocked)                         │
       ◄───────────────────────────────────────────────────────────────────────────┤
       │                                                                            │
       │ Ad doesn't load ✓                                                         │
       │                                                                            │
```

## File Organization

```
IVPN-desktop-app/
│
├── ui/                                    # Frontend (Electron/Vue.js)
│   └── src/
│       ├── components/
│       │   └── settings/
│       │       └── settings-antitracker.vue    ◄─── Main UI Component
│       ├── store/
│       │   ├── module-settings.js              ◄─── State Management
│       │   └── module-vpn-state.js             ◄─── VPN State
│       └── daemon-client/
│           └── index.js                        ◄─── IPC Communication
│
├── daemon/                                # Backend (Go)
│   ├── service/
│   │   ├── service.go                         ◄─── Core Logic
│   │   ├── service_connect.go                 ◄─── Connection Handling
│   │   ├── types/
│   │   │   └── connection.go                  ◄─── AntiTrackerMetadata
│   │   └── dns/
│   │       ├── dns.go                         ◄─── DNS Management
│   │       ├── dns_settings.go                ◄─── DNS Settings
│   │       └── resolvers.go                   ◄─── DNS Resolvers
│   ├── protocol/
│   │   ├── protocol.go                        ◄─── Protocol Handler
│   │   └── types/
│   │       ├── requests.go                    ◄─── SetAlternateDns
│   │       └── responses.go                   ◄─── DnsStatus
│   ├── api/
│   │   └── types/
│   │       └── servers.go                     ◄─── Server Config Structs
│   └── References/
│       └── common/
│           └── etc/
│               └── servers.json               ◄─── DNS IPs Configuration
│
├── cli/                                   # Command Line Interface
│   └── commands/
│       ├── dns.go                             ◄─── AntiTracker CLI
│       └── connection.go                      ◄─── Connection w/ flags
│
└── docs/                                  # Documentation
    ├── ANTITRACKER_ARCHITECTURE.md            ◄─── Full Architecture
    └── ANTITRACKER_FILES_REFERENCE.md         ◄─── Quick Reference
```

## Data Structure Flow

```
┌───────────────────────────────────────────────────────────────┐
│                    Frontend (JavaScript)                      │
│                                                                │
│  {                                                             │
│    antiTracker: {                                             │
│      Enabled: true,                ┌─────────────────────┐   │
│      Hardcore: false,              │  Vuex Store         │   │
│      AntiTrackerBlockListName:     │  (State Management) │   │
│        "Basic"                     └─────────────────────┘   │
│    }                                                           │
│  }                                                             │
└───────────────────────────────────────────────────────────────┘
                            │
                            │ IPC Message
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│                      Backend (Go)                             │
│                                                                │
│  type AntiTrackerMetadata struct {                            │
│    Enabled                  bool                              │
│    Hardcore                 bool                              │
│    AntiTrackerBlockListName string                            │
│  }                                                             │
│                            │                                   │
│                            │ Resolution                        │
│                            │                                   │
│                            ▼                                   │
│  DNS Server IP: 10.0.254.4 (from servers.json)               │
│                                                                │
└───────────────────────────────────────────────────────────────┘
                            │
                            │ Apply to System
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│                   Operating System                            │
│                                                                │
│  DNS Configuration: 10.0.254.4                                │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

## Block List Selection Logic

```
User Selects "Basic"
        │
        ▼
┌──────────────────────────────────────┐
│  Frontend: AtPlusListNameSelected    │
│  = "Basic"                            │
└──────────┬───────────────────────────┘
           │
           │ IPC: SetDNS()
           │
           ▼
┌──────────────────────────────────────┐
│  Backend: getAntiTrackerDns()        │
│                                       │
│  1. atListName = "basic" (lowercase) │
│  2. Search servers.json:              │
│     antitracker_plus.DnsServers[]    │
│  3. Find: Name="Basic"                │
│  4. Check Hardcore mode               │
│     • No:  Return Normal=10.0.254.4  │
│     • Yes: Return Hardcore=10.0.254.5│
└──────────┬───────────────────────────┘
           │
           │ DNS IP determined
           │
           ▼
┌──────────────────────────────────────┐
│  Apply DNS settings to system        │
│  Set resolver to: 10.0.254.4         │
└──────────────────────────────────────┘
```

## Hardcore Mode Effect

```
Normal Mode                          Hardcore Mode
     │                                    │
     ▼                                    ▼
┌─────────────────┐              ┌─────────────────┐
│  DNS: 10.0.254.4│              │  DNS: 10.0.254.5│
│                 │              │                 │
│  Blocks:        │              │  Blocks:        │
│  • Ads          │              │  • Ads          │
│  • Trackers     │              │  • Trackers     │
│  • Malware      │              │  • Malware      │
│                 │              │  • Google       │◄── Additional
│                 │              │  • Facebook     │◄── Blocking
└─────────────────┘              └─────────────────┘

Example Domains:

Normal: ad.example.com → BLOCKED ✓
        google.com → ALLOWED ✓
        facebook.com → ALLOWED ✓

Hardcore: ad.example.com → BLOCKED ✓
          google.com → BLOCKED ✓     ◄── Difference
          facebook.com → BLOCKED ✓   ◄── Difference
```

## Multi-Platform Implementation

```
┌────────────────────────────────────────────────────────────────┐
│                    Cross-Platform DNS Manager                   │
│                     (daemon/service/dns/dns.go)                 │
└────────────────────┬───────────────┬───────────────┬───────────┘
                     │               │               │
        ┌────────────▼─────┐  ┌──────▼────────┐  ┌──▼──────────────┐
        │   dns_windows.go │  │ dns_linux.go  │  │  dns_darwin.go  │
        │                  │  │               │  │                 │
        │ • WinAPI calls   │  │ • resolvectl  │  │ • scutil        │
        │ • Registry edits │  │ • resolv.conf │  │ • networksetup  │
        └──────────────────┘  └───────────────┘  └─────────────────┘
                │                     │                    │
                ▼                     ▼                    ▼
        ┌──────────────────────────────────────────────────────────┐
        │           System DNS Configuration Updated               │
        │              All DNS → 10.0.254.4                        │
        └──────────────────────────────────────────────────────────┘
```

## CLI Workflow

```
Command Line                          Daemon                       Cloud

$ ivpn antitracker -on
        │
        │ Parse command
        │
        ▼
┌──────────────────┐
│  CmdAntitracker  │
│  Run()           │
└────────┬─────────┘
         │
         │ IPC: SetAlternateDns
         │ {
         │   AntiTracker: {
         │     Enabled: true,
         │     Hardcore: false,
         │     BlockListName: ""
         │   }
         │ }
         │
         ▼
     ┌──────────────────┐
     │  Service         │
     │  SetManualDNS()  │
     └────────┬─────────┘
              │
              │ Apply DNS
              │
              ▼
     ┌──────────────────┐
     │  System DNS      │
     │  → 10.0.254.4    │─────────────►  DNS Queries
     └──────────────────┘                     │
              │                               │
              │ Response                      ▼
              │                     ┌──────────────────┐
              ▼                     │ AntiTracker DNS  │
     ┌──────────────────┐          │ Filters requests │
     │  Status output   │          └──────────────────┘
     │  AntiTracker: ON │
     └──────────────────┘

$ ivpn status
        │
        ▼
AntiTracker: ON (Basic)
```

## Connection Lifecycle

```
1. VPN Disconnected
   ├─ DNS: System default (e.g., 8.8.8.8, ISP DNS)
   └─ AntiTracker: NOT ACTIVE

2. VPN Connecting
   ├─ Read AntiTracker settings from preferences
   ├─ Determine DNS IP (e.g., 10.0.254.4)
   └─ Prepare firewall rules

3. VPN Connected
   ├─ Apply AntiTracker DNS: 10.0.254.4
   ├─ Route DNS through VPN tunnel
   ├─ Update firewall (allow 10.0.254.4)
   └─ AntiTracker: ACTIVE ✓

4. DNS Query
   ├─ App requests: ad.tracker.com
   ├─ System sends to: 10.0.254.4
   ├─ Through VPN to IVPN AntiTracker DNS
   ├─ Server checks block list
   └─ Response: NXDOMAIN (blocked) ✓

5. VPN Disconnecting
   ├─ Restore original DNS settings
   └─ Remove firewall rules

6. VPN Disconnected
   ├─ DNS: Back to system default
   └─ AntiTracker: NOT ACTIVE
```

## Summary

The key takeaway from these diagrams:

1. **Client-side (this repo):** UI configuration + DNS routing setup
2. **Server-side (IVPN cloud):** Actual blocking lists + filtering logic
3. **Communication:** VPN tunnel carries DNS queries to IVPN's AntiTracker DNS servers
4. **Result:** Ads and trackers are blocked at the DNS level before they can load

The desktop application is primarily a **configuration interface** that tells the VPN which DNS servers to use. The heavy lifting (maintaining block lists, filtering domains) happens on IVPN's infrastructure.
