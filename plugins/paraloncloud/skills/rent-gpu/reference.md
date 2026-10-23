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
curl https://paraloncloud.com/api/v1/rentals/<rental_id> -H "Authorization: Bearer $PARALON_API_KEY"

# Active rentals (?status=all for history)
curl https://paraloncloud.com/api/v1/rentals -H "Authorization: Bearer $PARALON_API_KEY"

# Stop it and its billing (safe to repeat)
curl -X DELETE https://paraloncloud.com/api/v1/rentals/<rental_id> -H "Authorization: Bearer $PARALON_API_KEY"
```

`POST /rentals` body: `node_id` (required), `type` (`"jupyter"`, the only type), `hours` (1 to 720, optional auto-stop), `name` (optional label).

`GET /gpus` item: `node_id`, `gpus[]` (`model`, `vram_mb`, `compute_cap`), `price_per_hour`, `list_price_per_hour`, `spot`, `verified`, `reserved_for_you` (a node its provider keeps for you or your organization; listed first, nobody else sees it), `country`, `country_code`.

`GET /rentals/{id}`: `status` (`pending`, `running`, `stopped`, ...), `connection_pending`, `connection.access_url`, `price_per_hour`, `expires_at`, `spot`, `reclaim_at`.

Errors: `401 invalid_api_key`, `402 insufficient_credits`, `403 insufficient_scope`, `404 rental_not_found` / `node_not_found`, `409 node_rented` / `node_offline` / `node_not_ready` / `rental_limit_reached`, `429 rate_limit_exceeded`.

Full docs: https://paraloncloud.com/docs/rental-api
