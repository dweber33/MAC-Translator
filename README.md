# rmac - Incident Response MAC Address Translator

> A zero-dependency Bash utility for network and security engineers to instantly reformat MAC addresses, detect randomization, and generate copy-ready Splunk queries - built for the speed of a live incident.

---

## The Business Problem

During a network outage or security incident, **Mean Time to Resolution (MTTR) is everything**. Engineers routinely need a user's MAC address to trace connectivity, verify authentication sessions, or hunt through logs - but every platform formats MAC addresses differently:

| Platform | Format Example |
|---|---|
| Cisco IOS / WLC | `5a6d.67c4.ffa4` |
| Windows / Active Directory | `5A-6D-67-C4-FF-A4` |
| Linux / macOS | `5A:6D:67:C4:FF:A4` |
| Splunk / raw logs | `5a6d67c4ffa4` |

Manually converting between these formats mid-incident **breaks focus, introduces typos, and wastes time** that could be spent resolving the issue. `rmac` eliminates this bottleneck by running directly on your Linux bastion host - no browser, no external tools, no context switching.

---

## Solving the MAC Randomization Problem

Modern operating systems (iOS 14+, Android 10+, Windows 11) enable **MAC Randomization** (also called *Private Wi-Fi Address*) by default. Instead of transmitting the hardware's factory-burned Burned-In Address (BIA), these devices generate a rotating, locally administered MAC - one that is invisible to your DHCP reservations, NAC profiles, and captive portal sessions.

This causes several real-world headaches:

- **Ghost devices** - A MAC appears in wireless logs that doesn't match any inventory record or vendor OUI lookup.
- **DHCP reservation failures** - The device gets a random IP because its MAC keeps changing.
- **Authentication loop issues** - ClearPass or ISE can't build a persistent profile for a randomized MAC, causing repeated re-authentication prompts.
- **Captive portal breakage** - Sessions tied to the original MAC are invalidated on reconnect.

### How `rmac` Helps

`rmac` automatically inspects the **Locally Administered bit** (Bit 1 of the first octet) of any MAC address you provide. This single bit is the definitive, standards-defined indicator of a randomized or virtual MAC.

- If the bit is `1`, the device is almost certainly using a randomized address - and you now have the evidence to advise the user to disable it.
- The output includes a **binary breakdown of the first byte** with the specific bit highlighted, giving you a clear, shareable artifact for your incident notes.

```
First Byte (Hex): 5A
First Byte (Bin): 010110[1]0
Bit 1 (Locally Administered Flag): 1
⚠ This is likely a RANDOMIZED/PRIVATE (locally administered) MAC address.
```

---

## Features

- **Instant format translation** - Accepts any common MAC format as input (colon, dash, dot, or no delimiter) and outputs all four formats simultaneously.
- **Randomization detection** - Checks the Locally Administered bit with a binary breakdown and a plain-English verdict.
- **Copy-ready Splunk queries** - Generates pre-built search filters scoped to your wireless, DNS, and authentication log indexes.
- **Zero dependencies** - Pure Bash. No Python, no npm, no API calls. Works wherever Bash runs.

---

## Installation

```bash
# Download and make executable
chmod +x rmac

# Optional: move to PATH for system-wide access
sudo mv rmac /usr/local/bin/
```

---

## Usage

Pass any valid MAC address as the first argument. The script automatically strips and normalizes colons, dashes, and dots.

```bash
rmac <MAC_ADDRESS>
```

### Accepted Input Formats

```bash
rmac 5A:6D:67:C4:FF:A4      # Colon-separated
rmac 5A-6D-67-C4-FF-A4      # Dash-separated
rmac 5A6D.67C4.FFA4         # Cisco dot-notation
rmac 5a6d67c4ffa4           # Plain (no delimiter)
```

### Example Output

```
$ rmac 5A-6D-67-C4-FF-A4

Check MAC Randomization Bit
Normalized MAC: 5A:6D:67:C4:FF:A4
First Byte (Hex): 5A
First Byte (Bin): 010110[1]0
Bit 1 (Locally Administered Flag): 1
This is likely a RANDOMIZED/PRIVATE (locally administered) MAC address.

Alternate MAC Formats:
Colon-separated (uppercase):     5A:6D:67:C4:FF:A4
Dash-separated (uppercase):      5A-6D-67-C4-FF-A4
Dot-separated (Cisco-style):     5A6D.67C4.FFA4
Plain lowercase (no delimiters): 5a6d67c4ffa4

Splunk Filter
(5a6d67c4ffa4 OR 5A6D.67C4.FFA4 OR 5A-6D-67-C4-FF-A4 OR 5A:6D:67:C4:FF:A4)
WIFI: index=wireless_index 5A6D.67C4.FFA4
DNS: index=dns_index 5A:6D:67:C4:FF:A4
AAA: index=AAA_index (5a6d67c4ffa4 OR 5A-6D-67-C4-FF-A4)
```

---

## How It Works

### 1. Input Normalization

The script accepts any delimiter style by uppercasing the input and stripping all separators (`-`, `.`, `:`) with `tr` and `sed`, producing a clean 12-character hex string (`INPUT`). It then validates that string against a strict regex before proceeding - any non-hex characters or wrong length cause an immediate exit.

```bash
INPUT=$(echo "$1" | tr '[:lower:]' '[:upper:]' | sed -E 's/[-.:]//g')

if ! [[ "$INPUT" =~ ^[0-9A-F]{12}$ ]]; then
    echo "Invalid MAC address format"
    exit 1
fi
```

From there, `sed` with a capture-group regex rebuilds the string into the canonical colon-separated form (`MAC`) that all later operations reference:

```bash
MAC=$(echo "$INPUT" | sed -E 's/(..)(..)(..)(..)(..)(..)/\1:\2:\3:\4:\5:\6/')
```

### 2. Randomization Detection

The script isolates the first byte using `cut`, converts it to decimal with Bash's built-in arithmetic (`$((16#...))`), then converts that decimal to an 8-bit binary string via `bc` and `printf`:

```bash
FIRST_BYTE_HEX=$(echo "$MAC" | cut -d ':' -f1)
DECIMAL=$((16#$FIRST_BYTE_HEX))
BINARY=$(printf "%08d" "$(echo "obase=2; $DECIMAL" | bc)")
```

**Bit 1** (the Locally Administered bit) is the second character from the right in the 8-bit string - position 7 counting from the left (1-indexed). It is extracted with `cut`:

```bash
BIT1=$(echo "$BINARY" | cut -c7)
```

A `sed` substitution then wraps that character in brackets for the visual output, making the flag instantly visible to an engineer under pressure:

```bash
BINARY_FORMATTED="$(echo "$BINARY" | sed "s/^\(......\)\(.\)\(.\)$/\1[\2]\3/")"
# Example: 01011010  →  010110[1]0
```

```
Hex: 5A  →  Binary: 01011010
                     ──────^── position 7: Locally Administered bit
                     ───────^─ position 8: Multicast bit
```

If `BIT1` equals `1`, the address is locally administered - assigned by the OS at runtime rather than burned into the NIC at the factory - which is the definitive indicator of MAC randomization.

### 3. Format Translation

With the colon-separated `MAC` variable as the source, each output format is derived using `sed` and `tr`:

| Variable | Command | Result |
|---|---|---|
| `COLON_UPPER` | `$MAC` (already colon form) | `5A:6D:67:C4:FF:A4` |
| `DASH_UPPER` | `sed 's/:/-/g'` | `5A-6D-67-C4-FF-A4` |
| `DOT_UPPER` | Strip colons, then `sed` groups of 4 chars | `5A6D.67C4.FFA4` |
| `PLAIN_LOWER` | Strip colons, then `tr` to lowercase | `5a6d67c4ffa4` |

The Cisco dot format is produced in two piped steps - first stripping all colons to get the 12-character string, then inserting dots after every 4th character:

```bash
DOT_UPPER=$(echo "$MAC" | sed 's/://g' | sed -E 's/(.{4})(.{4})(.{4})/\1.\2.\3/')
```

### 4. Splunk Query Generation

Each index in the Splunk output reflects how that specific platform stores MAC addresses internally:

| Index | Format Used | Why |
|---|---|---|
| `index=network` (WiFi / WLC) | Cisco dot (`DOT_UPPER`) | Wireless LAN Controllers export MACs in dot-notation |
| `index=ipam` (DNS / DHCP) | Colon (`COLON_UPPER`) | IPAM and DNS platforms typically log in colon format |
| `index=aaa` (NAC) | Plain + dash (`PLAIN_LOWER OR DASH_UPPER`) | ClearPass logs MACs in plain lowercase for RADIUS, dash-separated in audit logs |

The universal filter at the top uses an `OR` chain across all four variants - useful at the start of an investigation when you aren't yet sure which index captured the relevant event.

---

## Customization

The Splunk index names are defined in the `echo` statements at the bottom of the script. To adapt `rmac` to your environment, replace them with your organization's actual index names:

```bash
# WiFi / Wireless LAN Controller logs
echo "WIFI: index=netops $DOT_UPPER"

# DNS / IPAM logs
echo "DNS: index=netipam $COLON_UPPER"

# NAC / ClearPass authentication logs
echo "Clearpass: index=etech_clearpass ($PLAIN_LOWER OR $DASH_UPPER)"
```

To add additional indexes (e.g., firewall, endpoint, or posture logs), duplicate any `echo` line and substitute the appropriate index name and MAC format variable (`$COLON_UPPER`, `$DASH_UPPER`, `$DOT_UPPER`, or `$PLAIN_LOWER`).

---

## Real-World Incident Workflow

1. User calls the helpdesk: *"I can't connect to the network."*
2. Engineer asks for the device's MAC address from the user's Wi-Fi settings.
3. Run `rmac <MAC>` on the bastion host.
4. **If randomized:** Instruct the user to disable *Private Wi-Fi Address* in their device settings and reconnect. This is the most common resolution for phantom DHCP or auth failures.
5. **If not randomized:** Copy the appropriate Splunk filter and begin log analysis immediately - no reformatting required.

---

## License

MIT - free to use, modify, and distribute.
