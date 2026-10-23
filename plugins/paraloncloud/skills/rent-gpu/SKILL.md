---
name: rent-gpu
description: Rent a cloud GPU on ParalonCloud by the minute and use it through JupyterLab: pick a GPU that fits the job, confirm the cost, start it, hand over the link, and stop it when done. Use when the user wants to rent, start or stop a GPU or GPU server, run something that needs a GPU (training, fine-tuning, running a model, image or video generation, rendering, CUDA work, data processing), or asks what GPU they need and what it would cost.
---

# Rent a GPU on ParalonCloud

ParalonCloud rents whole GPU machines by the minute. A rental is a container with the machine's GPUs, reached through **JupyterLab** in the browser (notebooks, a terminal, a file browser). It bills per minute from the account's credit balance until it is stopped.

## Tools

With the ParalonCloud MCP server (`@paraloncloud/mcp`) connected, use its tools:

| Tool | Use |
| --- | --- |
| `list_gpus` | GPUs free right now: `node_id`, GPU model, `vram_mb`, `compute_cap`, `price_per_hour`, `spot`, `verified`, `reserved_for_you`, country |
| `get_balance` | The credit balance, in USD |
| `create_rental` | Start a rental on a `node_id`; optional `hours` auto-stop. **Spends credits** |
| `get_rental` | Status and the Jupyter link; it waits for startup itself, so call it once or twice, not in a loop |
| `list_rentals` | Active rentals (`status: "all"` for history) |
| `destroy_rental` | Stop a rental and its billing; safe to call twice |

Without the MCP server, the same calls exist over HTTP at `https://paraloncloud.com/api/v1` (`GET /gpus`, `GET /balance`, `POST /rentals`, `GET /rentals/{id}`, `GET /rentals`, `DELETE /rentals/{id}`) with `Authorization: Bearer <key>`. See `reference.md` in this folder.

The key must have the **GPU rentals** permission (Console → API keys). A `403 insufficient_scope` means it does not: tell the user to turn it on at https://paraloncloud.com/console.

## How to do it

1. **Ask what they want to run, not which GPU.** The job decides the GPU: what software or model, how big, how long, whether it can be interrupted. If they already named a GPU, still check it fits the job.
2. **Find GPUs that fit.** Call `list_gpus` and filter with the rules in "Choosing the GPU" below. Prefer `verified: true`. Two nodes with the same GPU differ by price, VRAM, country and `node_id`. A node with `reserved_for_you: true` is one its provider keeps for this user or their organization; such nodes come first in the list: offer one when it fits the job, and say it is reserved for them.
3. **Show the choice and the cost, and wait for a yes.** Name the GPU, VRAM, country, `price_per_hour`, and an estimate for the expected duration (price × hours). Check `get_balance` covers it: a rental needs at least one hour of balance to start, or the requested hours if `hours` is set.
4. **Start it.** `create_rental` with the `node_id`. By default a rental is open-ended and bills until stopped: say so, and offer an `hours` cap (it stops itself even if everyone forgets). Pass `hours` when the user wants a limit or the job has a known length.
5. **Get the link.** Call `get_rental` once; it returns when the Jupyter link (`connection.access_url`) is ready, usually 20 to 40 seconds after creating. Give the user the link. If it is still pending, call it once more.
6. **Help them use it** (see "Working on the machine").
7. **Stop it when the work is done.** `destroy_rental` stops the billing. Before stopping, remind the user that **everything on the machine is deleted when the rental ends**: download results first. If you started it for a task you finished, stop it yourself after the user has their results.

Never start a rental without the user's confirmation of GPU and price. Never leave a rental running that you started for a finished task.

## Choosing the GPU

Match the job, then take the cheapest verified node that fits.

| The job | What matters |
| --- | --- |
| Run or serve a model (LLM, embeddings, speech) | VRAM for the weights plus working memory. Rough weights size: parameters (billions) × 2 GB at FP16/BF16, × 1 at FP8/INT8, × 0.5 at 4-bit. Add 20 to 30% for context and runtime. FP8 needs `compute_cap` 8.9 or higher |
| Train or fine-tune | Much more VRAM than inference: full fine-tuning needs about 4× the FP16 weights; LoRA/QLoRA fits on far less. BF16 needs `compute_cap` 8.0 or higher. Several GPUs in one node help |
| Image or video generation (Stable Diffusion, Flux, video models) | A VRAM threshold set by the model: SDXL runs on 12 GB, Flux needs about 24 GB at full precision, video models often 24 GB or more. Faster GPUs finish sooner |
| 3D rendering (Blender Cycles and similar) | An NVIDIA GPU with enough memory for the scene; newer GPUs render faster |
| CUDA development, testing, benchmarks | The exact GPU or architecture they target (`compute_cap`), not the biggest one |
| Data processing, notebooks, research | Enough VRAM for the data; usually the cheapest fitting GPU |
| Long jobs that can resume | A **spot** node (`spot: true`) is cheaper but its owner can take it back with two minutes' notice: only for work that saves checkpoints and can restart |

`compute_cap` is the CUDA architecture: 7.x Volta/Turing (FP16), 8.0 to 8.6 Ampere (adds BF16), 8.9 Ada (adds FP8), 9.0 Hopper, 10.x and 12.x Blackwell. A cheap GPU with a low `compute_cap` may not run a modern stack at all.

When nothing free fits, say so and offer the closest option (a smaller precision, a smaller model, two GPUs in one node, or waiting), rather than renting something that will not work.

## Working on the machine

- Open the link: JupyterLab with Python, a terminal and `nvidia-smi`. Install with `pip`, `conda` or `apt`.
- **No Docker inside the rental** (it is an unprivileged container), and only Jupyter is reachable from outside: no SSH, no other open ports.
- Disk is the machine's free disk; large models and datasets must fit.
- Processes keep running if the browser closes. The instance can restart, which stops processes but keeps files: checkpoint long jobs.
- Upload data through the Jupyter file browser, or download it on the machine (`wget`, `huggingface-cli`, `git clone`). Download results through the file browser before stopping.
- On a spot rental, `get_rental` shows `reclaim_at` when the owner takes the machine back: save work immediately, the rental stops at that time.

## Costs and safety

- Billing is per minute while the rental runs, at the price locked when it started.
- `create_rental` sends an idempotency key, so a retried call never starts a second GPU.
- The key's "at most N rentals at once" limit caps how many it can hold.
- Errors: `402 insufficient_credits` (top up at https://paraloncloud.com/add-credits), `409 node_rented` / `node_offline` / `node_not_ready` (pick another node from a fresh `list_gpus`), `409 rental_limit_reached` (the key's cap; stop one or raise the cap in the Console).

Docs: https://paraloncloud.com/docs/rental-api and https://paraloncloud.com/docs/rent-a-gpu
