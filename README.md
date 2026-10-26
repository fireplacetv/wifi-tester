# wifi-tester

A single bash script that diagnoses slow or flaky public Wi-Fi on macOS. It
separates Wi-Fi radio problems from venue-internet problems, and flags DNS,
captive-portal, TLS-inspection, IPv6, port-filtering and bufferbloat issues.
It uses only built-in macOS tools and runs in about 10 seconds.

## Usage

```sh
./wifi-tester                 # light checks, then asks before the speed test
./wifi-tester --no-speed      # light checks only, never loads the link
./wifi-tester --speed         # light checks + speed test, no prompt
./wifi-tester -q              # verdict only
```

The light checks send a few hundred bytes. The speed test (`networkQuality -s`,
about 15 seconds) saturates the link, so it is opt-in.

## What it checks

| Check | How | Flags when |
|---|---|---|
| Wi-Fi signal / SNR | `system_profiler SPAirPortDataType` | RSSI ≤ -70 dBm (marginal) or ≤ -75 (weak); SNR < 20 dB |
| Router | 20 pings to the default gateway | loss > 2%, jitter > 20 ms, or avg > 30 ms |
| Internet | 20 pings each to 1.1.1.1 and 8.8.8.8 | loss > 2%, avg > 120 ms, or jitter > 50 ms |
| Large packets | 1472-byte unfragmented ping | fails (path MTU below 1500, often a VPN) |
| DNS | `dig` via the network resolver vs. 1.1.1.1 | slow, failing, or private answers for a public name |
| Captive portal | Apple and Google HTTP probes | both fail (and not a Google 204) |
| HTTPS | timed request to cloudflare.com | non-200, or first byte > 1.5 s |
| TLS inspection | issuer of the HTTPS certificate | issuer belongs to a known filtering proxy |
| IPv6 | global address + IPv6 request | address exists but IPv6 traffic fails |
| Ports | TCP to 80, 443, 53, 22, 587, 993 | any port filtered |
| Speed (optional) | `networkQuality -s` | downlink < 5 Mbps, uplink < 1 Mbps, or "Low" responsiveness |

## Reading the verdict

The verdict lists the most serious findings first. The thresholds are rules of
thumb, not calibrated values, so treat a single borderline warning as a lead.

- **Router no reply but internet OK**: many public routers drop ping. The script
  then can't tell radio problems from venue-link problems, and says so.
- **Router OK, internet bad**: the problem is beyond the access point. The venue's
  link is congested or throttled.
- **Router slow but radio fine**: ping alone can't tell the air from a busy
  router. The script says so and does not blame the Wi-Fi link.
- **Router bad with weak radio**: the Wi-Fi link is congested or noisy between
  you and the AP. Try another location or band.
- **Large packets fail**: if a tunnel interface (VPN or Private Relay) has a
  smaller MTU, the script names it. With no tunnel, the smaller MTU is probably
  on the link itself, and nothing on your Mac will fix it.
- **No throughput test**: the default run doesn't measure speed or bufferbloat,
  and the verdict says so. Use `--speed` to check them.
- **Comparing with your phone**: when the verdict finds problems, it suggests
  running the test on your phone's hotspot. If the hotspot is clearly faster,
  the problem is this network, not your device.
- **Speed Low responsiveness (bufferbloat)**: latency spikes once the link is
  busy. Congestion or poor queueing on the venue side causes lag during large
  transfers.
- **Speed low but latency clean**: the connection is capped or thinly shared. It
  isn't interference.

## Requirements

macOS 12 or later. Uses `system_profiler`, `networkQuality`, `dig`, `nc`, `curl`,
`ping`, `route`, `scutil`, and `ifconfig`, all of which ship with macOS.
