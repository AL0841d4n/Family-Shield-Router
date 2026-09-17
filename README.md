# Family Shield Router

A parental-control router solution built on **OpenWrt**, with a self-hosted **Flask** backend and a lightweight **HTML/CSS/JS** web dashboard. It turns any OpenWrt-compatible router into a network-level parental control system, so no app installs are needed on kids' devices since everything is enforced at the router.

> Graduation project, submitted as a group assignment. I designed and built the full system end-to-end (firmware setup, backend, and frontend).

## What it does

Parents can open a dashboard from any browser on the home network and:

- **See every connected device**, auto-discovered from DHCP leases and the ARP table, with online/offline status
- **Pause internet access** instantly for any device (blocks its traffic at the firewall)
- **Set daily screen-time schedules**, so a device automatically loses and regains internet access at chosen times
- **Filter content by category** (e.g. social media, gaming) or by custom domain, per device
- **Add custom blocklist categories** beyond the built-in ones
- **View an activity log** of pauses, unblocks, and schedule changes for accountability
- **See an at-a-glance dashboard summary** (total devices, online, paused, protected)

## Tech stack

| Layer | Technology |
|---|---|
| Router firmware/OS | OpenWrt |
| Firewall / device blocking | `nftables` (MAC-based drop rules) |
| DNS & DHCP | `dnsmasq` (device discovery + category-based domain blocking) |
| Scheduling | `cron` (daily block/unblock windows per device) |
| Backend | Python + Flask (REST API) |
| Frontend | HTML, CSS, vanilla JavaScript (single-page dashboard) |
| Storage | JSON files on the router filesystem (no external DB, kept lightweight for embedded hardware) |

## How it works

1. **Device discovery**: `devices.py` reads `/tmp/dhcp.leases` and `/proc/net/arp` to build a live list of devices on the network, merges it with a persisted JSON "database" of known devices (names, profiles, settings), and keeps `last_seen` timestamps.
2. **Blocking**: `firewall.py` manages a dedicated `nftables` table/chain and adds or removes `drop` rules matched on a device's MAC address to instantly pause/unpause its internet access.
3. **Scheduling**: `scheduler.py` writes tagged entries into the router's crontab that call the same `nftables` blocking logic at the configured start/end times, so schedules keep running independently of the dashboard being open.
4. **Content filtering**: `dns_filter.py` maintains category to domain lists (built-in + custom), works out which categories are blocked by at least one device, and regenerates a `dnsmasq` config that returns `0.0.0.0` / `::` for blocked domains, then reloads `dnsmasq`.
5. **Activity log**: `logger.py` appends every pause/unblock/schedule/filter change to a capped JSON log (last 100 events) so parents can review recent activity.
6. **API + dashboard**: `app.py` exposes all of the above as a REST API and serves the static dashboard, which calls the API to render devices, schedules, filters, and the activity feed.

## Project structure

```
family-shield/
├── backend/
│   ├── app.py            # Flask app & REST API routes
│   ├── devices.py        # Device discovery, DB, pause/hide state
│   ├── firewall.py       # nftables rule management
│   ├── scheduler.py      # Cron-based screen-time schedules
│   ├── dns_filter.py     # Category/domain-based DNS filtering
│   └── logger.py         # Activity log
└── frontend/
    └── index.html        # Dashboard UI (HTML/CSS/JS)
```

## Setup

This runs directly on an OpenWrt router (or a Linux box acting as the gateway, for testing).

**1. Flash OpenWrt**
Install a recent OpenWrt build on a supported router. See the [OpenWrt Table of Hardware](https://openwrt.org/toh/start) to confirm your device is supported, then follow OpenWrt's install guide for your model.

**2. Install dependencies on the router**
```bash
opkg update
opkg install python3 python3-pip nftables dnsmasq
pip3 install flask
```

**3. Copy the project onto the router**
```bash
scp -r family-shield root@<router-ip>:/root/
```

**4. Run the backend**
```bash
cd /root/family-shield/backend
python3 app.py
```
The Flask app serves both the API and the dashboard on port `5000`.

**5. Open the dashboard**
From any device on the same network, browse to:
```
http://<router-ip>:5000
```

**6. (Optional) Run on boot**
Add an `/etc/init.d` service or a `rc.local` entry that starts `app.py` on router boot so the dashboard survives a reboot.

> ⚠️ This project directly manipulates `nftables` rules and `dnsmasq` config on the router, so test on a spare/lab router before deploying on your main home router.

## Notes / limitations

- DNS-based filtering blocks a domain network-wide once any device has it blocked (dnsmasq doesn't do per-device DNS answers), so per-device DNS filtering would need per-device DNS redirection as a future improvement.
- Device data is stored in flat JSON files rather than a database, which keeps it dependency-free on constrained router hardware but doesn't scale past a typical home network.
- No authentication on the dashboard yet; it's intended for trusted local-network access only.
