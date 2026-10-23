# What the Comet X exposes

Read out of GL.iNet's published firmware source,
[gl-inet/glkvm](https://github.com/gl-inet/glkvm) at commit `3e8dd23`
(tagged 1.10.0, 2026-08-11), on 2026-10-23. Paths below are relative to that
repository. Nothing here has been checked against a real device yet: the
[checklist at the end](#verify-on-hardware) is what to confirm first.

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
  (`valid_auth_token`, same file).
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

**The device does OCR itself.** `ocr=1` on the same endpoint returns JSON
instead of a JPEG. It takes `ocr_langs` and a crop: `ocr_left`, `ocr_top`,
`ocr_right` and `ocr_bottom`, with -1 meaning no limit.
`GET /api/streamer/ocr` reports the engine and its available languages. One
code comment mentions an RKNN (Rockchip NPU) mode, where the language list is
empty and not validated. The shape of the OCR result is not in this
repository's handler; capture one from the device. If the result carries word
positions, there is no need for local Tesseract.

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

## Not used, but present

These are in kvmd but out of scope unless a config turns them on:

- virtual media (`/api/msd/*`);
- ATX on a single-host device (`/api/atx/*`; per-port ATX is the switch's);
- Wake-on-LAN (`/api/wol`);
- the Fingerbot (`/api/fingerbot`);
- Redfish (`/api/redfish`);
- the VPN integrations (Tailscale, ZeroTier, NetBird, Cloudflare).

## Verify on hardware

Before writing code against the real device, in this order:

1. **Firmware version.** Check it in the web UI. If it is not 1.10.x, re-read
   this file against the matching source.
2. **`/api/` is reachable.** `curl -k https://<device>/api/auth/check` should
   answer 401 or 403, not 404. kvm-pilot reports a 404 on all of `/api/*` on
   some GLKVM firmware until an nginx block is uncommented, and says a firmware
   upgrade can revert the edit.
3. **Log in** with a bounded `expire`, and record the `auth_token` cookie.
4. **`GET /api/switch`.** Save the whole response into this repo as a test
   fixture. It answers where port names live, what the signal state looks like,
   and whether the model block is filled.
5. **Switch by API.** `set_active?port=1`, timing how long until the `switch`
   event arrives and until a fresh snapshot shows the new machine.
6. **The WebSocket** with the cookie only, no token in the URL.
7. **One OCR snapshot.** Save the JSON as a fixture and judge its word
   positions against local Tesseract on the same frame.
8. **The stuck-key bug.** Type a long string at full speed with and without
   the workaround.
9. **Two-step login,** if it is turned on.
10. **The certificate.** Its fingerprint, and whether it survives a reboot and
    a firmware update. That decides how pinning works.
