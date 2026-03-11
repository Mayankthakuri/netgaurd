# NetGuard — Local Network Privacy Utility

A defensive networking tool written in C for Linux and macOS that protects local
services from unauthorized inbound connections. NetGuard manages OS firewall rules
to whitelist trusted devices, block untrusted traffic on protected ports, and
automatically harden the system when connected to public networks.

**This is a purely defensive tool.** It does not spoof IPs, evade IDS/IPS, hide
from network administrators, or include any stealth or anti-detection features.

## Features

- **Localhost binding protection** — block inbound connections to sensitive ports
- **Trusted device whitelist** — allow specific IPs through the firewall
- **Automatic strict mode** — detects untrusted WiFi and locks down automatically
- **mDNS/Bonjour blocking** — prevents local service discovery on public networks
- **Structured logging** — every rule change and connection event is logged with
  timestamp, source IP, port, and reason
- **Interactive CLI menu** and scriptable command-line flags
- **Cross-platform** — macOS (pfctl) and Linux (iptables)

## Project Structure

```
netguard/
├── Makefile                   # Build system
├── README.md                  # This file
├── netguard.conf.example      # Sample configuration
├── include/
│   ├── netguard.h             # Common definitions, constants, types
│   ├── config.h               # Configuration management interface
│   ├── firewall.h             # Firewall integration interface
│   ├── logger.h               # Logging subsystem interface
│   └── utils.h                # Utility functions interface
└── src/
    ├── main.c                 # Entry point, CLI menu, argument parsing
    ├── config.c               # Config file parsing and management
    ├── firewall.c             # pfctl (macOS) / iptables (Linux) integration
    ├── logger.c               # Timestamped structured logging
    └── utils.c                # IP validation, network detection, mDNS control
```

## Building

### Prerequisites

- C11-compatible compiler (GCC, Clang)
- macOS or Linux
- Root/sudo access (required for firewall management)

### Build

```sh
make
```

### Debug build (with AddressSanitizer)

```sh
make debug
```

### Clean

```sh
make clean
```

## Configuration

Copy the example config and edit:

```sh
cp netguard.conf.example netguard.conf
```

### Config keys

| Key                         | Values            | Description                              |
|-----------------------------|-------------------|------------------------------------------|
| `mode`                      | `normal`/`strict` | Normal allows trusted IPs; strict blocks all |
| `log_file`                  | file path         | Where to write logs                      |
| `disable_mdns_on_untrusted` | `yes`/`no`        | Block mDNS (UDP 5353) on untrusted nets  |
| `trusted_ip`                | IPv4 address      | IP allowed through (one per line)        |
| `protected_port`            | 1–65535           | Port to protect (one per line)           |
| `trusted_network`           | SSID string       | WiFi name considered safe                |

## Usage

### Interactive mode

```sh
sudo ./netguard
# or with a custom config:
sudo ./netguard -c /path/to/netguard.conf
```

This opens the CLI menu:

```
  1. Add trusted device
  2. Remove trusted device
  3. Show active rules
  4. Enable STRICT local-only mode
  5. Enable NORMAL mode
  6. Apply firewall rules
  7. Clear all rules
  8. Check network status
  9. Export logs
 10. Show configuration
  0. Exit
```

### Command-line (scriptable)

```sh
# Apply rules from config
sudo ./netguard --apply

# Switch to strict mode and apply
sudo ./netguard --strict

# Switch to normal mode and apply
sudo ./netguard --normal

# Show current status
sudo ./netguard --status

# Clear all NetGuard rules
sudo ./netguard --clear

# Show help
./netguard --help
```

### Example workflow

```sh
# 1. Set up config
cp netguard.conf.example netguard.conf

# 2. Edit trusted IPs and ports
#    (edit netguard.conf)

# 3. Build
make

# 4. Apply rules
sudo ./netguard --apply

# 5. Check what's active
sudo ./netguard --status

# 6. When on public WiFi, lock down
sudo ./netguard --strict

# 7. When back home, relax
sudo ./netguard --normal

# 8. Export logs for review
sudo ./netguard
# (choose option 9 from the menu)
```

## How It Works

### macOS (pfctl)

NetGuard creates a PF anchor named `netguard` in `/etc/pf.conf`. Rules are
written to a temporary file and loaded into the anchor:

```
pfctl -a netguard -f /tmp/netguard_rules.conf
```

A backup of the original `/etc/pf.conf` is created at
`/etc/pf.conf.netguard.bak` before any modifications.

### Linux (iptables)

NetGuard creates a custom iptables chain `NETGUARD` and inserts it at the top
of the `INPUT` chain. Rules use `RETURN` (not `ACCEPT`) to avoid interfering
with other firewall rules. Protected ports get `LOG` + `DROP` rules for
denied connections.

### mDNS blocking

Rather than stopping system services (which may require SIP changes on macOS),
NetGuard blocks mDNS traffic at the firewall level by dropping UDP port 5353.
This is safe, documented, and fully reversible.

### Auto network detection

On startup, NetGuard checks the current WiFi SSID against the `trusted_network`
list. If the network is untrusted and mode is `normal`, it automatically
switches to `strict` mode for maximum protection.

## Security Practices

- **No hardcoded secrets** — all configuration is in the config file
- **Input validation** — IPs validated via `inet_pton()`, ports range-checked,
  shell metacharacters stripped from all user input
- **Memory safety** — bounded buffers (`snprintf`), `memset` initialization,
  no dynamic allocation (all stack/static)
- **Least privilege** — root is required only for firewall commands
- **Command injection prevention** — all shell commands built from validated
  components via `snprintf()`, never from raw user input
- **Graceful shutdown** — SIGINT/SIGTERM handlers ensure clean exit
- **Immediate log flushing** — logs are flushed after every write
- **Log file permissions** — log file created with mode 0600

## Uninstalling

```sh
# Remove the binary
sudo make uninstall

# Clear any active firewall rules
sudo ./netguard --clear

# On macOS, restore pf.conf from backup if desired:
sudo cp /etc/pf.conf.netguard.bak /etc/pf.conf
sudo pfctl -f /etc/pf.conf

# On Linux, remove the iptables chain:
sudo iptables -D INPUT -j NETGUARD
sudo iptables -F NETGUARD
sudo iptables -X NETGUARD
```

## License

This project is provided as-is for educational and defensive security purposes.
