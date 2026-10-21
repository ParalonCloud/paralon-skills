# ParalonCloud skills for Claude Code

A Claude Code plugin for [ParalonCloud](https://paraloncloud.com): three skills that teach the agent how to use ParalonCloud well, and the [ParalonCloud MCP server](https://github.com/ParalonCloud/mcp-rentals) that gives it the tools.

| Skill | What the agent learns |
| --- | --- |
| `rent-gpu` | Rent a GPU by the minute for any job (training, running models, image or video generation, rendering, CUDA work, data processing): pick a GPU that fits, confirm the cost, start it, hand over the JupyterLab link, stop it when done |
| `gpu-provider` | Run the GPU machines you rent out: node states and health (load, temperature), add a node, take it off the market and back, price against the market, earnings |
| `llm-inference` | Call open models through the OpenAI-compatible API: chat, embeddings, classification, Responses, Anthropic Messages, batch, images, and Claude Code on an open model |

## Install

```bash
claude plugin marketplace add ParalonCloud/paralon-skills
claude plugin install paraloncloud@paraloncloud
```

Or inside Claude Code: `/plugin marketplace add ParalonCloud/paralon-skills`, then `/plugin install paraloncloud@paraloncloud`.

The MCP server reads your key from the `PARALON_API_KEY` environment variable. Create a key at [paraloncloud.com/console](https://paraloncloud.com/console) (API keys) and give it the permissions you need:

- **Inference** to call models (on by default),
- **GPU rentals** to rent GPUs,
- **My nodes** to manage the nodes you provide.

```bash
export PARALON_API_KEY=prlc_your_key_here
claude
```

Use a key only for the agent, with only the permissions it needs.

## Try it

> "I want to fine-tune an 8B model with LoRA for about three hours. What GPU do I need and what will it cost? Start it if it's under $5."

> "How are my nodes doing? Is any GPU running hot? Am I priced right for my 4090?"

> "Switch this script from OpenAI to an open model on ParalonCloud."

## Safety

The skills tell the agent to confirm before anything that costs money or changes a node: starting a rental, a price, availability, adding a node, reclaiming a spot node. Rentals bill per minute until stopped; the agent offers an auto-stop and stops rentals it started when the work is done. Payouts are never reachable from the API.

## Other clients

The MCP server works in any MCP client (Claude Desktop, Cursor, ...): see [mcp-rentals](https://github.com/ParalonCloud/mcp-rentals). The skills are plain `SKILL.md` folders under `plugins/paraloncloud/skills/` and can be used wherever Agent Skills are supported.

## Links

- Rental API: https://paraloncloud.com/docs/rental-api
- Provider API: https://paraloncloud.com/docs/provider-api
- Inference API: https://paraloncloud.com/docs/inference-api

MIT
