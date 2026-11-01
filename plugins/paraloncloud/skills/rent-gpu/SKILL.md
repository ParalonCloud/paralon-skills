---
name: rent-gpu
description: Rent a cloud GPU on ParalonCloud by the minute and use it through JupyterLab or SSH, or with an app such as ComfyUI or Open WebUI (with Ollama) installed and ready to open: pick a GPU that fits the job, confirm the cost, start it, hand over the link or run commands on it over SSH, and stop it when done. Use when the user wants to rent, start or stop a GPU or GPU server, run something that needs a GPU (training, fine-tuning, running a model, image or video generation, rendering, CUDA work, data processing), or asks what GPU they need and what it would cost.
---

# Rent a GPU on ParalonCloud

ParalonCloud rents whole GPU machines by the minute. A rental is a container with the machine's GPUs, reached one of two ways, chosen at start: **JupyterLab** in the browser (notebooks, a terminal, a file browser), or **SSH** (a root shell for commands, scp/rsync, VS Code Remote-SSH). It can also start with an **app** installed and running, with its own link: today **ComfyUI** for image and video generation. It bills per minute from the account's credit balance until it is stopped.

## Tools

With the ParalonCloud MCP server (`@paraloncloud/mcp`) connected, use its tools:

| Tool | Use |
| --- | --- |
| `list_gpus` | GPUs free right now: `node_id`, GPU model, `vram_mb`, `compute_cap`, `price_per_hour`, `spot`, `verified`, `reserved_for_you`, `ssh` (can start with SSH), `templates` (apps it can start with), country |
| `list_templates` | Apps a rental can start with (`slug`, `version`, `min_vram_gb`, `min_disk_gb`): today `comfyui` and `openwebui` |
| `get_balance` | The credit balance, in USD |
| `create_rental` | Start a rental on a `node_id`, `type` `jupyter` (default) or `ssh`; optional `template` (an app slug) and `hours` auto-stop. **Spends credits** |
| `get_rental` | Status and the Jupyter link or the SSH command, and with a template `app` (its state) and `connection.app_url` (its link); it waits for startup itself, so call it once or twice, not in a loop |
| `list_rentals` | Active rentals (`status: "all"` for history) |
| `destroy_rental` | Stop a rental and its billing; safe to call twice |
| `list_ssh_keys` / `add_ssh_key` / `delete_ssh_key` | The SSH public keys an SSH rental installs (all of them) |

Without the MCP server, the same calls exist over HTTP at `https://paraloncloud.com/api/v1` (`GET /gpus`, `GET /templates`, `GET /balance`, `POST /rentals`, `GET /rentals/{id}`, `GET /rentals`, `DELETE /rentals/{id}`, `GET/POST/DELETE /ssh-keys`) with `Authorization: Bearer <key>`. See `reference.md` in this folder.

The key must have the **GPU rentals** permission (Console → API keys). A `403 insufficient_scope` means it does not: tell the user to turn it on at https://paraloncloud.com/console.

## How to do it

1. **Ask what they want to run, not which GPU.** The job decides the GPU: what software or model, how big, how long, whether it can be interrupted. If they already named a GPU, still check it fits the job.
2. **Find GPUs that fit.** Call `list_gpus` and filter with the rules in "Choosing the GPU" below. Prefer `verified: true`. Two nodes with the same GPU differ by price, VRAM, country and `node_id`. A node with `reserved_for_you: true` is one its provider keeps for this user or their organization; such nodes come first in the list: offer one when it fits the job, and say it is reserved for them.
3. **Show the choice and the cost, and wait for a yes.** Name the GPU, VRAM, country, `price_per_hour`, and an estimate for the expected duration (price × hours). Check `get_balance` covers it: a rental needs at least one hour of balance to start, or the requested hours if `hours` is set.
4. **Offer an app when one fits.** If the job is image or video generation with ComfyUI-style workflows (Flux, SDXL, Wan), or the user names ComfyUI, call `list_templates` and offer to start with `template: "comfyui"`: it comes installed and running, with its own link, so nobody sets it up. Pick a node whose `templates` contains it. If the user wants to chat with an open model in the browser (a private ChatGPT on a GPU, Llama, Qwen, Gemma, DeepSeek, Mistral), or names Open WebUI or Ollama, offer `template: "openwebui"`: Open WebUI with Ollama, from 8 GB of VRAM. Otherwise the clean machine (no template) is the default.
5. **Pick how to connect.** JupyterLab when the user will work in the browser. SSH when you (the agent) will run commands on the machine, or the user wants their terminal or editor; it needs a node with `ssh: true` and a saved key (see "Over SSH").
6. **Start it.** `create_rental` with the `node_id`, `type: "ssh"` for SSH, and `template` for an app. By default a rental is open-ended and bills until stopped: say so, and offer an `hours` cap (it stops itself even if everyone forgets). Pass `hours` when the user wants a limit or the job has a known length.
7. **Get the connection.** Call `get_rental` once; it returns when the rental answers, usually 20 to 40 seconds after creating: `connection.access_url` (the Jupyter link) or `connection.ssh_command`. Give it to the user, or use it. If it is still pending, call it once more. With a template, the app comes up a minute or two after the machine: call `get_rental` again until `app.state` is `ready`, then give `connection.app_url` (see "With an app").
8. **Help them use it** (see "Working on the machine").
9. **Stop it when the work is done.** `destroy_rental` stops the billing. Before stopping, remind the user that **everything on the machine is deleted when the rental ends**: download results first. If you started it for a task you finished, stop it yourself after the user has their results.

Never start a rental without the user's confirmation of GPU and price. Never leave a rental running that you started for a finished task.

## Choosing the GPU

Match the job, then take the cheapest verified node that fits.

| The job | What matters |
| --- | --- |
| Run or serve a model (LLM, embeddings, speech) | VRAM for the weights plus working memory. Rough weights size: parameters (billions) × 2 GB at FP16/BF16, × 1 at FP8/INT8, × 0.5 at 4-bit. Add 20 to 30% for context and runtime. FP8 needs `compute_cap` 8.9 or higher |
| Train or fine-tune | Much more VRAM than inference: full fine-tuning needs about 4× the FP16 weights; LoRA/QLoRA fits on far less. BF16 needs `compute_cap` 8.0 or higher. Several GPUs in one node help |
| Image or video generation (Stable Diffusion, Flux, video models) | A VRAM threshold set by the model: SDXL runs on 12 GB, Flux needs about 24 GB at full precision, video models often 24 GB or more. Faster GPUs finish sooner. Offer the ComfyUI template |
| 3D rendering (Blender Cycles and similar) | An NVIDIA GPU with enough memory for the scene; newer GPUs render faster |
| CUDA development, testing, benchmarks | The exact GPU or architecture they target (`compute_cap`), not the biggest one |
| Data processing, notebooks, research | Enough VRAM for the data; usually the cheapest fitting GPU |
| Long jobs that can resume | A **spot** node (`spot: true`) is cheaper but its owner can take it back with two minutes' notice: only for work that saves checkpoints and can restart |

`compute_cap` is the CUDA architecture: 7.x Volta/Turing (FP16), 8.0 to 8.6 Ampere (adds BF16), 8.9 Ada (adds FP8), 9.0 Hopper, 10.x and 12.x Blackwell. A cheap GPU with a low `compute_cap` may not run a modern stack at all.

When nothing free fits, say so and offer the closest option (a smaller precision, a smaller model, two GPUs in one node, or waiting), rather than renting something that will not work.

## Over SSH

- Keys: an SSH rental installs every public key saved on the account (`list_ssh_keys`). A key added after the rental started is not on it.
- To run commands yourself, make a key for the session and save its public half: `ssh-keygen -t ed25519 -N '' -f ~/.ssh/paralon_ed25519`, then `add_ssh_key` with the line of `~/.ssh/paralon_ed25519.pub`. Never send a private key.
- Connecting needs `cloudflared` on the computer that holds the private key (macOS `brew install cloudflared`, Windows `winget install --id Cloudflare.cloudflared`, Linux: the release binary from github.com/cloudflare/cloudflared). Then run `connection.ssh_command`, adding `-i ~/.ssh/paralon_ed25519` when the key is not a default one, and `-o StrictHostKeyChecking=accept-new` for a non-interactive first connection.
- One-off commands: `ssh … root@<host> 'nvidia-smi'`. Files: `scp` and `rsync -e 'ssh -o ProxyCommand="cloudflared access ssh --hostname %h"'`.
- `409 ssh_not_available`: the node cannot take SSH; pick one with `ssh: true`. `400 no_ssh_key`: save a key first.

## With an app

- `list_templates` gives each app's `slug`, the `version` a rental gets (tested and pinned by ParalonCloud) and its minimums; `min_vram_gb` is per GPU, since the app runs on one card. `list_gpus` already lists in `templates` the apps each node can run: pick from those.
- The rental is the same (JupyterLab or SSH, same price, billed from the moment the machine answers); the app installs on top. `get_rental` shows `app.state`: `installing`, `starting`, then `ready` with `connection.app_url`. It usually takes a minute or two after the machine answers for ComfyUI, a few minutes for Open WebUI (Ollama is a 1.5 GB download).
- `app_url` carries the app's token: whoever has the link can open it. Give it to the user only.
- The app starts empty: models are downloaded onto the machine. In ComfyUI, tell the user to press **Download to this GPU** in ComfyUI's top bar after opening a workflow: it fetches every model the workflow names into the right folder on the rented GPU, with progress. ComfyUI's own **Download** buttons in *Missing Models* save the file on the user's computer instead, which does not help: say so if they mention them. Other ways: ComfyUI-Manager's Model Manager, or `wget` / `huggingface-cli` on the machine into `/workspace/ComfyUI/models/<folder>`. Everything is deleted when the rental ends.
- In Open WebUI there is no login (the link is the key) and no model yet: tell the user to pull one from the model selector, or run `ollama pull <model>` on the machine. Ollama answers on `localhost:11434` there. Thinking models (qwen3, deepseek-r1) write long reasoning before answering and feel slow; for quick chat suggest a non-thinking one such as `gemma3:4b`, `llama3.2:3b` or `qwen2.5:7b`. Chats and models are deleted with the rental.
- `app.state` `failed`: the app did not start, the machine still runs and bills. Say so; the log is `/var/log/paralon-app.log` on the machine. Offer to stop it.
- On an SSH rental the app is also reachable without its link: `ssh -L 8188:localhost:8188` (ComfyUI) or `-L 8080:localhost:8080` (Open WebUI; `-L 11434:localhost:11434` for Ollama's API) with the rental's `ssh_command`, then `http://localhost:<port>`.
- Errors: `400 unknown_template` (see `list_templates`), `409 template_not_available` / `template_needs_more_vram` / `template_needs_newer_gpu` / `template_needs_more_disk` (pick a node whose `templates` contains it).

## Working on the machine

- JupyterLab or the SSH shell: Python, `nvidia-smi`, root. Install with `pip`, `conda` or `apt`. The working folder is `/workspace`.
- **No Docker inside the rental** (it is an unprivileged container), and only the chosen access is reachable from outside: JupyterLab or SSH, no other open ports.
- Disk is the machine's free disk; large models and datasets must fit.
- Processes keep running if the browser closes. The instance can restart, which stops processes but keeps files: checkpoint long jobs.
- Upload data through the Jupyter file browser or `scp`/`rsync` over SSH, or download it on the machine (`wget`, `huggingface-cli`, `git clone`). Copy results off before stopping.
- On a spot rental, `get_rental` shows `reclaim_at` when the owner takes the machine back: save work immediately, the rental stops at that time.

## Costs and safety

- Billing is per minute while the rental runs, at the price locked when it started.
- `create_rental` sends an idempotency key, so a retried call never starts a second GPU.
- The key's "at most N rentals at once" limit caps how many it can hold.
- Errors: `402 insufficient_credits` (top up at https://paraloncloud.com/add-credits), `409 node_rented` / `node_offline` / `node_not_ready` (pick another node from a fresh `list_gpus`), `409 rental_limit_reached` (the key's cap; stop one or raise the cap in the Console).

Docs: https://paraloncloud.com/docs/rental-api and https://paraloncloud.com/docs/rent-a-gpu
