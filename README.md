# Porch Agent

An MCP server that exposes a single [WiZ](https://www.wizconnected.com/) smart bulb to an AI
agent, over streamable-HTTP. It speaks to the bulb on the local network with
[pywizlight](https://github.com/sbidy/pywizlight).

The bulb is assumed to be one you may switch off at the wall. The server treats
"unreachable" as a normal state: it always starts, polls for the bulb in the background, and
answers state queries instantly from a cache rather than blocking on a dead device.

## Requirements

- **Python 3.11 or newer.** `pywizlight` requires it. Check before deploying:
  ```bash
  python3 --version
  ```
- A WiZ bulb on the same network as this server, and its IP address.

## Install

```bash
git clone https://github.com/steveh250/Porch-Agent.git
cd Porch-Agent
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
```

## Configure

Copy the example and set your bulb's address. That is the only required setting:

```bash
cp .env.example .env
$EDITOR .env          # set WIZ_BULB_IP
```

| Variable | Default | Meaning |
| --- | --- | --- |
| `WIZ_BULB_IP` | *(required)* | Address of the bulb |
| `WIZ_BULB_PORT` | `38899` | Bulb's UDP port |
| `WIZ_POLL_INTERVAL_SECONDS` | `30` | How often to check the bulb |
| `WIZ_REQUEST_TIMEOUT_SECONDS` | `5` | Timeout for one bulb request |
| `MCP_HOST` | `127.0.0.1` | Address to bind |
| `MCP_PORT` | `8000` | Port to bind |
| `MCP_PATH` | `/mcp` | Path the endpoint is served on |
| `MCP_ALLOWED_HOSTS` | *(empty)* | Host values accepted from clients |

Variables already set in the environment take precedence over `.env`, so a deployment can
override any of them without editing the file.

A missing `WIZ_BULB_IP`, or a setting that is not a positive number, stops startup with a
message naming the variable. An unreachable *bulb* does not — that is expected.

## Run the server

```bash
porch-agent
```

It logs the configuration actually in effect, so you can confirm it at a glance:

```
INFO  porch_agent: Serving MCP over streamable-HTTP on http://127.0.0.1:8000/mcp
INFO  porch_agent: Bulb address: 192.168.1.42:38899
INFO  porch_agent: Poll interval: 30s
INFO  porch_agent: Request timeout: 5s
INFO  porch_agent.bulb: Bulb at 192.168.1.42:38899 is now reachable
```

Point your agent at `http://<host>:<port><path>`.

### Reaching it from another machine

Set `MCP_HOST=0.0.0.0` **and** `MCP_ALLOWED_HOSTS`. Both are needed: Host-header protection
is active, so with no allow-list only loopback clients are accepted and remote requests are
rejected with `421 Misdirected Request`. List the host values clients connect as:

```
MCP_HOST=0.0.0.0
MCP_ALLOWED_HOSTS=porch.local:8000,192.168.1.10:8000
```

The server warns at startup if you bind off-loopback without an allow-list.

There is no authentication. Run it only on a network you trust.

## Run the test harness

The harness is a standalone MCP client that drives a **running server** against your **real
bulb**, so you can confirm the whole path works on the machine that can actually reach it.
It changes the light visibly, then puts it back how it found it.

In one terminal:

```bash
porch-agent
```

In another:

```bash
python harness.py                 # uses .env for the server address
python harness.py --pause 3       # linger 3s on each change, easier to watch
python harness.py --url http://192.168.1.10:8000/mcp
```

It prints a pass/fail line per tool and exits non-zero if anything failed, so it is usable
from a script. It distinguishes the two failures worth telling apart:

| Exit code | Meaning |
| --- | --- |
| `0` | Every check passed |
| `1` | One or more checks failed |
| `2` | Could not reach the **server** — is it running? |
| `3` | Server is fine, but it reports the **bulb** as unreachable |

## Tools

| Tool | Notes |
| --- | --- |
| `get_light_state` | Read-only. Served from the poll cache, so it never blocks. Reports `reachable`, `ever_confirmed` and `last_confirmed`. |
| `turn_on` | Optionally takes `brightness_pct`, `red`/`green`/`blue`, or `color_temp_kelvin`. |
| `turn_off` | |
| `set_brightness` | `brightness_pct` is **0–100**, not the device's 0–255. |
| `set_color` | `red`, `green`, `blue`, each 0–255. |
| `set_color_temp` | `color_temp_kelvin`, validated against the range your bulb reports. |
| `set_scene` | By **name** (`"Cozy"`), case-insensitive — not a numeric scene id. |
| `list_scenes` | Read-only. The scene names your bulb supports. |

Setting an appearance also switches the light on; there is no supported way to change a WiZ
bulb's appearance without doing so.

Brightness round-trips are slightly lossy: the device stores a coarser scale, so a value read
back may differ by a point or two from the one written. That is expected.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `421 Misdirected Request` | `MCP_ALLOWED_HOSTS` does not list the host the client is connecting as. |
| `Configuration error: WIZ_BULB_IP is not set` | No `.env`, or the variable is missing from it. |
| `[bulb_unreachable]` / `reachable: false` | Bulb is powered off, or `WIZ_BULB_IP` is wrong. The address in the error is the one being used. |
| `[capabilities_unknown]` | The bulb has not been reachable yet, so its Kelvin range and scene list are unknown. |
| `[bulb_unsupported_operation]` | Your bulb model does not have that feature (e.g. colour on a white-only bulb). |

## Notes for maintainers

**`pywizlight`'s README disagrees with its shipped code.** Verified against 0.6.6 — follow the
code, not the docs:

- The default port is **38899**. The README says `12345`.
- `updateState()` returns `List[Optional[PilotParser]]`, so state is at index `[0]`. The
  README's `bulb.state.get_state()` raises `AttributeError`.
- `PilotBuilder` takes `colortemp=`, not `kelvin=`.
- The method is `getMac()`, not `getMAC()`. (Not used here, but easy to trip over.)
- `set_state()` emits a `setState` message that bulbs do not implement; `turn_on()` emits the
  supported `setPilot`.

**`mcp` 2.x renamed `FastMCP` to `MCPServer`** (`mcp.server.mcpserver`), and its models use
snake_case (`input_schema`, `is_error`, `read_only_hint`). Most MCP examples online are 1.x
`FastMCP` code and will not run.

There are no automated tests; validation is by hand with the harness against real hardware.
`pywizlight` does ship a working fake bulb at `pywizlight.tests.fake_bulb.startup_bulb()` if
a hardware-free suite is ever wanted.
