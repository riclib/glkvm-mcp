# Prior art

## kennypeh85/glkvm-mcp

[github.com/kennypeh85/glkvm-mcp](https://github.com/kennypeh85/glkvm-mcp),
Python, MIT. Reviewed at `b5102d0` on 2026-10-23. Four commits from one author
(2026-05 to 2026-06), 15 stars. CI only checks that the tools register.

**What it is.** One file, `glkvm_mcp.py`, about 900 lines, run by
`uv run --script` with inline dependencies. It provides tools for:

- connect and disconnect;
- typing, key chords and holding a key;
- mouse move (absolute and by percentage), click and scroll;
- screenshots, returned as image content or written to a file;
- local Tesseract OCR, with a "click this text" tool.

**What it gets right, and what we keep:**

- The stuck-key workaround: keydown, at least 25 ms, keyup with
  `finish=true`; modifiers outside the main key; a 250 ms watchdog. Ported as
  is.
- OCR to find click targets, rather than trusting a vision model's guess at
  coordinates.
- No power, virtual media or anything else beyond keyboard, mouse and screen.
- Nothing phones home. The only network peer is the device.

**Why we replace it rather than fork it:**

1. **One machine.** It holds a single global connection and has no tool for
   the Comet X's four inputs (`/api/switch`).
2. **The password is a tool argument** (`kvm_connect(host, password)`), so it
   lands in the agent's transcript and in every archive of it.
3. **The token never expires** (`expire: 0`), travels in the WebSocket URL, and
   TLS verification is off with no pinning.
4. **The dependencies are unpinned** (`mcp>=1.2`, `websockets>=12`, …),
   resolved fresh on each launch.
5. **Smaller things:**
   - `kvm_screenshot_to_file` writes to any path it is given while declaring
     itself read-only.
   - Two-step login is refused outright.
   - The password stays in memory for the whole session.

## kvm-pilot

[pypi.org/project/kvm-pilot](https://pypi.org/project/kvm-pilot/), a Python
client for PiKVM and GLKVM, listing the GL-RM1 and GL-RM1PE among its targets.
Useful mainly for one claim: on some GLKVM firmware every `/api/*` call
returns 404 until a block in `/etc/kvmd/nginx-kvmd.conf` is uncommented, and a
firmware upgrade can revert that edit. No switch support.

## PiKVM

GLKVM is a fork of [pikvm/kvmd](https://github.com/pikvm/kvmd). PiKVM's API
documentation describes the shared endpoints, including `/api/switch`, which
PiKVM uses for its own serial-chained Switch. On the Comet X the same
endpoints drive GL.iNet's built-in four-port switch instead.
