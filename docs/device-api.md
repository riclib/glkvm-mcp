# What the Comet X exposes

Read out of GL.iNet's published firmware source,
[gl-inet/glkvm](https://github.com/gl-inet/glkvm) at commit `3e8dd23`
(tagged 1.10.0, 2026-08-11), on 2026-10-23. Paths below are relative to that
repository. **Checked against the real device on 2026-10-26**, first on the
shipped firmware **1.8.1 release2** and again after its upgrade to
**1.10.1** the same day (both report kvmd 4.82, ustreamer 6.13, kernel
6.1.141, Rockchip RV1126B; GL.iNet's firmware number is not kvmd's). The
source for neither is published (tags stop at v1.5.0release1), so the notes
stay read against 1.10.0 and each claim is marked below with what the device
actually did. Responses captured that day, scrubbed, are in
`tests/fixtures/device-1.8.1/` and `tests/fixtures/device-1.10.1/`. The
[checklist at the end](#verify-on-hardware) says what is still open.

GLKVM is a fork of PiKVM's kvmd, so most of PiKVM's API applies. Everything
below is under `https://<device>/api/`: nginx strips the prefix and proxies to
kvmd (`configs/nginx/kvmd.ctx-server.conf`, `location /api`).

## Authentication

`POST /api/auth/login`, form fields `user`, `passwd`, `expire`
(`kvmd/apps/kvmd/api/auth.py:180`).

- `expire` is a whole number, at least 0 (`kvmd/validators/auth.py:64`).
  0 asks for a token that does not expire, which is what the Python server
  sends. We send a bounded value.
- On success: JSON `{"token": …, "failed_since_last_success": …}` and an
  `auth_token` cookie. A token is 64 hex characters
  (`valid_auth_token`, same file). **On 1.8.1** the reply body is
  `{"token": …}` only, no `failed_since_last_success`; the cookie is
  `auth_token`, path `/`, with no `Secure` flag and no `Expires` even with
  `expire=3600`, so the server-side lifetime is the only bound. Login took
  65 ms; two-step was off out of the box. `POST /api/auth/logout` answered
  200 and `GET /api/auth/check` 403 afterwards. **On 1.10.1** the reply is
  `{"failed_since_last_success": 0, "token": …}`, as the source says; the
  cookie is unchanged. The server must accept both shapes.
- Login is rate limited: `429` with `remaining_time` when tripped.
- Logging in records the User-Agent (device type and browser) in the device
  log. Send an honest one, e.g. `glkvm-mcp/<version>`.
- `POST /api/auth/logout` ends the session.

**Two-step login** is an approval on the device, not a TOTP code. With it
enabled, login returns
`{"two_step_required": true, "two_step_token": …, "expires_in": …}`. The
person at the desk approves on the touchscreen: the `two_step_pending` and
`two_step_approve` endpoints are only callable by `/usr/sbin/gl_kvm_gui`, the
touchscreen UI. The client then polls `POST /api/auth/two_step_complete` with
`two_step_token` until it answers with the token rather than
`{"status": "pending"}` (`auth.py:261`). A separate TOTP feature exists under
`/api/2fa/*` (`kvmd/apps/kvmd/api/twofa.py`).

## Keyboard and mouse: the WebSocket

`wss://<device>/api/ws?stream=false` (nginx rewrites it to kvmd's `/ws`). The
Python server passes the token as an `auth_token` query parameter; the cookie
should do, and keeps the token out of URLs and logs. To verify.

Events sent, as used by the Python server (the shapes match PiKVM's):

```json
{"event_type": "key",          "event": {"key": "KeyA", "state": true, "finish": false}}
{"event_type": "mouse_button", "event": {"button": "left", "state": true}}
{"event_type": "mouse_move",   "event": {"to": {"x": 0, "y": 0}}}
{"event_type": "mouse_wheel",  "event": {"delta": {"x": 0, "y": -1}, "squash": false}}
```

- Keys are web `KeyboardEvent.code` names (`KeyA`, `ShiftLeft`, `Enter`); the
  firmware's table is `keymap.csv` at the repository root.
- Mouse coordinates are absolute, from -32768 to 32767 across the screen.
- Wheel deltas are clamped to -127 to 127.
- Send a ping regularly to keep the socket open.

kvmd also **pushes state** on the same socket, one event type per subsystem
(`kvmd/apps/kvmd/server.py:162-173`): `hid`, `streamer`, `ocr`, `atx`, `msd`,
`switch`, and others. The `switch` event is how to see an input change land
without polling.

### The stuck-key bug

Firmware up to 1.9.0 drops or repeats keys when keydown and keyup arrive too
close together: gl-inet/glkvm issue #52, and forum thread 64940. The working
pattern, from the Python server:

- each key goes down, waits at least 25 ms, then comes up with `finish: true`;
- modifiers go down before the main key and come up after it, never
  interleaved (gl-inet/glkvm issue #22);
- anything still held after 250 ms is released by a watchdog.

Whether 1.10 still needs it is on the checklist; keep it either way, it costs
little.

## Screenshots and OCR

`GET /api/streamer/snapshot` (`kvmd/apps/kvmd/api/streamer.py:54`) returns a
JPEG of the active input.

- `preview=1` with `preview_max_width`, `preview_max_height` and
  `preview_quality` (default 80) asks for a scaled-down copy.
- `allow_offline=1` returns a frame even with no signal.
- Without a live frame it returns an `UnavailableError`.
- **On 1.8.1**, with no machine showing video, `preview=1&allow_offline=1`
  returned an 800-px "NO LIVE VIDEO" JPEG once and, a minute later, a plain
  text `500 Internal Server Error` while the streamer was restarting (the
  WebSocket pushed five `streamer` events in a row). Treat a 500 here as
  transient and retry; `/api/streamer` itself answered fine both times.
  The streamer reports `h264`, `h265`, `venc_mode` and `zero_delay`, no
  `resolution`, and `desired_fps` up to 70.

**OCR on the device is off as shipped.** The 1.10.0 source has `ocr=1` on the
same endpoint returning JSON instead of a JPEG, with `ocr_langs` and a crop
(`ocr_left`, `ocr_top`, `ocr_right`, `ocr_bottom`, -1 for no limit), and
`GET /api/streamer/ocr` reporting the engine and its languages. **On 1.8.1**
the WebSocket's `ocr` event says `{"enabled": false, "langs": {"default": [],
"available": []}}` and `snapshot?ocr=1` answers a plain text
`500 Internal Server Error` (`tests/fixtures/device-1.8.1/snapshot_ocr_response.txt`).
So local Tesseract is the path, not the fallback, until a firmware enables it.
Keep the device call behind a check of the `ocr` event so it can take over if
a later firmware turns it on. **On 1.10.1** it is still off, but the event now
names the engine: `{"enabled": false, "engine": "tesseract", "langs": …}`,
and `snapshot?ocr=1` answers a JSON `503 UnavailableError` instead of the
plain 500. Right after a reboot every snapshot answered that 503 while
`/api/streamer` reported `"streamer": null`, so "no streamer yet" and "OCR
off" look the same from this endpoint; read the `ocr` event to tell them
apart.

**New on 1.10.1:** the WebSocket also pushes `serial`
(`{"exist": false, "devices": []}`) and `recorder`
(`{"recording": false, "elapsed_seconds": 0, "file": ""}`) events, and sends
its whole state dump twice on connect. Both subsystems are out of scope.

## Switching between the four machines

On an RM4PE kvmd turns on the switch subsystem:
`kvmd/apps/kvmd/__init__.py:95` checks `hw_model == "rm4pe"`, where the model
is read from `/proc/gl-hw-info/model`. It uses PiKVM's switch API, but backed
by GL.iNet's own driver, `kvmd/apps/kvmd/switch/sysfs_chain.py`, not PiKVM's
serial-chained Switch. The handlers are in `kvmd/apps/kvmd/api/switch.py`.

| Call | Does |
| --- | --- |
| `GET /api/switch` | Full state, see below |
| `POST /api/switch/set_active?port=N` | Make input N active, 0–3 |
| `POST /api/switch/set_active_prev`, `set_active_next` | Step, stopping at the ends |
| `POST /api/switch/set_port_params?port=N&name=…` | Name an input. Also takes `edid_id`, `dummy`, and the ATX click delays |
| `POST /api/switch/atx/power?port=N&action=…` | `on`, `off`, `off_hard`, `reset_hard` |
| `POST /api/switch/atx/click?port=N&button=…` | `power`, `power_long`, `reset` |
| `POST /api/switch/edids/create`, `change`, `remove` | EDID management |

Things to know:

- **Ports are 0–3.** The handler accepts PiKVM's `unit.channel` notation
  (`1.2`) too and translates it, as `(unit - 1) * 4 + (channel - 1)`
  (`sysfs_chain.py:50`).
- **An out-of-range port is silently ignored**, and the call still returns
  success (`sysfs_chain.py:70`). The server must validate the port itself.
- **The state** (`kvmd/apps/kvmd/switch/state.py`) has
  `summary.active_port`, and `summary.active_id` written as `1.<port+1>`.
  On this hardware `synced` is always true. There is per-input video signal
  and USB link state from `get_video_links` and `get_usb_otg_links`
  (`sysfs_chain.py`), which `kvm_hosts` can use for "has signal".
- `model.ports` and `model.units` start empty in the state code. Where the
  port names set with `set_port_params` show up has to be read from a real
  response.
- **The real response on 1.8.1** (`tests/fixtures/device-1.8.1/switch.json`,
  5.7 KB, 33 ms): top-level `atx`, `beacons`, `colors`, `edids`, `model`,
  `summary`, `usb_host`, `usb_otg`, `video`. `model.ports` has four entries,
  `id` `1.1`–`1.4`, `channel` 0–3, `unit` 0, `name` `""` (so names live
  there), `video.dummy` true and per-port ATX click delays;
  `model.firmware.version` is 8. `summary` is
  `{"active_port": 0, "active_id": "1.1", "synced": true}`. `video.links` and
  `usb_otg.links` are four booleans, one per port (that day
  `[true, false, true, true]`: three machines cabled, port 2 empty), and
  `usb_host` is false. `edids.all` carries a default "PiKVM V3" EDID. The
  WebSocket pushes the same shape as a `switch` event on connect, and
  afterwards a short form with only `summary`, `video`, `usb_otg` and
  `usb_host`. `/api/switch/state` is not a route (404).
- A `video.links` true with "NO LIVE VIDEO" on the active port happened
  together, so a link means a cable is detected, not that the machine is
  sending a picture.
- **Switching, measured on 1.8.1** (sequence 0 → 2 → 3 → 0):
  `POST set_active` blocks for about 2.5 s and answers `{"ok": true,
  "result": {}}`; the `switch` event showing the new `active_port` arrived
  2.6 s after the POST; `GET /api/switch` agreed at once; the first 200
  snapshot of the new machine came 2.3–2.6 s after the POST. Port 3's
  machine gave only 503s for 15 s (no picture), and the event for the
  return to port 0 never arrived within 10 s although the GET showed it, so
  confirm a switch with a GET rather than waiting on the event alone.
  `set_active?port=7` answered ok and changed nothing, as the source said.

## Not used, but present

These are in kvmd but out of scope unless a config turns them on:

- virtual media (`/api/msd/*`);
- ATX on a single-host device (`/api/atx/*`; per-port ATX is the switch's);
- Wake-on-LAN (`/api/wol`);
- the Fingerbot (`/api/fingerbot`);
- Redfish (`/api/redfish`);
- the VPN integrations (Tailscale, ZeroTier, NetBird, Cloudflare).

## Verify on hardware

Before writing code against the real device, in this order. Ticked items were
done on 2026-10-26 against firmware 1.8.1 release2, read-only, with the
session logged out afterwards.

1. [x] **Firmware version.** 1.8.1 release2, which the device calls latest.
   Its source is not published, so this file stays read against 1.10.0 and
   every claim is verified on the device instead.
2. [x] **`/api/` is reachable.** `/api/auth/check` answers 401 unauthenticated,
   and an unknown `/api/` path answers 404, so a 401 on a route means the
   route exists. No nginx edit was needed.
3. [x] **Log in** with a bounded `expire`: see Authentication for what 1.8.1
   returns.
4. [x] **`GET /api/switch`.** Fixture saved; see Switching for the shape.
5. [x] **Switch by API.** Done on 1.8.1 with permission; timings under
   Switching. About 2.5 s end to end.
6. [x] **The WebSocket** with the cookie only, no token in the URL: 101, and
   it pushed `loop`, `gpio`, `ocr`, `info` (four parts), `hid_keymaps`,
   `switch`, `streamer`, `hid`, `msd`, `fingerbot` and `repeater` on connect.
7. [x] **One OCR snapshot.** Not available: OCR is disabled on 1.8.1 (see
   Screenshots and OCR). Local Tesseract is the path.
8. [ ] **The stuck-key bug.** Type a long string at full speed with and without
   the workaround. Approved, but not run: every cabled machine was at its
   lock screen, and a password field is the one place not to type into.
   Needs an unlocked machine with a text field focused. 1.10.1 is past the
   1.9.0 cut-off in the issue, so this now also tells whether the workaround
   is still needed.
9. [ ] **Two-step login.** Off out of the box; test when it is turned on.
10. [x] **The certificate.** The default is self-signed, `CN=localhost`,
    `O=GLKVM`, valid 1970-01-01 to 1979-12-30 (generated with the clock at
    epoch, so every verifier calls it expired), SHA-256
    `39A42EB639A429832E8AE3F4A810DD38EA06752E48255279616A00557CB7A135`, the
    same on the LAN and over Tailscale, and **it survived the upgrade from
    1.8.1 to 1.10.1 unchanged**, so a pin on it holds across updates. But
    1.10.1 adds a **custom TLS certificate** upload (Settings, "TLS
    Certificate", type Custom: a PEM certificate with an optional CA chain,
    and the private key). Ricardo's device now serves a certificate from his
    mkcert root with SANs for the tailnet name, the short name and both IPs,
    which every machine that trusts that root verifies normally. So the
    server verifies against a CA file first, and pins a fingerprint only for
    a device still on its default certificate.
