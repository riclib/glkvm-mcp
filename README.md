# glkvm-mcp

An MCP server that lets a coding agent drive the computers behind a
[GL.iNet Comet X](https://www.gl-inet.com/en-us/products/gl-rm4pe) (GL-RM4PE)
four-port IP-KVM. The agent can see a screen, type, click, read text off the
screen, and switch between the four machines **by name**. It is written in
Rust and ships as one binary.

It will also ship an Omarchy bar widget that shows what the agent is doing
with the KVM: which machine it is acting on, and a log of its calls.

**Status: specified, not built.** The device arrives the week of 2026-10-26.
The build is [#1](https://github.com/riclib/glkvm-mcp/issues/1) and the bar
widget is [#2](https://github.com/riclib/glkvm-mcp/issues/2). This README is
the spec. [`docs/device-api.md`](docs/device-api.md) is what the device
exposes, read out of GL.iNet's published source, and
[`docs/prior-art.md`](docs/prior-art.md) is a review of the existing Python
server this replaces.

## Why another one

[kennypeh85/glkvm-mcp](https://github.com/kennypeh85/glkvm-mcp) (Python, MIT)
already drives a GL.iNet KVM, and its stuck-key workaround is worth keeping.
Three things make a rewrite worth it rather than a fork:

- **It cannot switch machines.** It holds one connection and has no idea the
  Comet X has four inputs. Switching is the whole reason to buy this device.
  The API for it exists (`/api/switch`, below), but no client we found uses it.
- **The KVM password is a tool argument.** `kvm_connect(host, password)` puts
  the admin password into the agent's conversation, its session files, and
  every archive of those (AgentsView, for one).
- **Its token never expires**, rides in the WebSocket URL, and its Python
  dependencies are unpinned and resolved fresh on each launch.

The script is about 900 lines. Rewritten in Rust it is a single binary with a
lockfile.

## The device

The Comet X is **one KVM with four inputs, not four KVMs.** Each host gets an
HDMI input and a USB-C port. The device controls one host at a time.
Switching is global: the front button, the touchscreen, and the web UI all
change the same active input, and the local HDMI output follows it.

Facts from the reviews and from GL.iNet's docs:

- **Switching is slow.** A review measured 7–8 seconds per switch, as the HDMI
  source drops and the next one is detected.
- **Switching has side effects.** Virtual media detaches. The mouse jiggler
  only runs on the selected input. Each input can have its own ATX power board
  or Fingerbot.
- **Video:** up to 4K30.
- **Power:** PoE (802.3af/at), with a 5 V USB-C backup.
- **Firmware:** GLKVM, GL.iNet's fork of PiKVM's kvmd. The published source is
  1.10.0 (2026-08-11).

## What the device exposes

The short version is below; [`docs/device-api.md`](docs/device-api.md) has the
detail, with file and line references into
[gl-inet/glkvm](https://github.com/gl-inet/glkvm) at `3e8dd23`.

| Need | Call |
| --- | --- |
| Log in | `POST /api/auth/login` (form `user`, `passwd`, `expire`) → `auth_token` cookie |
| Two-step login | approval on the device's own touchscreen, polled with `POST /api/auth/two_step_complete` |
| Keyboard and mouse | WebSocket `/api/ws`: `key`, `mouse_button`, `mouse_move`, `mouse_wheel` events |
| Screenshot | `GET /api/streamer/snapshot` (`preview`, `preview_max_width`, `preview_quality`) |
| Read text | local Tesseract on a snapshot; the device's own `snapshot?ocr=1` is disabled on shipped firmware 1.8.1 |
| Which machine is active | `GET /api/switch` → `summary.active_port` (0–3); also pushed as `switch` events on the WebSocket |
| Switch machine | `POST /api/switch/set_active?port=N` (0–3), or `set_active_prev` / `set_active_next` |
| Name the inputs | `POST /api/switch/set_port_params` |
| Power a machine | `POST /api/switch/atx/power` and `/atx/click`, per port (needs GL.iNet's ATX board) |

On a Comet X kvmd only turns the switch API on when it detects the RM4PE
(`hw_model == "rm4pe"` in `kvmd/apps/kvmd/__init__.py`), and it is backed by
GL.iNet's own driver (`kvmd/apps/kvmd/switch/sysfs_chain.py`), not by PiKVM's
serial switch. **None of this is confirmed on hardware yet.** Shipping
firmware can differ from the published source, and the
[kvm-pilot](https://pypi.org/project/kvm-pilot/) client says `/api/*` returns
404 on some GLKVM firmware until an nginx block is uncommented. The published
nginx config has `location /api` enabled. The first job when the device
arrives is the hardware checklist in
[`docs/device-api.md`](docs/device-api.md#verify-on-hardware).

## Design

**One binary, MCP over stdio.** Built with the official Rust MCP SDK
(`rmcp`), `tokio`, `reqwest` for HTTP and `tokio-tungstenite` for the
WebSocket.

**Credentials never pass through the agent.** The device address, user and
password come from a config file and a secret source: a `pass` entry, an
environment variable, or a file readable only by its owner. Tools take no
credentials. The server logs in on its first tool call and holds the session.

**Machines by name.** The config maps the four inputs to names (`nas`,
`workstation`, …), and the server can push those names to the device so its
touchscreen agrees. `kvm_switch("nas")` sets the input, waits for
`active_port` to change and for a fresh video frame, and only then returns,
so the 7–8 second switch is invisible to the agent.

**Session hygiene.**
- The server asks for a bounded token lifetime (`expire`) rather than one that
  never expires, and logs out on exit.
- The token travels as a cookie, never in a URL.
- Two-step login works: the server waits while you approve it on the touchscreen.
- TLS is pinned to the device's own certificate, recorded on first contact, not
  simply switched off.

**Keyboard that does not stick.** The workaround from the Python server,
ported: every key goes down, waits at least 25 ms, then comes up with
`finish=true`; modifiers wrap outside the main key; a watchdog releases
anything held longer than 250 ms; disconnecting releases everything.

**OCR locally, with the device's as an upgrade.** The 1.10.0 source can OCR a
snapshot on the device, but on the shipped 1.8.1 firmware OCR is disabled and
the call fails (verified 2026-10-26, see `docs/device-api.md`). So local
Tesseract (a subprocess, not a linked library) does the reading. The server
watches the WebSocket's `ocr` event and switches to the device's OCR if a
firmware ever reports it enabled.

**Destructive extras off by default.** ATX power and virtual media are not
offered as tools unless the config turns them on. Keyboard and mouse already
give the agent full control of whichever machine is active; there is no
reason to add power buttons to that by default.

### Tools

| Tool | Does |
| --- | --- |
| `kvm_status` | Device reachable, logged in, which machine is active, held keys |
| `kvm_hosts` | The four inputs: name, port, which is active, whether it has signal |
| `kvm_switch(host)` | Make a machine active, by name or port; returns once its picture is live |
| `kvm_screenshot(max_width?, quality?)` | JPEG of the active machine, as MCP image content |
| `kvm_read_screen(text?, region?)` | OCR: every word with its position, or the matches for `text` |
| `kvm_click_text(text, button?, count?, region?)` | Find text on screen and click its centre |
| `kvm_type(text, wpm?)` | Type a string |
| `kvm_keys(combo)` | A chord: `Ctrl+Alt+Delete`, `Super+L`, `F11` |
| `kvm_hold_key(key, ms)` | Hold a key |
| `kvm_release_all` | Release every key and button |
| `kvm_mouse_move(x_pct, y_pct)` | Move by percentage of the screen |
| `kvm_click(button?, count?)` | Click where the pointer is |
| `kvm_scroll(dx?, dy?)` | Scroll |
| `kvm_power(host, action)` | Opt-in. ATX power on, off, reset for one machine |

Every tool that touches a machine names that machine in its result, so the
agent's transcript always says what it acted on.

### The call log

The server writes one line of JSON per tool call to
`$XDG_STATE_HOME/glkvm-mcp/calls.jsonl`: the time, the tool, the target
machine, the arguments with text redacted to its length, how long it took, and
the outcome. It also publishes live state (connected, active machine, the call
in flight) on a Unix socket in `$XDG_RUNTIME_DIR`. That is what the Omarchy
widget reads, and it is useful on its own.

## Omarchy bar widget

A bar widget, plugin id `riclib.glkvm`, ships in this repo. It is quiet until
an agent connects. It shows:

- which machine is active, and whether an agent is acting on it right now;
- a pulse while a call is in flight, labelled with the tool;
- a popup with the recent call log;
- buttons to release all keys and to disconnect the agent.

It must stay compatible with the Omarchy shell's `Commons.Color` (not the bare
`Color` that breaks on Qt 6.12). The details are
[#2](https://github.com/riclib/glkvm-mcp/issues/2).

## License

MIT. The stuck-key workaround follows
[kennypeh85/glkvm-mcp](https://github.com/kennypeh85/glkvm-mcp) (MIT). The
device API is read out of [gl-inet/glkvm](https://github.com/gl-inet/glkvm)
(GPL-3.0); none of its code is copied here.
