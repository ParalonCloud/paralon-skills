# Rental API over HTTP

For when the ParalonCloud MCP server is not connected. Base URL `https://paraloncloud.com/api/v1`, header `Authorization: Bearer $PARALON_API_KEY` (a `prlc_` key with the GPU rentals permission).

```bash
# GPUs free now
curl https://paraloncloud.com/api/v1/gpus -H "Authorization: Bearer $PARALON_API_KEY"

# Balance
curl https://paraloncloud.com/api/v1/balance -H "Authorization: Bearer $PARALON_API_KEY"

# Start a rental (async). Idempotency-Key makes a retry return the same rental.
curl -X POST https://paraloncloud.com/api/v1/rentals \
  -H "Authorization: Bearer $PARALON_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{"node_id": "<node_id from /gpus>", "type": "jupyter", "hours": 2}'

# Poll every few seconds until connection_pending is false; then connection.access_url is the Jupyter link
# (or connection.ssh_command for type "ssh")
curl https://paraloncloud.com/api/v1/rentals/<rental_id> -H "Authorization: Bearer $PARALON_API_KEY"

# Active rentals (?status=all for history)
curl https://paraloncloud.com/api/v1/rentals -H "Authorization: Bearer $PARALON_API_KEY"

# SSH: save a public key first (every saved key is installed on an SSH rental), then rent with "type": "ssh"
curl -X POST https://paraloncloud.com/api/v1/ssh-keys \
  -H "Authorization: Bearer $PARALON_API_KEY" -H "Content-Type: application/json" \
  -d "{\"public_key\": \"$(cat ~/.ssh/id_ed25519.pub)\", \"name\": \"laptop\"}"
curl https://paraloncloud.com/api/v1/ssh-keys -H "Authorization: Bearer $PARALON_API_KEY"

# Apps a rental can start with (ComfyUI), then rent a node whose /gpus "templates" has it
curl -s https://paraloncloud.com/api/v1/templates -H "Authorization: Bearer $PARALON_API_KEY"
curl -s -X POST https://paraloncloud.com/api/v1/rentals \
  -H "Authorization: Bearer $PARALON_API_KEY" -H "Content-Type: application/json" \
  -d '{"node_id": "<node_id>", "template": "comfyui"}'
# Poll until app.state is "ready"; connection.app_url is its link (with its token)

# Stop it and its billing (safe to repeat)
curl -X DELETE https://paraloncloud.com/api/v1/rentals/<rental_id> -H "Authorization: Bearer $PARALON_API_KEY"
```

`POST /rentals` body: `node_id` (required), `type` (`"jupyter"`, the default, or `"ssh"`), `hours` (1 to 720, optional auto-stop), `name` (optional label).

`GET /gpus` item: `node_id`, `gpus[]` (`model`, `vram_mb`, `compute_cap`), `price_per_hour`, `list_price_per_hour`, `spot`, `verified`, `reserved_for_you` (a node its provider keeps for you or your organization; listed first, nobody else sees it), `ssh` (can start with `type: "ssh"`), `country`, `country_code`.

`GET /rentals/{id}`: `status` (`pending`, `running`, `stopped`, ...), `connection_pending`, `connection.access_url`, for SSH `connection.ssh_command` / `ssh_host` / `ssh_user`, `price_per_hour`, `expires_at`, `spot`, `reclaim_at`.

Errors: `401 invalid_api_key`, `402 insufficient_credits`, `403 insufficient_scope`, `404 rental_not_found` / `node_not_found`, `409 node_rented` / `node_offline` / `node_not_ready` / `rental_limit_reached` / `ssh_not_available`, `400 no_ssh_key` / `invalid_key`, `429 rate_limit_exceeded`.

Full docs: https://paraloncloud.com/docs/rental-api
