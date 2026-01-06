# AntiTracker Files Quick Reference

This is a quick reference for developers looking to find or modify AntiTracker (ad and tracker blocking) related code.

## Quick Answer

**Where is the ad/tracker blocking?**
- **Client Side (this repo):** Configuration UI and DNS routing only
- **Server Side (IVPN cloud):** Actual blocking logic and lists (NOT in this repo)

## Key Files by Component

### 🎨 Frontend UI (Electron/Vue.js)

| File | Purpose | Lines of Interest |
|------|---------|------------------|
| `ui/src/components/settings/settings-antitracker.vue` | Main AntiTracker settings UI | Full file (172 lines) |
| `ui/src/views/Component-Settings.vue` | Settings page with AntiTracker tab | Lines 83-88, 172-173 |
| `ui/src/components/blocks/block-connection-details.vue` | AntiTracker toggle in connection details | Lines 34-53 |
| `ui/src/store/module-settings.js` | AntiTracker state management | Lines 155-158, 384-393, 835-837 |
| `ui/src/store/module-vpn-state.js` | VPN state with AntiTracker status | Lines 168-172, 693-718 |
| `ui/src/daemon-client/index.js` | IPC communication for AntiTracker | Lines 407-410, 726, 1744-1750 |

### 🔧 Backend Daemon (Go)

| File | Purpose | Key Functions |
|------|---------|---------------|
| `daemon/service/service.go` | Core AntiTracker service logic | `GetAntiTrackerStatus()`, `SetManualDNS()`, `getAntiTrackerDns()`, `normalizeAntiTrackerBlockListName()` (lines 798-996) |
| `daemon/service/types/connection.go` | AntiTracker metadata structure | `AntiTrackerMetadata` struct (lines 45-65) |
| `daemon/protocol/protocol.go` | Protocol handler for AntiTracker | Lines 109-112, 857-865 |
| `daemon/protocol/types/requests.go` | Request types | `SetAlternateDns` struct (lines 156-157) |
| `daemon/protocol/types/responses.go` | Response types | `DaemonSettings` and `DnsStatus` (lines 135, 232) |
| `daemon/api/types/servers.go` | Server configuration structures | `AntitrackerInfo`, `AntiTrackerPlusServer`, `AntiTrackerPlusInfo` (lines 130-146, 222-223) |
| `daemon/service/service_connect.go` | VPN connection with AntiTracker | Lines 245, 300, 307-437, 493-599 |

### 💻 CLI (Command Line)

| File | Purpose | Key Components |
|------|---------|----------------|
| `cli/commands/dns.go` | AntiTracker CLI commands | `CmdAntitracker` struct, `GetAntitrackerIP()` (lines 102-433) |
| `cli/commands/connection.go` | Connection with AntiTracker flags | Lines 132-133, 190-192, 687-695 |
| `cli/main.go` | CLI entry point | Line 112 (registers antitracker command) |

### 📄 Configuration Files

| File | Purpose |
|------|---------|
| `daemon/References/common/etc/servers.json` | DNS server IPs and block list configuration |

### 🧪 DNS Implementation

| File | Purpose |
|------|---------|
| `daemon/service/dns/dns.go` | DNS management core |
| `daemon/service/dns/dns_settings.go` | DNS settings structures |
| `daemon/service/dns/resolvers.go` | DNS resolver setup |
| `daemon/service/dns/dns_windows.go` | Windows DNS implementation |
| `daemon/service/dns/dns_linux.go` | Linux DNS implementation |
| `daemon/service/dns/dns_darwin.go` | macOS DNS implementation |

## Code Search Tips

### Find AntiTracker References

```bash
# Search for AntiTracker in Go files
grep -r "antitracker\|AntiTracker" daemon/ --include="*.go"

# Search for AntiTracker in Vue/JS files
grep -r "antitracker\|AntiTracker" ui/src/ --include="*.vue" --include="*.js"

# Search for DNS blocking
grep -r "dns\|DNS" daemon/service/dns/ --include="*.go"
```

### Common Search Terms

- `AntiTracker` - Main feature name
- `antitracker` - Lowercase variant
- `AntiTrackerMetadata` - Go struct
- `SetAlternateDns` - Protocol message
- `getAntiTrackerDns` - DNS IP resolution
- `settings-antitracker` - UI component

## Data Flow

```
User Action (UI)
    ↓
settings-antitracker.vue
    ↓
Store (module-settings.js)
    ↓
IPC Message (daemon-client/index.js)
    ↓
Protocol Handler (protocol/protocol.go)
    ↓
Service (service/service.go)
    ↓
DNS Manager (service/dns/*.go)
    ↓
System DNS Configuration
    ↓
VPN Tunnel → IVPN AntiTracker DNS Server (Cloud)
```

## Common Tasks

### Add a New Block List

**Note:** This requires server-side changes by IVPN team

1. IVPN deploys new DNS servers with the block list
2. Update `daemon/References/common/etc/servers.json`:
   ```json
   {
     "Name": "NewList",
     "Description": "My New Block List",
     "Normal": "10.0.254.XX",
     "Hardcore": "10.0.254.YY"
   }
   ```
3. UI automatically picks it up (no code changes needed)

### Change Default Block List

Edit `daemon/service/service.go`, line ~977:
```go
atListName := "basic"  // Change to your preferred default
```

### Modify Hardcore Mode Behavior

**Note:** Actual blocking is server-side

Client-side display changes in:
- `ui/src/components/settings/settings-antitracker.vue`
- `daemon/service/service.go` (hardcore flag handling)

### Customize UI Text

Edit `ui/src/components/settings/settings-antitracker.vue`:
- Line 3: "ANTITRACKER SETTINGS" title
- Lines 6-9: Description text
- Lines 31-42: Block list description

### Add AntiTracker to Custom VPN Protocol

If adding a new VPN protocol:
1. Include `AntiTrackerMetadata` in connection parameters
2. Call `service.SetManualDNS()` before connecting
3. Handle DNS configuration in protocol-specific code
4. See `daemon/service/service_connect.go` for examples

## Development Workflow

### Testing AntiTracker Changes

1. **Build the app:**
   ```bash
   cd ui/References/Windows  # or macOS/Linux
   ./build.bat  # or ./build.sh
   ```

2. **Run daemon in debug:**
   ```bash
   cd daemon
   go run main.go
   ```

3. **Test via CLI:**
   ```bash
   ivpn antitracker -on
   ivpn status
   ```

4. **Test via UI:**
   - Open IVPN app
   - Go to Settings → AntiTracker
   - Toggle settings and verify

### Debugging

**Go Backend:**
```bash
# Run daemon with logging
cd daemon
go run main.go -logging
```

**Vue Frontend:**
```bash
# Run in development mode
cd ui
npm run electron:serve
```

**Check DNS:**
```bash
# Windows
ipconfig /all | findstr "DNS"

# macOS/Linux  
cat /etc/resolv.conf
# or
resolvectl status
```

## Important Constants

### DNS IP Ranges

- AntiTracker DNS servers: `10.0.254.x`
- Local dnscrypt-proxy: `127.0.0.x`

### Default Block List

- Default: "Basic" (10.0.254.4 normal, 10.0.254.5 hardcore)
- Legacy: "Oisdbig" (10.0.254.2 normal, 10.0.254.3 hardcore)

### IPC Messages

- `SetAlternateDns` - Set DNS/AntiTracker configuration
- `SetAlternateDNSResp` - Response with DNS status
- `renderer-request-show-settings-antitracker` - Show AntiTracker settings

## Related Features

### Features That Interact With AntiTracker

1. **Custom DNS** - Disabled when AntiTracker is enabled
2. **Split Tunneling** - AntiTracker not available in Inverse mode
3. **Firewall** - Must allow AntiTracker DNS IPs
4. **Multi-hop** - Uses multihop-specific AntiTracker IPs

### Priority Rules

1. AntiTracker DNS > Custom DNS (when both configured)
2. Split Tunnel (Inverse) disables AntiTracker
3. Disconnected VPN = no AntiTracker (uses system DNS)

## Contributing

When contributing AntiTracker-related changes:

1. **Update both UI and daemon** if changing behavior
2. **Test on all platforms** (Windows, macOS, Linux)
3. **Verify CLI still works** after UI changes
4. **Check backward compatibility** with old preferences files
5. **Update this documentation** if adding new features

## FAQ

**Q: Can I modify the block lists?**
A: No, lists are maintained on IVPN's servers. You can only select which list to use.

**Q: Can I add my own DNS blocklist?**
A: Not directly. You'd need to set up your own DNS server and use Custom DNS instead of AntiTracker.

**Q: Where are the actual filtering rules?**
A: On IVPN's DNS servers in the cloud, not in this codebase.

**Q: Can I use AntiTracker without IVPN's VPN?**
A: No, AntiTracker DNS IPs are only accessible through IVPN's VPN tunnel.

**Q: How do I add a new AntiTracker feature?**
A: Depends on what you want:
- **New UI option:** Modify Vue components and state management
- **New block list:** Contact IVPN to deploy server-side
- **New DNS behavior:** Modify daemon service and DNS management

## See Also

- [ANTITRACKER_ARCHITECTURE.md](./ANTITRACKER_ARCHITECTURE.md) - Detailed architecture
- [readme-build-manual.md](./readme-build-manual.md) - Build instructions
- [IVPN AntiTracker Docs](https://www.ivpn.net/antitracker) - Official documentation
