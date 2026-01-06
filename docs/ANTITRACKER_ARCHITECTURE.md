# AntiTracker (Ad + Tracker Blocking) Architecture

## Overview

IVPN's **AntiTracker** is a DNS-based ad and tracker blocking feature that blocks ads, malicious websites, and third-party trackers using private DNS servers. This document explains where and how this functionality is implemented in the IVPN Desktop application.

## Key Concept: DNS-Based Blocking

**Important:** AntiTracker is primarily a **cloud/server-side** feature, not implemented client-side. The desktop application:
- Configures which AntiTracker DNS servers to use
- Routes DNS queries through IVPN's private AntiTracker DNS servers
- The actual blocking happens on IVPN's DNS servers in the cloud

The blocking lists and filtering logic reside on IVPN's infrastructure, not in this desktop application code.

## Architecture

```
[Desktop App Client] --> [IVPN VPN Tunnel] --> [IVPN AntiTracker DNS Server (Cloud)]
                                                        |
                                                        v
                                                [Blocking Lists]
                                                [DNS Filtering Logic]
```

### How It Works

1. **User enables AntiTracker** in the desktop app UI
2. **App configures DNS settings** to use IVPN's AntiTracker DNS servers (e.g., 10.0.254.2)
3. **DNS queries are routed** through the VPN tunnel to IVPN's AntiTracker DNS servers
4. **Server-side filtering** blocks domains based on configured block lists
5. **Clean DNS responses** are returned to the client

## File Locations

### Frontend/UI Components (Electron/Vue.js)

#### Main AntiTracker Settings UI
- **File:** `ui/src/components/settings/settings-antitracker.vue`
- **Purpose:** User interface for configuring AntiTracker settings
- **Features:**
  - Enable/disable AntiTracker
  - Select block lists (Basic, Comprehensive, Restrictive, Individual lists)
  - Toggle Hardcore Mode (blocks Google/Facebook)

#### Integration Points
- `ui/src/views/Component-Settings.vue` - Settings page with AntiTracker tab
- `ui/src/views/Component-Main.vue` - Quick access to AntiTracker settings
- `ui/src/components/blocks/block-connection-details.vue` - Connection details showing AntiTracker status
- `ui/src/store/module-settings.js` - State management for AntiTracker settings
- `ui/src/store/module-vpn-state.js` - VPN state including AntiTracker status
- `ui/src/daemon-client/index.js` - Communication with daemon for AntiTracker configuration
- `ui/src/daemon-client/connectionParams.js` - Connection parameters including AntiTracker metadata

### Backend/Daemon (Go)

#### Core Service Implementation
- **File:** `daemon/service/service.go`
- **Key Functions:**
  - `GetAntiTrackerStatus()` - Returns current AntiTracker configuration
  - `SetManualDNS(dnsCfg, antiTracker)` - Sets DNS with AntiTracker parameters
  - `getAntiTrackerDns(isHardcore, antiTrackerPlusList)` - Determines which DNS server IPs to use
  - `normalizeAntiTrackerBlockListName(antiTracker)` - Validates and normalizes block list names
  - `saveDefaultDnsParams(dnsCfg, antiTrackerCfg)` - Persists AntiTracker configuration

#### DNS Configuration Files
- `daemon/service/dns/dns.go` - DNS management core
- `daemon/service/dns/dns_settings.go` - DNS settings data structures
- `daemon/service/dns/resolvers.go` - DNS resolver setup
- `daemon/service/dns/dns_windows.go` - Windows-specific DNS implementation
- `daemon/service/dns/dns_linux.go` - Linux-specific DNS implementation  
- `daemon/service/dns/dns_darwin.go` - macOS-specific DNS implementation

#### Protocol/Communication
- `daemon/protocol/protocol.go` - Daemon protocol handling AntiTracker requests
- `daemon/protocol/types/requests.go` - Request types including `SetAlternateDns` with AntiTracker
- `daemon/protocol/types/responses.go` - Response types including AntiTracker status
- `daemon/service/types/connection.go` - Connection metadata including `AntiTrackerMetadata` struct

#### Server Configuration
- `daemon/api/types/servers.go` - Server list structures including AntiTracker DNS IPs
  - `AntitrackerInfo` struct - Basic AntiTracker DNS servers (default/hardcore)
  - `AntiTrackerPlusServer` struct - AntiTracker Plus block list servers
  - `AntiTrackerPlusInfo` struct - Collection of AntiTracker Plus servers
- `daemon/References/common/etc/servers.json` - Server configuration including AntiTracker DNS IPs

#### Connection Management
- `daemon/service/service_connect.go` - VPN connection setup with AntiTracker DNS
- `daemon/service/preferences/preferences.go` - Persistence of AntiTracker settings

### CLI (Command Line Interface)

- **File:** `cli/commands/dns.go`
- **Features:**
  - `CmdAntitracker` - AntiTracker command implementation
  - Enable/disable AntiTracker from CLI
  - Select block lists via command line
  - Functions: `GetAntitrackerIP()`, `printAntitrackerConfigInfo()`

- **File:** `cli/commands/connection.go`
- **Features:**
  - Connection command with `--antitracker` and `--antitracker_hard` flags
  - Apply AntiTracker to specific VPN connections

## AntiTracker Configuration

### Block Lists (AntiTracker Plus)

The application supports multiple DNS block lists:

#### Pre-defined Lists
1. **Basic** - Standard ad and tracker blocking
2. **Comprehensive** - More aggressive blocking
3. **Restrictive** - Most restrictive blocking

#### Individual Lists
- **Oisdbig** - OISD Big list
- **Easylist** - EasyList + EasyPrivacy
- **Stevenblack** - Steven Black Unified + Ads + Malware
- **Hagezilight** - Hagezi Light
- **Hagezipro** - Hagezi Pro
- **Hageziproplus** - Hagezi Pro++
- **Hageziultimate** - Hagezi Ultimate

Each list has both **Normal** and **Hardcore** variants.

### DNS Server IPs

AntiTracker uses private DNS server IPs in the 10.0.254.x range:

**Example from servers.json:**
```json
"antitracker": {
  "default": {
    "ip": "10.0.254.2",
    "multihop-ip": "10.0.254.102"
  },
  "hardcore": {
    "ip": "10.0.254.3",
    "multihop-ip": "10.0.254.103"
  }
}
```

**AntiTracker Plus Lists:**
```json
"antitracker_plus": {
  "DnsServers": [
    {"Name": "Basic", "Normal": "10.0.254.4", "Hardcore": "10.0.254.5"},
    {"Name": "Comprehensive", "Normal": "10.0.254.6", "Hardcore": "10.0.254.7"},
    {"Name": "Oisdbig", "Normal": "10.0.254.2", "Hardcore": "10.0.254.3"},
    ...
  ]
}
```

### Hardcore Mode

**Hardcore Mode** adds additional blocking for major surveillance-based companies:
- Currently blocks: Google and Facebook services
- May impact user experience (e.g., Google services, Facebook login)
- Uses separate DNS servers with additional blocking rules

## Data Structures

### AntiTrackerMetadata (Go)
```go
// daemon/service/types/connection.go
type AntiTrackerMetadata struct {
    Enabled                  bool   // Is AntiTracker enabled
    Hardcore                 bool   // Is Hardcore mode enabled
    AntiTrackerBlockListName string // Selected block list name
}
```

### Settings State (JavaScript)
```javascript
// ui/src/store/module-settings.js
antiTracker: {
  Enabled: false,
  Hardcore: false,
  AntiTrackerBlockListName: ""
}
```

## Communication Flow

### Enabling AntiTracker

1. **User clicks toggle** in `settings-antitracker.vue`
2. **Vue component updates state** via `$store.dispatch("settings/antiTracker", at)`
3. **IPC message sent** to main process via `sender.SetDNS()`
4. **Daemon receives request** in `protocol.go` handling `SetAlternateDns`
5. **Service processes request** in `service.go` via `SetManualDNS()`
6. **DNS configuration applied** via platform-specific DNS managers
7. **Firewall rules updated** to allow AntiTracker DNS servers
8. **Status returned to UI** showing AntiTracker enabled

### During VPN Connection

1. **Connection parameters collected** including AntiTracker metadata
2. **DNS servers determined** based on AntiTracker settings
3. **VPN configured** to route DNS through selected AntiTracker servers
4. **All DNS queries** flow through IVPN's AntiTracker DNS servers
5. **Blocking happens server-side** on IVPN infrastructure

## Important Notes

### What's NOT in This Repository

1. **Blocking Lists** - Maintained by third parties (OISD, EasyList, Hagezi, etc.)
2. **DNS Filtering Logic** - Runs on IVPN's server infrastructure
3. **Block List Updates** - Managed on IVPN's DNS servers
4. **Actual DNS Resolution** - Performed by IVPN's AntiTracker DNS servers

### What IS in This Repository

1. **Configuration UI** - Select block lists, enable/disable, hardcore mode
2. **DNS Server Selection** - Choose which AntiTracker DNS IPs to use
3. **VPN Integration** - Route DNS through the VPN tunnel
4. **Settings Persistence** - Remember user's AntiTracker preferences
5. **Status Display** - Show AntiTracker state in the UI

## Customization & Extension

### Adding New Block Lists

To support additional block lists:

1. **Server-side:** IVPN must deploy new DNS servers with the desired lists
2. **Update `servers.json`:** Add new entries to `antitracker_plus.DnsServers`
3. **UI automatically updates:** The dropdown will show new lists

### Modifying Block Lists

Block lists cannot be modified in the desktop app. They are:
- Maintained on IVPN's DNS server infrastructure
- Updated regularly by IVPN's operations team
- Based on third-party sources (OISD, EasyList, etc.)

### Custom DNS vs AntiTracker

Users can choose between:
- **AntiTracker:** Use IVPN's filtering DNS servers
- **Custom DNS:** Use their own DNS servers (no filtering)
- **Mix:** Not supported - AntiTracker has priority over custom DNS

## Testing AntiTracker

### Verify It's Working

1. **Connect to IVPN** with AntiTracker enabled
2. **Check DNS settings** on your system - should show 10.0.254.x
3. **Visit ad-heavy websites** - ads should be blocked
4. **Check DNS queries** - should resolve through IVPN's DNS servers

### CLI Testing

```bash
# Enable AntiTracker with Basic list
ivpn antitracker -on

# Enable Hardcore mode with specific list  
ivpn antitracker -on_hardcore Hagezipro

# Disable AntiTracker
ivpn antitracker -off

# Check status
ivpn status
```

## Security & Privacy

### AntiTracker Benefits

1. **Privacy Protection:** Block trackers from collecting your data
2. **Ad Blocking:** Remove ads at DNS level (works across all apps)
3. **Malware Protection:** Block known malicious domains
4. **Speed Improvement:** Fewer resources loaded, faster browsing

### Privacy Considerations

- **DNS queries visible to IVPN:** Your DNS queries go through IVPN's servers
- **IVPN's no-logs policy:** IVPN claims not to log DNS queries
- **Trade-off:** Enhanced blocking vs. trusting IVPN with DNS

## Related Documentation

- [IVPN AntiTracker](https://www.ivpn.net/antitracker)
- [AntiTracker Hardcore Mode FAQ](https://www.ivpn.net/antitracker/hardcore)  
- [AntiTracker Plus Lists Explained](https://www.ivpn.net/knowledgebase/general/antitracker-plus-lists-explained/)
- [IVPN Privacy Policy](https://www.ivpn.net/privacy/)

## Summary

The ad and tracker blocking in IVPN Desktop is implemented through:

1. **Client-side configuration** (this repository): UI to enable/configure AntiTracker
2. **DNS routing** (this repository): Route DNS through IVPN's tunnel
3. **Server-side filtering** (IVPN cloud): Actual blocking logic and lists

The desktop application focuses on configuration and DNS routing, while the actual blocking is performed by IVPN's cloud infrastructure.
