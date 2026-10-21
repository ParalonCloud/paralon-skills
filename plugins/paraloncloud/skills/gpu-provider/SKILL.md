---
name: gpu-provider
description: Manage the GPU machines a provider rents out on ParalonCloud: check each node's state and health (online, rented, GPU load and temperature), add a node and get its install command, take a node off the market and back, set prices against the market, take back a spot node, and report earnings. Use when the user provides GPUs on ParalonCloud and asks about their nodes, machines, rigs, prices, availability, temperatures, or earnings.
---

# Run your GPU nodes on ParalonCloud

A **node** is a machine the user rents out through the ParalonCloud agent. Renters pay by the minute; the provider earns the price minus the platform fee, credited in US dollars. This skill is for the provider side: their own nodes, never other people's.

## Tools

With the ParalonCloud MCP server (`@paraloncloud/mcp-rentals`) connected:

| Tool | Use |
| --- | --- |
| `list_my_nodes` | Every node: `state`, GPUs, price (default, min, max, custom, what a renter pays), verification, reputation, the current rental, and `metrics` (per-GPU load %, temperature °C, power W, VRAM; host CPU and memory) |
| `get_my_node` | One node, same fields |
| `add_node` | Add a node; returns `docker_command` to run on the machine |
| `get_node_install_command` | The install command of an existing node |
| `set_node_available` | Off the market (`false`) or back (`true`) |
| `set_node_price` | Price in USD per hour for the whole node, or `null` for the default |
| `update_node` | Name, spot, availability and price in one call |
| `reclaim_spot_node` | Take a spot node back; a spot rental on it **stops in two minutes** |
| `get_earnings` | Earnings by UTC day (rentals per node, inference per account) and the withdrawable balance |
| `list_gpus` | What renters can rent right now, with prices: use it to compare the market |

Without the MCP server, the same calls exist over HTTP at `https://paraloncloud.com/api/v1` (`GET/POST /nodes`, `GET/PATCH /nodes/{id}`, `GET /nodes/{id}/command`, `POST /nodes/{id}/reclaim`, `GET /earnings`). See `reference.md` in this folder.

The key must have the **My nodes** permission (Console → API keys). A `403 insufficient_scope` means it does not: tell the user to turn it on at https://paraloncloud.com/console. Payouts are not available through the API: requesting one or changing where it goes is done on the site.

## Node states

| `state` | Meaning | What to do |
| --- | --- | --- |
| `available` | Online, verified, ready, on the market | Nothing |
| `rented` | A rental is running (`rental` has its start and price; the renter is never shown) | Nothing; do not change it under the renter without the user's say |
| `paused` | The owner took it off the market | Put back with `set_node_available(true)` when they want |
| `not_verified` | The GPU check has not passed | Show `verification.error`; it re-runs after a reboot or a driver change |
| `not_ready` | Online but cannot take a rental yet | Show `rent_ready_reason` (often the rental image still downloading, or a network check) |
| `offline` | The agent is not connected | The machine, its network or the agent container is down: check it is powered, online, and the agent container is running |

## Common requests

**"How are my nodes?"** Call `list_my_nodes` and give a short table: name, state, GPU, price, and for online nodes the GPU load and temperature from `metrics`. Point out what needs attention: offline nodes, failed verification, a GPU at 85 °C or more, a node that is `available` but has not been rented (check its price).

**"Is anything running hot?"** From `metrics.gpus[]`: 80 °C and above is warm, 85 °C and above deserves a look (airflow, fan curve, power limit). `metrics` is `null` when the node sent nothing in the last few minutes.

**"Add a machine."** Confirm a name and the type (`gpu`, `cpu` or `mac`) and OS (`linux` or `windows`), then `add_node`. Give the user `docker_command` to run on that machine, and say it holds the node's secret token: keep it private. The node shows `offline` until the command runs, then goes through the GPU check. Machine requirements: https://paraloncloud.com/docs/hardware-requirements

**"Take it off the market" / "I need the machine" / "it got rented elsewhere."** `set_node_available(false)`. A rental already running continues to its end; nothing new starts. Put it back later with `set_node_available(true)`. For a machine listed on several marketplaces, this is how it never gets rented twice.

**"Set my price" / "am I priced right?"** Call `list_gpus` and look at what renters pay for the same GPU model, and `get_my_node` for the node's `price.default`, `price.min` and `price.max` (a custom price must sit between them: 50% to 300% of the default). Suggest a price with the reason, wait for a yes, then `set_node_price`. `null` returns to the default. A running rental keeps the price it started with. A spot node always rents at the spot price and takes no custom price. Never quote other providers' private data; `list_gpus` shows only public listings.

**"I need my spot machine now."** Only for a spot node. Warn that the renter's rental stops two minutes later, get a clear yes, then `reclaim_spot_node`. Put it back with `set_node_available(true)`.

**"How much did I earn?"** `get_earnings` (default 30 days; `days` up to 366). Report the total, the best days and the split by node, and the `earnings_balance` that can be withdrawn on the site. Rentals still running are added on the day they end; `running_rentals_billed_before_fee` is what they billed so far, before the fee.

## Rules

- Confirm before any change: price, availability, spot, name, adding a node, and always before a reclaim.
- Never change a rented node in a way that affects the renter without a clear instruction.
- Deleting a node and payouts are on the site only: https://paraloncloud.com/dashboard/nodes and https://paraloncloud.com/earnings

Docs: https://paraloncloud.com/docs/provider-api
