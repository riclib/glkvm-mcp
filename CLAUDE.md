# Working in glkvm-mcp

The README is the spec. `docs/device-api.md` is what the device exposes and
`docs/prior-art.md` is why this exists. Read all three before starting; do
not re-derive them. What to build next is in the GitHub issues (`gh issue
list`); a decision that changes the spec goes into the README in the same
change, not only into a commit message.

## Branches

One branch per issue, named `<issue>-<slug>` (e.g. `1-server`). Open a PR
that says `Closes #N`. `main` stays buildable.

## The device

There is one Comet X, on Ricardo's desk, and it controls real machines.

- **Without the device**, test against the fake: a local server that answers
  the HTTP endpoints and the WebSocket from the fixtures in `tests/fixtures/`.
  Every tool needs a test there.
- **With the device**, never switch inputs, type or click without being asked
  to in this session. The machine behind the active input may be in use. Read
  calls (`GET /api/switch`, a snapshot) are fine.
- **Fixtures captured from the device** are scrubbed before they are
  committed: tokens, cookies, serials, MAC and IP addresses, hostnames.
- When something is verified on hardware, tick it off in
  `docs/device-api.md#verify-on-hardware` with the firmware version.
- Never assume a call works because the published source says so: the
  shipping firmware can differ. Say which one you checked.

## Secrets

The device password never appears in code, tests, fixtures, logs, tool
arguments or commit messages. The server reads it from `pass`, an environment
variable, or a file readable only by its owner. The call log redacts typed
text to its length.

## Rust

- Stable Rust, edition 2024. `cargo fmt`, `cargo clippy -- -D warnings`
  and `cargo test` pass before a PR.
- Keep the dependencies few and the lockfile committed.

## The Omarchy widget

The widget lives in `plugin/` with plugin id `riclib.glkvm`.

- Colours come from `Commons.Color`, never bare `Color`; bare `Color` breaks
  on Qt 6.12.
- Any text that comes from the server is set with `textFormat: Text.PlainText`.
- It reads the server's state socket and the call log. It never talks to the
  device itself.
