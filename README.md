# netwatchdog

Gets a BeagleBone's (or any systemd Linux board's) Wi-Fi back when the driver
stops being able to scan, and gives the board its mDNS name back when avahi has
renamed it.

## The problem

With some radios — BeagleBone Green Wireless/Black Wireless onboard wl18xx, and
rt2800usb dongles — iwd's scan requests are rejected after the link drops:

    iwd: Received error during CMD_TRIGGER_SCAN: Invalid argument (22)

After an ordinary deauthentication or beacon loss the board then stays
disconnected, retrying, for hours or days. Scanning works again after a fresh
start of the supplicant, which is what this script arranges.

Separately, when a board rejoins the network avahi may find its own name still
announced on its behalf (by a sleep proxy, or a cache elsewhere on the LAN) and
rename itself `myboard-2`. It never changes back, so `myboard.local` stops
resolving even though the board is reachable.

## What it does

A timer runs the script every minute, starting 4 minutes after boot. While the
link is healthy it does nothing except put back a renamed mDNS name. When the
wireless interface has no carrier, or no routable address, it escalates:

| Down for | Action |
|---|---|
| 3 min | restart the supplicant (iwd or wpa_supplicant, whichever is running) |
| 7 min | bounce the interface and restart the supplicant again |
| every 10 min after | repeat the bounce and restart |
| 60 min | reboot — only if the link has worked since this boot, so a wrong password or a missing AP can't cause a boot loop |

Radios that re-associate by themselves never reach the first step. Boards with
no wireless interface skip the service.

The mDNS name is reclaimed with `avahi-set-host-name`, so avahi-daemon and the
services that depend on it keep running. It's left alone if another host
answers for the name.

## Install

    sudo install -m 0755 netwatchdog /usr/local/sbin/netwatchdog
    sudo install -m 0644 netwatchdog.service netwatchdog.timer /etc/systemd/system/
    sudo systemctl daemon-reload
    sudo systemctl enable --now netwatchdog.timer

It needs bash, iproute2 and systemd. avahi-utils and dbus are only needed for
the name reclaim, which is skipped without them.

Its messages go to the journal:

    journalctl -u netwatchdog

## Settings

Optional, in `/etc/default/netwatchdog`:

    RESTART_AFTER=180        # seconds down before restarting the supplicant
    BOUNCE_AFTER=420         # seconds down before bouncing the interface too
    RETRY_EVERY=600          # seconds between retries after that
    REBOOT_AFTER=3600        # seconds down before rebooting; 0 never reboots
    SETTLE=120               # grace between escalation steps
    IPV4_REQUIRED=no         # yes: only an IPv4 address counts as connected
    RECLAIM_MDNS_NAME=yes    # no: leave avahi's name alone
    INTERFACE=               # watch this interface instead of the first wireless one

By default a global IPv6 address counts as connected, so an IPv6-only network
isn't mistaken for a dead link. Set `IPV4_REQUIRED=yes` if the board must have
IPv4, for instance when losing its DHCP lease should count as down.

## License

MIT — see [LICENSE](LICENSE).
